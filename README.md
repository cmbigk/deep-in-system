# Deep-in-System

## Overview

This project involves setting up a complete Linux server environment on a Virtual Machine (Ubuntu Server), including a LAMP stack (Linux, Apache, MySQL, PHP), WordPress CMS, an FTP server, and an automated backup system using cron jobs.

---

## Table of Contents
1. [Virtual Machine Setup](#1-virtual-machine-setup)
2. [System Updates](#2-system-updates)
3. [Apache Web Server](#3-apache-web-server)
4. [MySQL Database](#4-mysql-database)
5. [PHP Installation](#5-php-installation)
6. [WordPress Installation](#6-wordpress-installation)
7. [FTP Server](#7-ftp-server)
8. [Automated Backup](#8-automated-backup)
9. [Key Concepts](#9-key-concepts)

---

## 1. Virtual Machine Setup

A Virtual Machine (VM) is a software-based computer that runs an operating system inside another operating system. We used **VirtualBox** to create an Ubuntu Server VM.

**Why:** VMs allow us to run a server environment safely without affecting the host machine. They are widely used in DevOps for testing and deployment.

**Steps:**
- Downloaded Ubuntu Server ISO
- Created a new VM in VirtualBox with 2GB RAM, 20GB storage
- Configured network adapter to **Bridged Adapter** so the VM gets its own IP on the network
- Installed Ubuntu Server following the installation wizard
- Created a user during setup (e.g., `chmyint`)

---

## 2. System Updates

Before installing anything, we updated the system package list and upgraded existing packages.

**Why:** Ensures we have the latest security patches and compatible package versions.

```bash
sudo apt update && sudo apt upgrade -y
```

---

## 3. Apache Web Server

Apache is an open-source HTTP web server. It serves web pages to users who visit our server's IP address.

**Why:** Apache is required to host the WordPress website and serve it over HTTP.

### Installation

```bash
sudo apt install apache2 -y
```

### Enable and start Apache

```bash
sudo systemctl enable apache2
sudo systemctl start apache2
sudo systemctl status apache2
```

### Verify

Open a browser and visit `http://YOUR_SERVER_IP` — you should see the Apache default page.

---

## 4. MySQL Database

MySQL is a relational database management system. WordPress stores all its content (posts, users, settings) in a MySQL database.

**Why:** WordPress requires a database to function.

### Installation

```bash
sudo apt install mysql-server -y
```

### Secure MySQL

```bash
sudo mysql_secure_installation
```

Choices made:
- Remove anonymous users: Yes
- Disallow root login remotely: Yes
- Remove test database: Yes
- Reload privilege tables: Yes

### Create WordPress Database and User

```bash
sudo mysql
```

```

---

## 5. PHP Installation

PHP is a server-side scripting language. WordPress is built with PHP, so Apache needs PHP to process WordPress files.

**Why:** Without PHP, Apache cannot run WordPress — it would just display raw .php files as text.

### Installation

```bash
sudo apt install php libapache2-mod-php php-mysql php-curl php-gd php-mbstring php-xml php-xmlrpc php-soap php-intl php-zip -y
```

### Verify

```bash
php -v
```

---

## 6. WordPress Installation

WordPress is a free, open-source Content Management System (CMS) used to build websites and blogs.

**Why:** The project requires hosting a WordPress site on our server.

### Download WordPress

```bash
cd /tmp
wget https://wordpress.org/latest.tar.gz
tar -xzf latest.tar.gz
```

### Move files to Apache web root

```bash
sudo mv /tmp/wordpress/* /var/www/html/
sudo rm -rf /tmp/wordpress
sudo rm /var/www/html/index.html
```

### Set correct permissions

```bash
sudo chown -R www-data:www-data /var/www/html/
sudo chmod -R 755 /var/www/html/
```

### Configure WordPress

```bash
sudo cp /var/www/html/wp-config-sample.php /var/www/html/wp-config.php
sudo nano /var/www/html/wp-config.php
```


### Complete setup in browser

Visit `http://YOUR_SERVER_IP` and follow the WordPress installation wizard:
- Site Title: your choice
- Username: admin
- Password: generated strong password (save it!)
- Email: your email

---

## 7. FTP Server

FTP (File Transfer Protocol) is a standard protocol used to transfer files between a client and a server over a network.

**Why:** The project requires an FTP server so the nami user can securely download backup files from the `/backup` directory.

### Installation

```bash
sudo apt install vsftpd -y
```

### Backup default config

```bash
sudo cp /etc/vsftpd.conf /etc/vsftpd.conf.bak
```

### Configure vsftpd

```bash
sudo bash -c 'cat > /etc/vsftpd.conf << EOF
listen=NO
listen_ipv6=YES
anonymous_enable=NO
local_enable=YES
write_enable=NO
local_umask=022
dirmessage_enable=YES
use_localtime=YES
xferlog_enable=YES
connect_from_port_20=YES
chroot_local_user=YES
allow_writeable_chroot=YES
secure_chroot_dir=/var/run/vsftpd/empty
pam_service_name=vsftpd
rsa_cert_file=/etc/ssl/certs/ssl-cert-snakeoil.pem
rsa_private_key_file=/etc/ssl/private/ssl-cert-snakeoil.key
ssl_enable=NO
EOF'
```

### Create FTP user nami

```bash
sudo adduser nami
sudo mkdir -p /backup
sudo chown root:root /backup
sudo chmod 755 /backup
sudo usermod -d /backup nami
```

### Restart and enable vsftpd

```bash
sudo systemctl restart vsftpd
sudo systemctl enable vsftpd
```

### Test FTP connection

```bash
ftp localhost
# Login as: nami
# Run: ls (should show /backup contents)
# Run: get <filename> (should download successfully)
```

---

## 8. Automated Backup

A cron job is a scheduled task in Linux that runs automatically at specified times. We use it to back up the WordPress database daily.

**Why:** Backups protect against data loss from hardware failure, human error, virus attacks, or disasters. Automated backups ensure consistency without manual intervention.

### Create /backup directory

```bash
sudo mkdir -p /backup
```

### Create backup script

```bash
sudo nano /usr/local/bin/backup.sh
```

Content:

```bash
#!/bin/bash
DATE=$(date +%Y-%m-%d_%H-%M-%S)
BACKUP_DIR=/backup

mysqldump -u wpuser -pStrongPass123 wordpress | gzip > $BACKUP_DIR/wordpress-db-$DATE.tar.gz

chmod 644 $BACKUP_DIR/wordpress-db-$DATE.tar.gz

echo "wordpress backup created!, date: $DATE" >> /var/log/backup.log
```

### Make it executable

```bash
sudo chmod +x /usr/local/bin/backup.sh
```

### Schedule with cron (every day at 00:00)

```bash
sudo crontab -e
```

Add this line:

```text
0 0 * * * /usr/local/bin/backup.sh
```

### Test manually

```bash
sudo bash /usr/local/bin/backup.sh
ls /backup
cat /var/log/backup.log
```

Expected output in log:

```text
wordpress backup created!, date: 2026-03-09_08-10-56
```

---

## 9. Key Concepts

### What is a Cron Job?

A cron job is a time-based task scheduler in Unix/Linux systems. It allows you to run scripts or commands automatically at fixed intervals (e.g., every minute, hourly, daily). The cron schedule format is:

```text
* * * * * command
│ │ │ │ │
│ │ │ │ └── Day of week (0-7)
│ │ │ └──── Month (1-12)
│ │ └────── Day of month (1-31)
│ └──────── Hour (0-23)
└────────── Minute (0-59)
```

Example: `0 0 * * *` runs every day at midnight.

### Why is Backup Important?

- **Hardware failure:** Disks can fail unexpectedly
- **Human error:** Files can be accidentally deleted
- **Virus/ransomware attacks:** Malware can corrupt or encrypt data
- **Power failures:** Unexpected shutdowns can corrupt databases
- **Disaster recovery:** Backups allow you to restore service quickly

### What is FTP?

FTP (File Transfer Protocol) is a standard network protocol for transferring files between a client and server. vsftpd (Very Secure FTP Daemon) is a lightweight, secure FTP server for Linux.

### What is a LAMP Stack?

| Component | Role |
|-----------|------|
| Linux | Operating System |
| Apache | Web Server |
| MySQL | Database |
| PHP | Server-side scripting |

### What is WordPress?

WordPress is an open-source CMS (Content Management System) that powers over 40% of websites worldwide. It stores content in MySQL and uses PHP to dynamically generate web pages served by Apache.