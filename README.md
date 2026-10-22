# PHP74Patch
Patch for high-risk remote code execution vulnerabilities in PHP 7.4, such as recently exposed CVE-226-109618 and CVE-226-109619

# PHP 7.4.33 Security Patch Update & Rollback Operations Guide (Remi Replacement)

This patch is designed to mitigate critical vulnerabilities in PHP 7.4.33 (including **CVE-2026-109618** and **CVE-2026-109619**). 
To address the fundamental incompatibility between OpenSSL 3.x and older PHP lifecycles on **CentOS Stream 9**, this patch leverages a **Twin-Track Hybrid Compilation Strategy**. This ensures the Apache `mod_php` module (`libphp74.so`) is securely patched while maintaining 100% binary compatibility with your existing Remi NTS extensions (such as `bcmath`, `curl`, `pdo`, etc.), yielding a zero-warning, seamless upgrade.

---

## 1. Quick Upgrade Steps (Only 2 Commands)

Upload `php74-remi-security-patch.tar.gz` to the `/tmp` directory of the target server, then execute the following commands with `root` privileges:

```bash
cd /tmp && tar -zxvf php74-remi-security-patch.tar.gz && cd php74-remi-patch/
sudo ./install_patch.sh
```

---

## 2. Post-Upgrade Verification

### A. Verify the Command Line Interface (CLI)
Execute the following command to inspect the runtime status of core extensions:
```bash
/opt/remi/php74/root/usr/bin/php -v
```
* **Expected Result**: The output must show the standard PHP 7.4.33 version info **without any** startup warnings like `Unable to load dynamic library`.

### B. Verify the Apache Web Environment
Temporarily create an `info.php` file in your website's document root directory:
```php
<?php phpinfo(); ?>
```
Access this page via a web browser and verify the following sections:
* **Build Date**: It must reflect the timestamp when you compiled this patch (instead of Remi's original upstream build date).
* **Configure Command**: It must contain `--with-openssl=/opt/openssl11` along with your custom parameters, confirming that Apache has successfully loaded the newly built security module.
* *Security Notice: Delete the `info.php` file immediately after verification to prevent server configuration exposure.*

---

## 3. Emergency Rollback Strategy (Downtime < 10 Seconds)

If any unexpected business logic failure or application regression occurs post-upgrade, this patch provides a **lossless rollback mechanism**. Since the `install_patch.sh` script automatically creates backup clones of the original Remi binaries prior to replacement, you can instantly revert the system via the following commands:

```bash
# 1. Revert the Remi official CLI core
if [ -f /opt/remi/php74/root/usr/bin/php.remi.bak ]; then
    mv -f /opt/remi/php74/root/usr/bin/php.remi.bak /opt/remi/php74/root/usr/bin/php
fi

# 2. Revert the Remi official Apache module
if [ -f /usr/lib64/httpd/modules/libphp74.so.bak ]; then
    mv -f /usr/lib64/httpd/modules/libphp74.so.bak /usr/lib64/httpd/modules/libphp74.so
    # Restore the original file's security context
    restorecon -v /usr/lib64/httpd/modules/libphp74.so
fi

# 3. Restart the Apache daemon
systemctl restart httpd

echo "[✓] System rolled back successfully to the original Remi PHP 7.4.33 environment."
```

---

## 4. Troubleshooting & FAQ

### Q1: The website throws a 500 Internal Server Error, or Apache fails to start after upgrading?
* **Root Cause**: The **SELinux** subsystem on CentOS Stream 9 is blocking Apache from executing the newly copied `libphp74.so` binary due to missing or mismatched security contexts.
* **Resolution**: Manually restore the standard SELinux security context for the Apache module folder:
  ```bash
  restorecon -v /usr/lib64/httpd/modules/libphp74.so
  systemctl restart httpd
  ```

### Q2: Why does this patch bundle an isolated `/opt/openssl11` directory?
* **Technical Reason**: CentOS Stream 9 ships natively with OpenSSL 3.x. The legacy PHP 7.4 source code cannot hook into the breaking API changes introduced in OpenSSL 3.x, causing compilation failures.
* **Stability Assurance**: This patch isolates the dedicated OpenSSL 1.1.1w static dependency within the `/opt/openssl11` chroot folder, **exclusively accessible by this patched PHP core**. It will never interfere with or pollute your server's global OpenSSL 3.x ecosystem, preserving host OS integrity.
