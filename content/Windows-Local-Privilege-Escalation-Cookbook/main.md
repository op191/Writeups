
```powershell
Expand-Archive          # 解压
Compress-Archive        # 压缩
Invoke-WebRequest       # 下载
Get-ChildItem           # 列目录（ls/dir）
Set-Location            # 切目录（cd）
Get-Command             # 查命令（which）
```

```bash
zip -r 压缩包名字 压缩文件1 压缩文件2 ...  #-r是压缩目录
```

win自带curl,但是curl不支持递归下载目录,只允许单个下载文件,所以要下载目录需要把那个目录压缩成单个压缩包文件,再用curl.exe下载这个压缩包,

win的解压工具默认.zip ,linux要用zip工具压缩成这种形式,

```cmd
net user 用户名 密码 /add #创建一个用户
```

```powershell
net localgroup 组名 用户名 /add #用户加入组
```

```powershell
sc.exe qc 服务名 #查询服务的配置信息
```

```powershell
sc.exe query 服务名字 #查询服务的当前状态
```

#### 把文件的内容复制过去,不动文件名字,适用于可修改服务文件内容(WP)

```powershell
cp C:\Program.exe "C:\Program Files\CustomSrv2\Service2.exe"
```

#### msi升级安装,AlwaysInstallElevated策略需要同时开启HKLM,和HKCU,下的

#### \SOFTWARE\Policies\Microsoft\Windows\Installer

### 才能利用,如果查询这两个结果有HKCU报错,那就利用不了

```powershell
msiexec /quiet /qn /i C:\Temp\evil.msi
```

/quiet静默安装 /qn 不显示ui /i 安装 ,这个是利用搭配,死记忆

当你安装一个服务时，Windows 会在系统里创建**两个东西**：

| 东西       | 位置                                           | 作用            |
| -------- | -------------------------------------------- | ------------- |
| **服务对象** | SCM（服务控制管理器）内部                               | 控制服务的启动、停止、配置 |
| **注册表键** | `HKLM\SYSTEM\CurrentControlSet\Services\服务名` | 存储服务的配置数据     |

两者**互相独立**，各有各的 ACL（权限列表）。

### 两个漏洞的区别

|             | WeakServicePermissions | WeakRegistryPermissions        |
| ----------- | ---------------------- | ------------------------------ |
| **攻击哪个**    | 服务对象                   | 注册表键                           |
| **改什么**     | 服务的 binPath（通过 SCM）    | 注册表 ImagePath（直接改文件）           |
| **用啥工具**    | `sc.exe config`        | `reg add` / `Set-ItemProperty` |
| **本质**      | 你让 SCM 改服务指向           | 你直接改配置文件                       |
| **SCM 知道吗** | 知道，是正规流程               | 不知道，你绕过了 SCM                   |

### 服务对象”是什么

“服务对象”指的是**每一个具体的服务**，在 SCM 内部被表示为一个**对象**。每个服务对象有：

- **自己的安全描述符（ACL）**：决定谁能启动、停止、配置这个服务

- **自己的状态**：RUNNING、STOPPED、START_PENDING 等

- **自己的配置信息**：binPath、启动类型、依赖关系等

东西组成：**SCM 里的服务对象、注册表里的配置、以及运行时的进程**。

### 

┌─────────────────────────────────────────────────────────┐
│  SCM (services.exe)                                      │
│  ┌─────────────────────────────────────────────────┐    │
│  │  服务对象 (Service Object)                        │    │
│  │  ├── 服务名 (Service Name)      : Vulnerable Service 4 │
│  │  ├── 显示名 (Display Name)      : Vuln Service 4       │
│  │  ├── 状态 (State)               : Stopped              │
│  │  ├── 安全描述符 (SDDL/ACL)      : D:(A;;RPWP;;;BU)...  │
│  │  └── 配置引用 → 注册表键                              │
│  └─────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────┘
                          │
                          │ 读取配置
                          ▼
┌─────────────────────────────────────────────────────────┐
│  注册表键                                                 │
│  HKLM\SYSTEM\CurrentControlSet\Services\Vulnerable Service 4 │
│  ├── ImagePath      : C:\Program Files\...\Service4.exe  │
│  ├── DisplayName    : Vuln Service 4                     │
│  ├── Start          : 2 (AUTO_START)                     │
│  ├── ObjectName     : LocalSystem                        │
│  └── (ACL: 谁有权改这个键)                                 │
└─────────────────────────────────────────────────────────┘
                          │
                          │ 启动时创建
                          ▼
┌─────────────────────────────────────────────────────────┐
│  运行时进程                                               │
│  Service4.exe (PID: xxxx)                                │
│  ├── 运行身份  : NT AUTHORITY\SYSTEM                     │
│  ├── 父进程    : services.exe (SCM)                      │
│  └── 响应 SCM 的控制码 (START/STOP/PAUSE)                 │
└─────────────────────────────────────────────────────────┘整体结构图
text

┌─────────────────────────────────────────────────────────┐
│  SCM (services.exe)                                      │
│  ┌─────────────────────────────────────────────────┐    │
│  │  服务对象 (Service Object)                        │    │
│  │  ├── 服务名 (Service Name)      : Vulnerable Service 4 │
│  │  ├── 显示名 (Display Name)      : Vuln Service 4       │
│  │  ├── 状态 (State)               : Stopped              │
│  │  ├── 安全描述符 (SDDL/ACL)      : D:(A;;RPWP;;;BU)...  │
│  │  └── 配置引用 → 注册表键                              │
│  └─────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────┘
                          │

把注册表想象成文件系统：

text

HKLM:\...\Services\Vulnerable Service 4    ← 键（文件夹）
    └── ImagePath                          ← 值（文件）

- `-Path` 指向文件夹

- `-Name` 指定文件夹里的文件名



|        | Linux         | Windows IIS                       |
| ------ | ------------- | --------------------------------- |
| Web服务器 | Nginx/Apache  | IIS(W3SVC)                        |
| 应用运行环境 | PHP-FPM/uWSGI | 应用程序池(w3wp.exe)                   |
| 配置文件   | nginx.conf    | applicationHost.config/web.config |
| 权限隔离   | www-data用户    | IIS AppPool\xxx虚拟账号               |
| 管理工具   | 命令行           | IIS Manager图形界面+appcmd            |



- #### RPC → 远程调用**函数**
- #### DCOM → 远程使用**对象**(底层用RPC通信)

```
COM对象  =  Windows把系统功能封装成的"工具箱"
DCOM     =  通过网络远程使用这些工具箱
RPC      =  DCOM用来传输调用请求的底层管道
135端口  =  "工具箱目录"(告诉你某个工具箱在哪个端口)
```

DCOM远程使用COM对象：

```
攻击机 → 网络 → 目标上的Shell对象.ShellExecute("恶意命令") → 在靶机执行
```

# **COM对象是什么**

Windows里微软把大量系统功能封装成了"COM对象"，让任何程序都能调用这些功能，不用自己重新实现。比如：

```
任务计划程序COM对象
└── 方法: CreateTask()、DeleteTask()、RunTask()

Shell COM对象
└── 方法: ShellExecute()、CreateFolder()、CopyFile()

WMI COM对象
└── 方法: 查询系统信息、执行命令、监控事件
```

**"使用对象"是什么意思**

就是**调用这个对象提供的方法**，让它帮你干事：

python

```python
# 伪代码
shell = 创建Shell对象()
shell.ShellExecute("calc.exe")  # 调用它的方法,让它执行计算器
```

### 编程里的"对象"----把**数据(属性)**和**能做的操作(方法)**打包在一起，形成一个独立的单元：

```
打印机对象
├── 属性: brand="Canon", ink=80%
└── 方法: print()、scan()、cancel()
```

# RPC/DCOM机制利用(土豆攻击)

## 调用IWbemServices接口(某个DCOM接口)

### → rpcss开始解析对象位置

### → GodPotato拦截OXID解析过程,把地址指向自己控制的命名管道

### → rpcss(SYSTEM)连接到这个命名管道

### → GodPotato调用ImpersonateNamedPipeClient捕获SYSTEM模拟令牌 ,ImpersonateNamedPipeClient(需要用户有SeImpersonatePrivilege)

### → DuplicateToken复制创建主令牌 (将模拟令牌造成主令牌)

### → CreateProcessWithTokenW用SYSTEM主令牌启动cmd.exe

### → 获得SYSTEM权限的shell



##### **XID Resolver 是 RPCSS（Remote Procedure Call Service）进程的一个核心组件**

```
rpcss.dll
  ├── 对象激活组件
  ├── OXID Resolver 组件  ← 土豆攻击利用的就是这个
  └── 垃圾回收组件
```

Running  RpcSs              Remote Procedure Call (RPC)

### C:\Windows\temp是所有用户可写文件夹,用来上传恶意软件的

### iis用户权限都很低,在iis服务的webshell里rce下载的木马,放在C:\Windows\temp最好,其他目录还要考虑权限问题





## `SeBackupPrivilege`（**Back up files and directories**）最核心、最确定的作用是：

> ##### **允许进程以“备份语义”访问文件和目录，从而绕过部分 NTFS/DACL 的读取限制。**

### 直接导出sam和system.hive做密钥

```
reg save hklm\sam C:\Users\ncv\Documents\sam.hive
reg save hklm\system C:\Users\ncv\Documents\system.hive
```

#### 或者用**diskshadow**,不过这个需要管理员权限

### 创建系统磁盘快照,下载快照文件

- **`reg save`** 生成的是一个**具名文件**，比如 `sam.hive`，安全软件和应急响应人员很容易发现。

- **VSS 快照** 是系统级功能，本身是 Windows 正常机制（备份软件也用它），快照存储区不在常规文件系统里，需要专门工具才能访问。



### 系统运行时SAM数据库一直被lsass.exe独占

`2026/09/29 20:33:59 ERROR 32 (0x00000020) Copying File c:\Windows\System32\config\SAM
The process cannot access the file because it is being used by another process.`

### 直接从c盘备份会失败,尽管是管理员,也是失败

```
robocopy /b c:\Windows\System32\config C:\download SAM SYSTEM
```

#### 所以要先创建磁盘快照,备份c盘文件在一个新的磁盘里,然后用robocop再去备份的自己能访问到的盘里,

**`robocopy /b` 不能“直接备份 C 盘”，它只是用备份权限复制指定文件。要复制被锁的 SAM，必须先通过快照拿到无锁副本，再从快照复制。**

#### `SeBackupPrivilege`只有这个权限是无法用diskshadow.exe创建快照的


