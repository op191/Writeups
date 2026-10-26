1.发现web端口开放443, 查看证书SAN字段,发现两个域名  
2.同时对两个域名进行目录爆破,发现登录页和database.sql,gz敏感文件,破解密码得到凭证,  
3.登录web后台,发现源码变了,源码发现订阅了mqtt的broker的data主题并发现前端用innerHTML函数,接受broker,传来的两个字段,可能存在XSS Dom-based  
4.mqtt客户端连上broker的data,注入XSS payload alert()检验XSS,网页回显,存在XSS  
5.再次注入XSS到data主题,获取其他订阅了data主题的用户web后台源码,拿到一个管理员后台源码和硬编码在源码里的mqtt凭据,并发现管理员订阅了record主题和发现管理员页面存在一个路由目录records和将页面截图发送到broker的record主题的一键按钮

6.XSS拿下records目录的源码,是个下载pdf的页面,并且点击下载后发送GET请求,尝试路径穿越注入键值对读取/etc/passwd,回显404,换路  
7.构造XSS下载pdf并传回我的web服务,exiftool或者pdf元数据,发现pdf生成器存在CVE,读取前端回传chart字段到record主题的数据渲染HTML,  
8.用管理员mqtt凭证脸上broker,,跳过web,直接注入record主题的chart字段读取index.php后端源码,pdf生成器只是存盘数据,没有执行渲染HTML,注入payload可以直接数据外带,也可以将源码渲染在pdf上,再去下载pdf,  
9.XSS在records页面下载pdf文件,触发pdf生成器执行HTML渲染,  
10.再读取之前目录扫描的到的文件源码和另一个域名的源码server.php,发现硬编码凭证做身份校验代码,登录另一个域名的后台server.php,发现命令执行框,只能选择已有的命令,无法进行恶意命令执行,换路  
11.在server.php发现后端处理命令执行时候会将处理结果发送到/server/command/output主题,猜想可能是存在双端输入(web端和mqtt端)

12.因为源码连着处理继续server.php凭证复用登录到broker,批量上传反弹访问我们的主机下载反弹shell的payload,到broker的未显示主题,因为不知道哪个有哪些主题存在和不知道哪个主题是输入点

13.发现server.php的命令执行框中出现我们注入的文件,点击执行,触发下载反弹shell,  
web触发拿到shell,  
14,`/etc/doas.conf`提权,发现一个用户可以冒充另一个用户的套娃,冒充用户使用mosquitto_pub访问文件,并发送broker查看,发现passphrase和密钥,gpg下载后本地破解,拿到用户密码,横向  
15.用上一个用户的身份冒充另一用户用kill命令发送信号给nodejs开调试模式,反弹shell给攻击者,  
16.以julian身份运行doas的backup命令提权,发现一个不常见的动态库,在这动态库里发现相对路径命令直接,并且doas允许继承环境变量,hijack PATH提权为root
