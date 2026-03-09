# Deep-in-System

## Overview

This project demonstrates practical Linux system administration skills through the setup and configuration of a production-ready Ubuntu Server environment. It covers essential administrative tasks including custom disk partitioning, network configuration, security hardening, user management, service deployment, and automated backup implementation.

## System Specifications

- **OS:** Ubuntu Server LTS
- **Disk Size:** 30GB with custom partitioning:
  - `swap`: 4GB
  - `/`: 15GB
  - `/home`: 5GB
  - `/backup`: 6GB
- **Hostname:** `chmyint-host`
- **Primary User:** `chmyint`

---

## Table of Contents

1. [Virtual Machine Setup](#virtual-machine-setup)
2. [Network Configuration](#network-configuration)
3. [Security Configuration](#security-configuration)
4. [User Management](#user-management)
5. [FTP Server](#ftp-server)
6. [Database Server](#database-server)
7. [Web Server & WordPress](#web-server--wordpress)
8. [Automated Backup System](#automated-backup-system)
9. [Key Concepts Learned](#key-concepts-learned)

---

## Virtual Machine Setup

Installed Ubuntu Server LTS with custom disk partitioning during installation. Used manual partitioning to create dedicated partitions for system, home directories, swap space, and backups.

**Key Commands:**
```bash
# Update system packages
sudo apt update && sudo apt upgrade -y

# Verify partition layout
lsblk
df -h
```

**Concept:** Disk partitioning separates data logically, improving organization, security, and backup management. The `/backup` partition isolates backup data from system files.

---

## Network Configuration

Configured static IP addressing to ensure consistent network accessibility. Disabled DHCP to prevent dynamic IP changes.

**Configuration File:** `/etc/netplan/00-installer-config.yaml`

```yaml
network:
  version: 2
  ethernets:
    enp0s3:
      addresses:
        - 192.168.1.100/24
      routes:
        - to: default
          via: 192.168.1.1
      nameservers:
        addresses:
          - 8.8.8.8
          - 8.8.4.4
```

**Apply Configuration:**
```bash
sudo netplan apply
ip addr show
ping -c 5 google.com
```

**Concept:** Static IP configuration ensures the server maintains a consistent address, essential for services like SSH, web hosting, and FTP.

---

## Security Configuration

Implemented multiple security layers following best practices for production servers.

### SSH Hardening

Modified SSH configuration to enhance security:

**File:** `/etc/ssh/sshd_config`

```bash
# Key modifications
Port 2222                        # Changed from default port 22
PermitRootLogin no               # Disabled root login
PasswordAuthentication yes       # Enabled for specific users
PubkeyAuthentication yes         # Enabled key-based auth
```

**Apply Changes:**
```bash
sudo systemctl restart sshd
sudo systemctl status sshd
```

### Firewall Configuration

Configured UFW (Uncomplicated Firewall) to control incoming/outgoing traffic:

```bash
# Enable firewall
sudo ufw enable

# Configure rules
sudo ufw allow 2222/tcp          # SSH
sudo ufw allow 80/tcp            # HTTP
sudo ufw allow 21/tcp            # FTP
sudo ufw allow 20/tcp            # FTP data
sudo ufw status verbose
```

**Concept:** The principle of least privilege - only open necessary ports. Custom SSH port reduces automated attack attempts.

---

## User Management

Created users with different authentication methods and permission levels.

### User: luffy (Admin with SSH Key)

```bash
# Create user
sudo adduser luffy
sudo usermod -aG sudo luffy

# Set up SSH key authentication
sudo mkdir -p /home/luffy/.ssh
sudo nano /home/luffy/.ssh/authorized_keys
# Paste public key here

# Set permissions
sudo chown -R luffy:luffy /home/luffy/.ssh
sudo chmod 700 /home/luffy/.ssh
sudo chmod 600 /home/luffy/.ssh/authorized_keys
```

### User: zoro (Standard user with password)

```bash
sudo adduser zoro
# Set password during creation
# Do NOT add to sudo group
```

### User: nami (FTP-only user)

```bash
sudo adduser nami
# Configured FTP access in FTP section
```

**Concept:** SSH key authentication is more secure than passwords - eliminates brute force attacks. Sudo access should be granted selectively based on responsibility.

---

## FTP Server

Deployed vsftpd (Very Secure FTP Daemon) with restricted user access.

### Installation & Configuration

```bash
sudo apt install vsftpd -y
sudo cp /etc/vsftpd.conf /etc/vsftpd.conf.backup
```

**Configuration:** `/etc/vsftpd.conf`

```bash
anonymous_enable=NO              # Disable anonymous access
local_enable=YES                 # Enable local users
write_enable=NO                  # Read-only access
chroot_local_user=YES            # Restrict to home directory
allow_writeable_chroot=YES
local_umask=022
```

### Configure nami User

```bash
# Set home directory to /backup
sudo usermod -d /backup nami
sudo chown root:root /backup
sudo chmod 755 /backup

# Restart service
sudo systemctl restart vsftpd
sudo systemctl enable vsftpd
```

**Testing:**
```bash
ftp localhost 21
# Login as nami
# Test: ls, get <file>
```

**Concept:** FTP chroot jails restrict users to specific directories, preventing unauthorized file system access. Read-only permissions prevent accidental or malicious modifications.

---

## Database Server

Set up MySQL server with security hardening and dedicated WordPress database.

### Installation & Security

```bash
sudo apt install mysql-server -y
sudo mysql_secure_installation
```

**Security Choices:**
- Remove anonymous users: ✓
- Disallow root remote login: ✓
- Remove test database: ✓
- Reload privilege tables: ✓

### Database Configuration

```bash
sudo nano /etc/mysql/mysql.conf.d/mysqld.cnf
# Verify: bind-address = 127.0.0.1 (local only)
```

### Create WordPress Database

```bash
sudo mysql
```

```sql
CREATE DATABASE wordpress;
CREATE USER 'wpuser'@'localhost' IDENTIFIED BY 'StrongPass123';
GRANT ALL PRIVILEGES ON wordpress.* TO 'wpuser'@'localhost';
GRANT PROCESS ON *.* TO 'wpuser'@'localhost';
FLUSH PRIVILEGES;
EXIT;
```

**Concept:** Principle of least privilege - the WordPress user only has access to the WordPress database, not all MySQL databases. Binding to localhost prevents external access.

---

## Web Server & WordPress

Deployed LAMP stack (Linux, Apache, MySQL, PHP) to host WordPress CMS.

### Apache Installation

```bash
sudo apt install apache2 -y
sudo systemctl enable apache2
sudo systemctl start apache2
```

### PHP Installation

```bash
sudo apt install php libapache2-mod-php php-mysql php-curl php-gd \
  php-mbstring php-xml php-xmlrpc php-soap php-intl php-zip -y
php -v
```

### WordPress Deployment

```bash
# Download and extract
cd /tmp
wget https://wordpress.org/latest.tar.gz
tar -xzf latest.tar.gz

# Deploy to web root
sudo mv wordpress/* /var/www/html/
sudo rm /var/www/html/index.html

# Set ownership and permissions
sudo chown -R www-data:www-data /var/www/html/
sudo chmod -R 755 /var/www/html/
```

### WordPress Configuration

```bash
sudo cp /var/www/html/wp-config-sample.php /var/www/html/wp-config.php
sudo nano /var/www/html/wp-config.php
```

```php
define( 'DB_NAME', 'wordpress' );
define( 'DB_USER', 'wpuser' );
define( 'DB_PASSWORD', 'StrongPass123' );
define( 'DB_HOST', 'localhost' );
```

**Secure configuration file:**
```bash
sudo chmod 640 /var/www/html/wp-config.php
sudo chown www-data:www-data /var/www/html/wp-config.php
```

**Access:** Navigate to `http://<server-ip>` to complete WordPress installation.

**Concept:** The LAMP stack works together - Linux (OS), Apache (serves HTTP), MySQL (stores data), PHP (processes logic). WordPress leverages this stack to deliver dynamic web content.

---

## Automated Backup System

Implemented cron-based automated daily backups of the WordPress database.

### Backup Script

**File:** `/usr/local/bin/backup.sh`

```bash
#!/bin/bash
DATE=$(date +%Y-%m-%d_%H-%M-%S)
BACKUP_DIR=/backup

# Create compressed database backup
mysqldump -u wpuser -pStrongPass123 wordpress | gzip > $BACKUP_DIR/wordpress-db-$DATE.tar.gz

# Set permissions for FTP access
chmod 644 $BACKUP_DIR/wordpress-db-$DATE.tar.gz

# Log success
echo "wordpress backup created!, date: $DATE" >> /var/log/backup.log
```

**Make executable:**
```bash
sudo chmod +x /usr/local/bin/backup.sh
```

### Cron Job Configuration

```bash
sudo crontab -e
# Add: 0 0 * * * /usr/local/bin/backup.sh
```

**Cron Schedule Format:**
```
* * * * * command
│ │ │ │ │
│ │ │ │ └─── Day of week (0-7)
│ │ │ └───── Month (1-12)
│ │ └─────── Day of month (1-31)
│ └───────── Hour (0-23)
└─────────── Minute (0-59)
```

**Testing:**
```bash
sudo bash /usr/local/bin/backup.sh
ls -lh /backup
cat /var/log/backup.log
```

**Concept:** Automated backups are critical for disaster recovery. Cron jobs eliminate human error in backup scheduling. Daily backups at midnight minimize impact on system performance during peak hours.

---

## Key Concepts Learned

### System Administration

- **Disk Partitioning:** Logical separation of data for better organization, security, and management
- **Static vs DHCP:** Static IPs provide consistent addressing for services; DHCP is suitable for client devices
- **Package Management:** `apt` handles software installation, updates, and dependencies on Debian-based systems

### Security Best Practices

- **Principle of Least Privilege:** Grant minimum necessary permissions
- **Defense in Depth:** Multiple security layers (firewall, SSH hardening, user restrictions)
- **Port Security:** Non-standard ports reduce automated attacks
- **Key-Based Authentication:** More secure than passwords; eliminates brute force risks

### Service Management

- **systemctl:** Modern Linux service management (start, stop, enable, status)
- **File Permissions:** Understanding owner/group/other permissions (rwx) and umask
- **Chroot Jails:** Restrict user access to specific directories

### Backup Strategy

- **3-2-1 Rule:** 3 copies, 2 different media, 1 offsite (this project: 2 copies)
- **Automation:** Cron eliminates manual backup errors
- **Compression:** gzip reduces storage requirements
- **Logging:** Tracking backup success/failure for audit purposes

### Web Stack Architecture

- **LAMP Stack Component Interaction:**
  - Apache receives HTTP requests
  - PHP processes application logic  
  - MySQL stores/retrieves data
  - Apache returns HTTP response
- **CMS Architecture:** WordPress separates content (database) from presentation (files)
- **Web Server Permissions:** `www-data` user owns web files for Apache access

---

## Conclusion

This project provided hands-on experience in essential Linux system administration tasks. Key achievements include setting up a secure, functional web server environment with proper network configuration, user management, service deployment, and automated disaster recovery mechanisms. The skills learned are directly applicable to managing production Linux servers in real-world scenarios.