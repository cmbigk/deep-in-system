Here is the complete, consolidated guide with every correction and addition included.

***

## Phase 1: VM Setup

**What to learn:** VirtualBox basics, Ubuntu Server installation, LVM partitioning.

1. Download **Ubuntu Server 24.04 LTS** ISO
2. Create a new VM — **30 GB disk**, set network adapter to Bridged or Host-Only
3. During installation, select **"Custom storage layout"**. Create a small EFI partition (~512 MB) and a /boot partition (~1 GB), then assign all remaining space to an **LVM Physical Volume**
4. Inside the LVM volume group, create these logical volumes **exactly**:

| Volume | Size | Format | Mount |
|--------|------|--------|-------|
| lv-swap | 4 GB | swap | — |
| lv-root | 15 GB | ext4 | `/` |
| lv-home | 5 GB | ext4 | `/home` |
| lv-backup | 6 GB | ext4 | `/backup` |

5. Set **username** = your exact school login name
6. Set **hostname** = `{your-login}-host` (e.g. `luffy-host`)

> 💡 **Golden habit:** Before modifying any config file, always back it up first:
> `sudo cp /etc/ssh/sshd_config /etc/ssh/sshd_config.bak`

***

## Phase 2: Static IP

**What to learn:** Netplan configuration on Ubuntu Server.

7. Check your interface name: `ip a`
8. Edit `/etc/netplan/00-installer-config.yaml`:
```yaml
network:
  version: 2
  ethernets:
    enp0s3:        # replace with your interface name
      dhcp4: no
      addresses: [192.168.1.50/24]
      gateway4: 192.168.1.1
      nameservers:
        addresses: [8.8.8.8, 1.1.1.1]
```
9. Apply: `sudo netplan apply`
10. Verify internet: `ping -c 5 google.com`

***

## Phase 3: Hostname Verification

11. Set hostname precisely:
```bash
sudo hostnamectl set-hostname {your-login}-host
```
12. Update `/etc/hosts` — find the `127.0.1.1` line and set it to:
```
127.0.1.1   {your-login}-host
```
13. Verify: `hostnamectl` — output must show `Static hostname: {your-login}-host`

***

## Phase 4: Firewall (UFW)

**What to learn:** UFW default deny, selective port opening.

> ⚠️ Do this **before** changing the SSH port, or you will lock yourself out.

14. Run in order:
```bash
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow 2222/tcp
sudo ufw allow 80/tcp
sudo ufw allow 20/tcp
sudo ufw allow 21/tcp
sudo ufw allow 40000:50000/tcp
sudo ufw enable
sudo ufw status verbose
```

**Port justifications — memorize these for the audit:**

| Port | Reason |
|------|--------|
| 2222 | SSH (moved from default 22 for security) |
| 80 | HTTP for WordPress |
| 20, 21 | FTP data and control (vsftpd) |
| 40000–50000 | FTP passive mode port range |

***

## Phase 5: SSH Security

**What to learn:** `sshd_config` directives, SSH hardening.

15. Backup first: `sudo cp /etc/ssh/sshd_config /etc/ssh/sshd_config.bak`
16. Edit `/etc/ssh/sshd_config`:
```
Port 2222
PermitRootLogin no
PasswordAuthentication yes
PubkeyAuthentication yes
```
17. Restart SSH: `sudo systemctl restart ssh`

***

## Phase 6: User Management

**What to learn:** `useradd`, `usermod`, SSH key generation, `.ssh` directory permissions.

**Create luffy** (sudo + key-based auth):
```bash
sudo useradd -m -d /home/luffy -s /bin/bash luffy
sudo usermod -aG sudo luffy

# On your HOST machine, generate the key pair:
ssh-keygen -t ed25519 -f ~/.ssh/luffy_key

# Copy public key to the server:
sudo mkdir -p /home/luffy/.ssh
sudo cp ~/.ssh/luffy_key.pub /home/luffy/.ssh/authorized_keys
sudo chown -R luffy:luffy /home/luffy/.ssh
sudo chmod 700 /home/luffy/.ssh
sudo chmod 600 /home/luffy/.ssh/authorized_keys
```

**Create zoro** (password auth, no sudo):
```bash
sudo useradd -m -d /home/zoro -s /bin/bash zoro
sudo passwd zoro    # set and remember your custom password
```

> 🔑 Keep `luffy_key` (private key file) and zoro's password ready for the audit.

***

## Phase 7: FTP Server (vsftpd)

**What to learn:** vsftpd, chroot jail, userlist, read-only config.

18. Backup config: `sudo cp /etc/vsftpd.conf /etc/vsftpd.conf.bak`
19. Install and create nami:
```bash
sudo apt install vsftpd
sudo useradd -m -d /backup -s /sbin/nologin nami
sudo passwd nami    # set and remember your custom password
```
20. Edit `/etc/vsftpd.conf`:
```
anonymous_enable=NO
local_enable=YES
write_enable=NO
chroot_local_user=YES
allow_writeable_chroot=YES
userlist_enable=YES
userlist_deny=NO
userlist_file=/etc/vsftpd.userlist
pasv_min_port=40000
pasv_max_port=50000
local_root=/backup
```
21. Create `/etc/vsftpd.userlist` and add only: `nami`
22. Restart: `sudo systemctl restart vsftpd`

> 🔑 Keep nami's password ready for the audit.

***

## Phase 8: MySQL Database

**What to learn:** MySQL installation, user/privilege management, bind-address.

23. Install:
```bash
sudo apt install mysql-server
sudo mysql_secure_installation
```
24. Log in and create the WordPress database and user:
```sql
sudo mysql

CREATE DATABASE wordpress_db;
CREATE USER 'wp_user'@'localhost' IDENTIFIED BY 'your_strong_password';
GRANT ALL PRIVILEGES ON wordpress_db.* TO 'wp_user'@'localhost';
FLUSH PRIVILEGES;
EXIT;
```
25. Backup and then confirm local-only binding:
```bash
sudo cp /etc/mysql/mysql.conf.d/mysqld.cnf /etc/mysql/mysql.conf.d/mysqld.cnf.bak
grep bind-address /etc/mysql/mysql.conf.d/mysqld.cnf
# Must show: bind-address = 127.0.0.1
```
26. `sudo systemctl restart mysql`

***

## Phase 9: WordPress

**What to learn:** Apache, PHP, WordPress installation, `.htaccess` security.

27. Install the LAMP stack:
```bash
sudo apt install apache2 php php-mysql libapache2-mod-php php-curl php-gd php-mbstring php-xml
sudo a2enmod rewrite
sudo systemctl restart apache2
```
28. Download and place WordPress:
```bash
cd /tmp
wget https://wordpress.org/latest.tar.gz
tar -xvf latest.tar.gz
sudo cp -r wordpress/* /var/www/html/
sudo rm /var/www/html/index.html
sudo chown -R www-data:www-data /var/www/html/
sudo chmod -R 755 /var/www/html/
```
29. Configure WordPress:
```bash
sudo cp /var/www/html/wp-config-sample.php /var/www/html/wp-config.php
sudo nano /var/www/html/wp-config.php
```
Set these values:
```
DB_NAME     = wordpress_db
DB_USER     = wp_user
DB_PASSWORD = your_strong_password
DB_HOST     = localhost
```
30. Block public access to `wp-config.php` — add to `/var/www/html/.htaccess`:
```apache
<Files wp-config.php>
    Order Allow,Deny
    Deny from all
</Files>
```
31. Verify:
```bash
# Should return 403:
curl -o /dev/null -s -w "%{http_code}" http://{your-ip}/wp-config.php
```

***

## Phase 10: Backup Cron Job

**What to learn:** Bash scripting, cron syntax, `mysqldump`, `tar`, exit code checking (`$?`).

32. Create `/usr/local/bin/wp_backup.sh`:
```bash
#!/bin/bash
DATE=$(date +%Y-%m-%d_%H-%M-%S)
DUMP_FILE="/backup/wordpress_db_$DATE.sql"
BACKUP_FILE="/backup/wordpress_db_$DATE.tar.gz"

mysqldump -u wp_user -p'your_strong_password' wordpress_db > "$DUMP_FILE"

if [ $? -eq 0 ]; then
    tar -czf "$BACKUP_FILE" -C /backup "wordpress_db_$DATE.sql"
    rm "$DUMP_FILE"
    echo "[$DATE] Backup successful: $BACKUP_FILE" >> /var/log/backup.log
else
    echo "[$DATE] Backup FAILED for wordpress_db" >> /var/log/backup.log
fi
```

**Know every line for the audit:**
- `DATE=$(date ...)` — no spaces around `=`, stores timestamp
- `mysqldump ...` — exports DB to a `.sql` file
- `$? -eq 0` — checks exit code of previous command (0 = success)
- `tar -czf` — creates compressed archive (`-c` create, `-z` gzip, `-f` filename)
- `rm "$DUMP_FILE"` — cleans up the intermediate `.sql` after archiving
- `>> /var/log/backup.log` — appends (never overwrites) the log

33. Make executable: `sudo chmod +x /usr/local/bin/wp_backup.sh`
34. **Test it manually first:** `sudo /usr/local/bin/wp_backup.sh`
35. Check: `cat /var/log/backup.log` and `ls /backup/`
36. Add cron job: `sudo crontab -e`
```
0 0 * * * /usr/local/bin/wp_backup.sh
```
Cron breakdown: `0 0 * * *` = minute 0, hour 0, every day = daily at midnight.

***

## Phase 11: Documentation & Submission

**What to learn:** Markdown, `sha1sum`, `cat -e`.

37. Write `README.md` — for every service cover:
    - What it does and why it's needed
    - Which config files you changed and what each setting means
    - Every command used with explanation of all flags

38. Save your command history: `history > setup_commands.txt`

39. Export your VM: **VirtualBox → File → Export Appliance → save as `.ova`**

40. Generate and verify the SHA1 file:
```bash
sha1sum deep-in-system.ova > deep-in-system.sha1
cat deep-in-system.sha1 | cat -e
```
Expected output format: `a3f9...255bfef9560  deep-in-system.ova$`
- The `$` at the end is `cat -e` showing a clean Unix line ending — this is correct
- If you see `^M$`, your file has Windows line endings — fix with: `sed -i 's/\r//' deep-in-system.sha1`

41. Push to your repo: `deep-in-system.sha1` and `README.md`

***

## Pre-Audit Final Checklist

Run all of these before the audit session:

```bash
hostnamectl | grep hostname                              # must match {login}-host
ip a                                                     # no DHCP interfaces
sudo ufw status verbose                                  # only required ports open
sudo ss -tlnp                                            # verify active listeners
sudo sshd -T | grep -E "port|permitrootlogin"            # port 2222, root disabled
lsblk && df -h                                           # verify partition sizes
ssh -i ~/.ssh/luffy_key -p 2222 luffy@{your-ip}          # test luffy key auth
ssh -p 2222 zoro@{your-ip}                               # test zoro password auth
curl -o /dev/null -s -w "%{http_code}" http://{ip}/wp-config.php  # must return 403
cat /var/log/backup.log                                  # successful backup entry
ls /backup/                                              # .tar.gz files present
cat deep-in-system.sha1 | cat -e                         # ends with $ not ^M$
```

> 🚨 **Final rule:** You must be able to explain every single command and config line during the audit. Using anything you cannot explain is considered cheating by the project rules.