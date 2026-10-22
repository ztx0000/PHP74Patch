# PHP74Patch
Patch for high-risk remote code execution vulnerabilities in PHP 7.4, such as recently exposed CVE-226-109618 and CVE-226-109619

# PHP 7.4.33 安全补丁更新与回滚操作指南 (Remi 替换版)

本补丁包用于修复 PHP 7.4.33 在现代 Linux 系统中的安全漏洞（包含 **CVE-2026-109618** 与 **CVE-2026-109619**）。
针对 **CentOS Stream 9** 环境下 OpenSSL 3.x 与旧版 PHP 的不兼容问题，本补丁通过**双轨混合编译技术**，既修复了 Apache `mod_php` (`libphp74.so`) 模块，又完美兼容了系统原装的所有 Remi NTS 扩展（如 `bcmath`, `curl`, `pdo` 等），实现 0 报错平滑升级。

---

## 一、 快速升级步骤 (仅需 2 行命令)

请将 `php74-remi-security-patch.tar.gz` 上传至目标服务器的 `/tmp` 目录，随后以 `root` 权限执行：

```bash
cd /tmp && tar -zxvf php74-remi-security-patch.tar.gz && cd php74-remi-patch/
sudo ./install_patch.sh
```

---

## 二、 升级验证方法

### 1. 验证命令行环境 (CLI)
执行以下命令检查 PHP 版本及扩展加载情况：
```bash
/opt/remi/php74/root/usr/bin/php -v
```
* **预期结果**：输出正常的 PHP 7.4.33 版本信息，且**没有任何**类似 `Unable to load dynamic library` 的警告提示。

### 2. 验证 Apache 网页端环境
在网站根目录下临时创建一个 `info.php` 文件：
```php
<?php phpinfo(); ?>
```
通过浏览器访问该页面，检查以下两项：
* **Build Date**：应显示为您编译该补丁包的最新时间（非 Remi 官方原始打包时间）。
* **Configure Command**：应包含 `--with-openssl=/opt/openssl11` 以及您自定义的编译参数，证明 Apache 已经成功载入了全新的安全模块。
* *验证完毕后请立即删除 `info.php` 文件，避免泄露服务器隐私。*

---

## 三、 紧急回滚方案 (耗时 < 10 秒)

如果在升级后发现网站业务出现异常，本补丁内置了**无损回滚机制**。由于 `install_patch.sh` 在执行时已自动将 Remi 官方原始二进制文件备份为 `.bak`，您只需执行以下命令即可瞬间还原系统：

```bash
# 1. 还原 Remi 官方命令行核心
if [ -f /opt/remi/php74/root/usr/bin/php.remi.bak ]; then
    mv -f /opt/remi/php74/root/usr/bin/php.remi.bak /opt/remi/php74/root/usr/bin/php
fi

# 2. 还原 Remi 官方 Apache 模块
if [ -f /usr/lib64/httpd/modules/libphp74.so.bak ]; then
    mv -f /usr/lib64/httpd/modules/libphp74.so.bak /usr/lib64/httpd/modules/libphp74.so
    # 恢复原文件安全上下文
    restorecon -v /usr/lib64/httpd/modules/libphp74.so
fi

# 3. 重启 Apache 服务
systemctl restart httpd

echo "[✓] 系统已成功回滚至原始 Remi PHP 7.4.33 环境。"
```

---

## 四、 常见故障排查

### Q1: 升级后网页访问报 500 错误，或 Apache 无法启动？
* **可能原因**：CentOS Stream 9 的 **SELinux** 阻止了 Apache 载入新复制的 `libphp74.so` 模块（通常是因为安全上下文丢失）。
* **解决方案**：手动为该模块重置 SELinux 标签：
  ```bash
  restorecon -v /usr/lib64/httpd/modules/libphp74.so
  systemctl restart httpd
  ```

### Q2: 为什么这个补丁包里会带有一个 `/opt/openssl11` 目录？
* **原理解析**：CentOS Stream 9 默认自带的是 OpenSSL 3.x，而 PHP 7.4 源码在底层无法原生兼容其 API，直接编译会引发严重的编译中断。
* **安全性保证**：本补丁包将专用的 OpenSSL 1.1.1w 静态库严格隔离在 `/opt/openssl11` 目录内，**仅供新版 PHP 核心调用**，绝对不会污染或篡改您目标服务器全局的 OpenSSL 3.x 环境，确保了操作系统的整体稳定性。
