
### 推荐顺序

| 顺序  | 脚本                                           | 难度  | 说明                               |
| --- | -------------------------------------------- | --- | -------------------------------- |
| 1   | `WeakServicePermissions.ps1`                 | 入门  | 弱服务权限，思路和 Unquoted 类似，但更直接       |
| 2   | `WeakServiceBinaryPermissions.ps1`           | 入门  | 服务二进制文件权限弱，替换 exe 即可             |
| 3   | `AlwaysInstallElevated.ps1`                  | 简单  | 注册表策略错误，用 MSI 提权                 |
| 4   | `WeakRegistryPermissions.ps1`                | 简单  | 注册表键权限弱，可改服务配置                   |
| 5   | `SeImpersonatePrivilege.ps1`                 | 中等  | 土豆系列（JuicyPotato/RoguePotato），经典 |
| 6   | `SeBackupPrivilege.ps1`                      | 中等  | 备份权限读 SAM/SYSTEM，配合 diskshadow   |
| 7   | `StoredCredentialsRunas.ps1`                 | 中等  | runas 存储凭证，cmdkey 泄露             |
| 8   | `StoredCredentialsWinlogon.ps1`              | 中等  | Winlogon 注册表存明文密码                |
| 9   | `HardcodedCredentialsDotNetApp.ps1`          | 中等  | .NET 应用硬编码凭证                     |
| 10  | `HardcodedCredentialsJavaApp.ps1`            | 中等  | Java 应用硬编码凭证                     |
| 11  | `LogonAutostartExecutionRegistryRunKeys.ps1` | 中等  | Run 键自启动                         |
| 12  | `LogonAutostartExecutionStartupFolder.ps1`   | 简单  | 启动文件夹自启动                         |
| 13  | `AnswerFiles.ps1`                            | 简单  | 无人值守应答文件泄露密码                     |
| 14  | `EnableLocalAdmin.ps1`                       | 简单  | 启用本地管理员                          |
| 15  | `EnableUAC.ps1`                              | 简单  | UAC 配置                           |
| 16  | `TurnOffAV.ps1`                              | 辅助  | 关杀软，不是漏洞                         |
