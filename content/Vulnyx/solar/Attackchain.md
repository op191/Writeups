# Vulnyx Solar 攻击链速查

> **一句话主线**：从 443 证书 SAN 找到两个域名，用泄露的数据库备份进入后台并拿到 MQTT 凭据；借 `data` 主题的 DOM XSS 盲打管理员，再用 `record` 主题 + wkhtmltopdf 读取源码；随后从 `server.php` 的命令队列拿到 webshell，最后沿 `www-data → lenam → julian → root` 的 `doas` 链提权。
>
> 完整命令、源码分析与截图见 [main.md](main.md)。

## 总览

| 阶段 | 步骤 | 核心动作 | 关键结果 |
| --- | --- | --- | --- |
| [外部打点](#阶段一外部打点) | 1–2 | 证书 SAN、目录爆破、破解备份 | 两个域名、登录页、`database.sql.gz`、后台凭证 |
| [MQTT XSS](#阶段二mqtt-与-dom-xss) | 3–5 | 审计后台源码、连 Broker、盲打管理员 | `data` 主题 DOM XSS、admin MQTT 凭据 |
| [管理员侦察](#阶段三records-与-pdf) | 6–9 | 读 `/records`、看 PDF 元数据、注入 `record` | wkhtmltopdf CVE、后端源码 |
| [第二入口](#阶段四第二入口与-webshell) | 10–13 | 审计 `server.php`、投递命令文件、执行 | `www-data` webshell |
| [横向移动](#阶段五横向) | 14–15 | `mosquitto_pub` 外带、GPG 破解、Node 调试 | `lenam`、`julian` |
| [提权](#阶段六root) | 16 | `backups` + `libbackup.so` + `PATH` 劫持 | `root` |

## 分步链

### 阶段一：外部打点

#### 1. 证书 SAN 枚举

- **动作**：访问 `https://<IP>`，查看 443 证书的 SAN 字段。
- **结果**：得到 `www.solar.nyx` 与 `www.sunfriends.nyx` 两个域名。
- **下一跳**：把域名加入 hosts，分别做目录爆破。

#### 2. 目录爆破与备份文件

- **动作**：对两个域名目录爆破，发现登录页和 `database.sql.gz`；下载后查看并破解压缩包。
- **结果**：得到登录凭证；`www.solar.nyx` 登录成功，`www.sunfriends.nyx` 登录失败。
- **下一跳**：登录 `www.solar.nyx` 后台，审计页面源码。

### 阶段二：MQTT 与 DOM XSS

#### 3. 后台源码泄露 MQTT 凭据

- **动作**：对比登录前后的页面源码，发现新增的 MQTT 与 Chart.js 逻辑。
- **关键点**：`wss://www.solar.nyx/wss/`；账号 `user` / `1tEa15klQpTx9Oub6ENG`；订阅 `data`；`JSON.parse` 后把 `solarEnergy`、`consumedEnergy` 写进 `innerHTML`。
- **下一跳**：连接 Broker，确认数据流与 XSS Sink。

#### 4. 验证 DOM XSS

- **动作**：用 MQTT 客户端连接 Broker，向 `data` 发布带 `alert(1)` 的 JSON。
- **结果**：后台页面弹窗，确认 `innerHTML` 导致 DOM XSS。
- **注意**：字段结构要满足前端 `JSON.parse` 后的取值逻辑。

#### 5. 盲打管理员

- **动作**：再次向 `data` 注入外带 Payload，读取其他订阅者的后台源码并回传。
- **结果**：拿到管理员 dashboard 源码、硬编码的 `admin` MQTT 凭据；发现管理员订阅 `record`，页面有 `/records` 路由和“Send record”按钮。
- **下一跳**：以管理员视角侦察 `/records`。

### 阶段三：records 与 PDF

#### 6. 读取 records 页面源码

- **动作**：用 XSS `fetch` 同源 `/records` 页面并外带；分析下载逻辑，尝试路径穿越读取 `/etc/passwd`。
- **结果**：页面是 PDF 下载页；`file` 参数被 `basename()` 限制且只允许 `.json`，穿越返回 404。
- **下一跳**：转而分析 PDF 生成链路。

#### 7. 从 PDF 元数据定位生成器

- **动作**：让 XSS 触发下载 PDF 并外带，用 `exiftool` 等查看元数据。
- **结果**：生成器为 `wkhtmltopdf 0.12.6.1`，对应 CVE-2022-35583（SSRF / 外部资源加载）。
- **下一跳**：确认 `record` 主题的 `chart` 字段如何进入 HTML。

#### 8. 直接向 record 注入

- **动作**：用 admin MQTT 凭据连接 Broker，绕过 Web 直接向 `record` 发布恶意 `chart`。
- **关键点**：后端先把数据存盘，只有下载 PDF 时才渲染；Payload 需闭合原 HTML 属性，可加载 `file://` 或直接把源码外带。
- **结果**：可读取 `index.php` 等后端源码；也可把源码渲染进 PDF 后再下载。
- **下一跳**：继续读 `session.php`、`login.php`、`server.php`。

#### 9. 下载 PDF 触发渲染

- **动作**：在 `/records` 页面点击下载，触发 wkhtmltopdf 渲染被污染的 `chart`。
- **结果**：SSRF / 文件读取生效，源码进入 PDF 或回传至攻击机。
- **注意**：只发 `record` 不下载，后端不会渲染。

### 阶段四：第二入口与 webshell

#### 10. 审计 server.php

- **动作**：读取此前目录爆破到的文件，以及 `www.sunfriends.nyx` 的 `server.php` 源码。
- **结果**：源码含硬编码身份校验凭证；登录后只有命令下拉框，只能执行 `commands/` 下已有文件，不能直接执行任意命令。
- **下一跳**：找命令文件是如何生成的。

#### 11. 发现双端输入

- **动作**：分析 `server.php` 的命令处理逻辑。
- **结果**：执行结果发布到 `server/command/output`；命令来源指向 MQTT 的 `server/command/new`，形成 Web + MQTT 双端输入。
- **下一跳**：用 `server.php` 的凭据登录 Broker。

#### 12. 向命令队列投放文件

- **动作**：复用 `5up3r` / `bloods` 登录 Broker；因不确定主题与输入点，批量向候选主题发送带 `name` 字段的下载命令。
- **结果**：命令文件被写入 `commands/`，并出现在 `server.php` 下拉框。
- **注意**：缺少 `name` 字段会报错；MQTT 只负责落文件，不直接执行。

#### 13. 点击执行拿 webshell

- **动作**：在 `server.php` 选择刚投放的命令并执行，让目标下载反弹 Shell；Web 端触发后回连。
- **结果**：拿到 `www-data` shell。
- **下一跳**：枚举 `doas`，开始横向。

### 阶段五：横向

#### 14. www-data → lenam

- **动作**：读 `/etc/doas.conf`，发现可以 `lenam` 身份执行 `mosquitto_pub`；用 `-f` 把 `user.txt`、`note.txt`、SSH 私钥等发到 `test` 主题再取回。
- **关键点**：从 `nano` 历史拿到 SSH 私钥 passphrase；下载 `.gpg` 文件后用 Hashcat 破解，得到 GPG passphrase 与 `lenam` 密码。
- **结果**：`lenam` / `qiFQI7buDp7zIQnAymEY`，完成第一次横向。

#### 15. lenam → julian

- **动作**：利用 `permit lenam as julian cmd /bin/kill`，向 `julian` 的 Node 进程发送 `SIGUSR1`。
- **结果**：Node 开启 `localhost:9229` 调试端口；用 `node inspect` 连接并执行 JS，以 `julian` 身份反弹 Shell。
- **下一跳**：检查 `julian` 的 `doas` 规则。

### 阶段六：root

#### 16. julian → root

- **动作**：利用 `permit setenv { PATH } julian as root cmd /usr/local/bin/backups`，分析 `backups` 依赖的 `libbackup.so`。
- **关键点**：库内用相对路径调用 `rm`；`doas` 保留调用者 `PATH`。
- **结果**：在 `julian` 的 `PATH` 目录放置恶意 `rm`，`backups` 以 root 执行时触发，拿到 root shell。

## 复习抓手

| 卡点 | 为什么会卡 | 正确做法 |
| --- | --- | --- |
| XSS 外带无数据 | HTTPS 页面请求 HTTP 资源被拦截或升级 | 改用 `location.href` 跳转外带 |
| 外带 URL 返回 414 | `btoa(encodeURIComponent())` 使 URL 过长 | 用 `btoa(String.fromCharCode(...))` 压缩；POST 也会被丢弃 |
| `/records` 路径穿越 404 | `basename()` 拦截穿越，且只允许 `.json` | 读 `index.php` 源码，改从 `record` 注入读文件 |
| 发布 `record` 后无请求 | PDF 只在点击下载时生成 | 先下载 PDF 触发渲染 |
| MQTT 命令没有回显 | 只落命令文件，不执行 | 回 `server.php` 面板选择并执行 |
| 向 `server/command/new` 发送失败 | 权限隔离 | 用 `5up3r` / `bloods`，不是 admin |
| GPG passphrase 猜错 | `user.txt` 里的字符串不是 GPG passphrase | 从 `nano` 历史取 SSH passphrase；用 Hashcat 破解 GPG |
| Node 调试连不上 | 调试接口很快关闭，进程 PID 也可能变化 | 发完信号立刻连；PID 以 `ps` 为准 |

## 凭据速查

| 身份 / 用途 | 凭据 |
| --- | --- |
| MQTT 普通用户 | `user` / `1tEa15klQpTx9Oub6ENG` |
| MQTT 管理员 | `admin` / `tJH8HvwVwC57BR6CEyg5` |
| `server.php` / MQTT | `5up3r` / `bloods` |
| MySQL（`login.php` 内） | `solar_user` / `lD5vkvLfMowAiaT7w64C` |
| SSH / `private_id.gpg` passphrase | `CzMO48xpwof8nvQ6JUhF` |
| GPG passphrase | `loveboy1` |
| `lenam` | `qiFQI7buDp7zIQnAymEY` |
| `julian` | `tk8QaHUi3XaMLYoP1BpZ` |

## 关键 Topic

| Topic | 作用 |
| --- | --- |
| `data` | 前端订阅并写入 `innerHTML`；DOM XSS 注入点 |
| `record` | 前端发布图表数据；后端存盘，下载 PDF 时渲染 `chart` |
| `server/command/new` | MQTT 输入端，生成 `commands/` 下的命令文件 |
| `server/command/output` | `server.php` 执行命令后发布结果 |
| `test` | 用 `mosquitto_pub -f` 外带文件 |
