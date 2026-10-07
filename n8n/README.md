# n8n 自动更新 edgetunnel 优选IP

用 n8n 每天自动跑一次测速，把「对你本地网络最快」的 Cloudflare IP 写回 edgetunnel 的 KV，实现订阅里始终是最优 IP。

- 工作流文件：`edgetunnel-cfst-daily.json`（n8n 里「导入」即可）
- 依赖工具：[CloudflareSpeedTest (CFST)](https://github.com/XIU2/CloudflareSpeedTest)
- 适用：自建 n8n（Docker / 裸机），**n8n Cloud 用不了 Execute Command 节点**

---

## 一、原理

edgetunnel 的 `/sub` 每次请求都会现场取优选 IP（`_worker.js:356-361`）：

```js
本地IP库.随机IP ? 生成随机IP(...)          // 从 CF-CIDR 段里随机抽，不测速
               : await env.KV.get('ADD.txt')  // ← 用你写进 KV 的 IP
```

所以思路是：

```
n8n 定时 → 本地跑 CFST 测速 → 结果 POST 到 /admin/ADD.txt → 存进 KV → 订阅里就是你实测的 IP
```

**关键前提：必须把 `本地IP库.随机IP` 设为 `false`**，否则 Worker 仍走随机 IP，根本不读 `ADD.txt`。

---

## 二、后台鉴权（为什么能这么调）

看 `_worker.js:105`，后台 Cookie 校验是：

```js
authCookie !== await MD5MD5(UA + 加密秘钥 + 管理员密码)
```

即 **Cookie 与 User-Agent 强绑定**。因此工作流里两次请求必须用**完全相同的 User-Agent**（示例里统一用 `edgetunnel-n8n/1.0`）。

工作流采用「先登录再写入」的方式，避免自己实现 `MD5MD5`：

| 步骤 | 请求 | 说明 |
| --- | --- | --- |
| 1 | `POST /login`，body `password=你的ADMIN` | 成功时返回 `Set-Cookie: auth=...`（`_worker.js:95-103`） |
| 2 | 从响应头取 `auth` 值 | 有效期 24h，但每次运行都会重新登录，无所谓 |
| 3 | `POST /admin/ADD.txt`，带 `Cookie: auth=...` | 内容整体写入 KV 的 `ADD.txt`（`_worker.js:277-286`） |

---

## 三、部署步骤

### 1. 准备 CFST

在 **n8n 宿主机**上（不是 Cloudflare，也不是你的客户端）安装 CFST：

```bash
mkdir -p /opt/cfst && cd /opt/cfst
wget -N https://github.com/XIU2/CloudflareSpeedTest/releases/latest/download/cfst_linux_amd64.tar.gz
tar -xzf cfst_linux_amd64.tar.gz
chmod +x cfst
./cfst -v          # 验证能跑
```

> **如果 n8n 跑在 Docker 里**：`Execute Command` 是在容器内执行的，要么把 cfst 二进制挂载进容器，要么改用 **SSH 节点**连到路由器/PC 上跑，再返回结果。

### 2. 准备 IP 源文件

CFST 默认读同目录的 `ip.txt`。直接用官方仓库自带的全量段即可（文件里已包含），也可以换成运营商段：

```bash
cd /opt/cfst && ./cfst -f ip.txt -o result.csv -p 20 -tl 300 -dn 5 -tp 443
```

常用参数（完整参数以 `./cfst -h` 为准）：

| 参数 | 含义 |
| --- | --- |
| `-f` | 输入 IP 文件，默认 `ip.txt` |
| `-o` | 输出结果文件，默认 `result.csv` |
| `-tp` | 测速端口，默认 443 |
| `-tl` | 平均延迟上限(ms)，超过则丢弃 |
| `-t` / `-n` | 延迟测速次数 / 线程数 |
| `-dn` | 下载测速数量 |
| `-dd` | 禁用下载测速（只测延迟，更快） |
| `-p` | 控制台显示结果数量 |
| `-url` | 下载测速地址 |

### 3. 导入 n8n 工作流

n8n → 工作流 → 右上角 `...` → **Import from File** → 选 `edgetunnel-cfst-daily.json`。

然后修改这 3 处占位符：

| 位置 | 改成 |
| --- | --- |
| `运行 CFST 测速` 的 command | 你自己的 CFST 路径 |
| `登录获取 Cookie` 的 url | `https://你的域名/login` |
| `登录获取 Cookie` 的 body `password` | 你的 `ADMIN` 密码 |
| `写入 ADD.txt` 的 url | `https://你的域名/admin/ADD.txt` |

> 建议把域名和密码放进 n8n 的 **Variables**（`$vars.XXX`）或 **Credentials**，不要硬编码在工作流里。

### 4. 关掉「随机IP」

登录后台 → 「优选订阅生成」→ 把 **本地IP库 / 随机IP** 关掉，保存。

（或直接 `POST /admin/config.json`，把 `优选订阅生成.本地IP库.随机IP` 改成 `false`。）

### 5. 手动跑一次验证

点 `Execute Workflow`：

- `解析 result.csv` 应输出类似：
  ```
  104.16.x.x:443#CF-1-12.34MB-45.2ms
  172.64.x.x:443#CF-2-11.02MB-48.7ms
  ```
- `写入 ADD.txt` 应返回 `{"success":true,"message":"自定义IP已保存"}`
- 浏览器访问 `https://你的域名/admin/ADD.txt` 应能看到刚写入的 IP（`_worker.js:290-293`）

### 6. 客户端刷新

把 `SUBUpdateTime`（订阅更新时间，小时）调成 `24`，或在客户端里设自动更新间隔。默认是 3 小时。

---

## 四、常见坑

| 现象 | 原因 / 解决 |
| --- | --- |
| 登录返回 302 跳 `/login` | 密码错，或 `ADMIN` 环境变量没设（`_worker.js:80` 会直接返回 404 noADMIN） |
| 写入返回 302 跳 `/login` | 两次请求的 **User-Agent 不一致**，Cookie 校验失败 |
| 写入成功但订阅里还是随机 IP | 忘了关 `本地IP库.随机IP` |
| 订阅里 IP 完全没变 | 客户端没到刷新时间；或 `ADD.txt` 被 Worker 当成空（检查是否含逗号被拆散） |
| n8n 请求被 Cloudflare 拦截 | 关掉 Bot Fight Mode / WAF 对该路径的挑战，或给 n8n 出口 IP 加白名单 |
| Execute Command 报找不到 cfst | n8n 在容器内，二进制没挂载进去 |

### 关于 ADD.txt 的格式

Worker 用 `整理成数组()`（`_worker.js:5911`）按 **逗号 / 制表符 / 引号 / 换行** 拆分成数组，所以：

- ✅ 每行一个 `IP:端口#备注`，用换行分隔
- ⚠️ **备注里不要出现逗号**，否则会被拆成两条无效记录
- 也支持 `sub://其他worker域名`（优选订阅生成器）和 `https://...`（优选API）两种写法

---

## 五、不用 n8n 的等价做法

如果只想用系统 cron，逻辑完全一样：

```bash
#!/bin/bash
set -e
DOMAIN="https://你的域名"
ADMIN="你的ADMIN密码"
UA="edgetunnel-n8n/1.0"

# 1) 测速并生成 IP 列表
cd /opt/cfst
./cfst -f ip.txt -o result.csv -p 20 -tl 300 -dn 5 -tp 443 >/dev/null 2>&1
BODY=$(tail -n +2 result.csv | awk -F, 'NR<=10 && $1 ~ /^[0-9.]+$/ {print $1":443#CF-"NR}' | paste -sd'\n')

# 2) 登录拿 Cookie（注意 -A 指定 UA，与写入时一致）
COOKIE=$(curl -s -i -A "$UA" -X POST "$DOMAIN/login" \
  --data-urlencode "password=$ADMIN" | grep -i '^set-cookie:' | sed -E 's/.*auth=([^;]+).*/\1/')

# 3) 写入 KV
curl -s -A "$UA" -X POST "$DOMAIN/admin/ADD.txt" \
  -H "Cookie: auth=$COOKIE" -H 'Content-Type: text/plain;charset=utf-8' \
  --data-binary "$BODY"
```

crontab 里加一行 `0 4 * * * /opt/cfst/update.sh` 即可。

---

## 六、安全提醒

`ADD.txt` 接口只要拿到 `auth` Cookie 就能写。因此：

- n8n 的域名、密码放 Variables / Credentials，别提交到 Git
- 别把 `ADMIN` 设得太弱
- 本项目 README 已声明仅供学习测试，请遵守当地法规
