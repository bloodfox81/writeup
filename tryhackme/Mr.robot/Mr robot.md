# THM Mr. Robot CTF - 渗透测试 Writeup

## 目标信息
- **IP**: 10.49.163.177
- **平台**: TryHackMe (THM)
- **类型**: Mr. Robot CTF
- **日期**: 2026-05-21
- **授权**: THM 授权靶场，全 IP 范围，无限制

## 攻击链总结

### 1. 信息收集
- **端口扫描**: 22/80/443 开放
- **服务识别**: Apache httpd, WordPress 4.3.1
- **目录扫描**: gobuster 发现 `/wp-admin`, `/wp-login.php`, `/license`, `/robots.txt`

### 2. 凭证获取
- **发现路径**: `/license` 文件
- **提取方法**: base64 解码隐藏在大量空白字符后的内容
- **获取凭据**: `elliot:ER28-0652`

### 3. WordPress 登录
- **用户名**: Elliot（首字母大写）
- **密码**: ER28-0652
- **权限**: 管理员（可编辑主题）

### 4. Web Shell 获取
- **方法**: 编辑 twentyfifteen 主题文件（404.php）
- **Payload**: PHP 反向 shell
- **监听**: `nc -lvnp 4444`
- **获取用户**: daemon（Bitnami WordPress 运行用户）

### 5. 密码破解
- **发现文件**: `/home/robot/password.raw-md5`
- **Hash**: `robot:c3fcd3d76192e4007dfb496cca67e13b`
- **破解工具**: hashcat + rockyou.txt
- **破解结果**: `abcdefghijklmnopqrstuvwxyz`
- **耗时**: 2 秒

### 6. 用户切换
- **方法**: SSH 登录
- **用户名**: robot
- **密码**: abcdefghijklmnopqrstuvwxyz
- **获取 Flag 2**: `822c73956184f694993bede3eb39f959`

### 7. 权限提升
- **枚举**: `sudo -l` 无权限，`find / -perm -4000` 发现 SUID nmap
- **漏洞**: `/usr/local/bin/nmap` 为 root 拥有的 SUID 文件
- **提权方法**: `nmap --interactive` → `!sh`
- **获取用户**: root

### 8. Flag 3 获取
- **位置**: `/root/key-3-of-3.txt`
- **Flag 3**: `04787ddef27c3dee1ee161b21670b4e4`

## 获取的 Flags

| Flag   | 值                                 | 位置                         |
| ------ | ---------------------------------- | ---------------------------- |
| Flag 1 | `073403c8a58a1f80d943455fb30724b9` | `/key-1-of-3.txt`            |
| Flag 2 | `822c73956184f694993bede3eb39f959` | `/home/robot/key-2-of-3.txt` |
| Flag 3 | `04787ddef27c3dee1ee161b21670b4e4` | `/root/key-3-of-3.txt`       |

## 关键技术点

### 1. 隐藏信息提取
- `/license` 文件使用大量空白字符隐藏 base64 编码的凭据
- 需要仔细检查文件内容，不可仅凭表面判断

### 2. WordPress 主题编辑 Getshell
- 管理员权限下编辑主题文件是常见的 getshell 方法
- 选择 `404.php` 或 `header.php` 等不常访问的文件降低被发现风险

### 3. MD5 快速破解
- `c3fcd3d76192e4007dfb496cca67e13b` 是常见密码的 MD5
- 使用 rockyou.txt 字典可快速破解

### 4. SUID Nmap 提权
- `/usr/local/bin/nmap` 为 root 拥有的 SUID 文件是经典提权点
- `nmap --interactive` → `!sh` 可获取 root shell
- 这是 CTF 中常见的故意配置漏洞

## 工具使用

| 工具     | 用途            |
| -------- | --------------- |
| nmap     | 端口扫描        |
| gobuster | 目录扫描        |
| curl     | HTTP 请求       |
| hashcat  | MD5 破解        |
| nc       | 反向 shell 监听 |
| wpscan   | WordPress 扫描  |

## 状态文件

- `target.md` - 目标信息
- `findings.md` - 完整发现记录
- `commands.md` - 命令历史
- `report.md` - 报告草稿
