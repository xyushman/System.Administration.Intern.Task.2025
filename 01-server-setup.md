# 01. Server Setup Guide

## Digital Ocean Droplet Creation

### Step 1: Create the Droplet
1. Logged into Digital Ocean control panel
2. Created → Droplets
![alt text](image-1.png)
3. Selected **Rocky Linux 10** as operating system
![alt text](image-2.png)
4. Chose **Basic Shared CPU** plan with:
   - 2GB RAM
   - 1 vCPU
   - 50GB SSD Disk
   - ($12/month)
5. Selected **SFO2** region for optimal performance
![alt text](image-3.png)
6. Enabled both **IPv4** and **IPv6** networking
7. Added SSH key for secure authentication
![alt text](image-4.png)
8. Set hostname: `xyushman`
9. Created droplet
![alt text](image.png)

### Step 2: Initial Server Hardening

Connected to the droplet as root user:
```bash
ssh root@167.172.205.191
```
![alt text](image-5.png)
Created a new user and granted administrative privileges:
```bash
# Create the user
adduser xyushman
passwd xyushman

# Add to sudo group
usermod -aG wheel xyushman

# Install text editor
dnf install nano

# Copy SSH keys for passwordless login
rsync --archive --chown=xyushman:xyushman ~/.ssh /home/xyushman
```

Disabled root SSH login for better security:
```bash
# Edit SSH configuration
nano /etc/ssh/sshd_config
# Changed PermitRootLogin yes to PermitRootLogin no

# Restart SSH service
systemctl restart sshd
```

Logged out and back in as the new user:
```bash
exit
ssh xyushman@167.172.205.191
```
![alt text](image-6.png)

### Step 3: Firewall Configuration

Configured the firewall to allow essential services:
```bash
# Switch to root user
sudo -i

# Install and enable firewalld
dnf install firewalld -y
systemctl enable --now firewalld
systemctl status firewalld

# Allow essential services
firewall-cmd --permanent --add-service=http
firewall-cmd --permanent --add-service=https
firewall-cmd --permanent --add-port=8080/tcp  # For Keycloak
firewall-cmd --permanent --add-service=ssh

# Apply the new rules
firewall-cmd --reload

# Verify firewall rules
firewall-cmd --list-all
```

![alt text](image-7.png)

### Step 4: System Updates & Core Components Installation

Updated system packages and installed essential components:
```bash
# Update all system packages
sudo dnf update -y

# Install EPEL and Remi repositories
sudo dnf install epel-release -y
sudo dnf install https://rpms.remirepo.net/enterprise/remi-release-10.rpm -y
sudo dnf module enable php:remi-8.3 -y

# Install Apache, PHP, MariaDB, Python, and other tools
sudo dnf install httpd php php-cli php-mysqlnd php-gd php-xml php-mbstring php-json php-fpm \
mariadb-server python3 python3-pip unzip wget -y

# Enable and start core services
sudo systemctl enable --now httpd
sudo systemctl enable --now php-fpm
sudo systemctl enable --now mariadb

# Secure MariaDB installation
sudo mysql_secure_installation
```
During MariaDB secure installation, used these options:
1. Enter current password for root: Press Enter (fresh installation)
2. Switch to unix_socket authentication: n
3. Change the root password: Y (set a strong password)
4. Remove anonymous users: y
5. Disallow root login remotely: y
6. Remove test database and access to it: y
7. Reload privilege tables: y

![alt text](image-13.png)
### Step 5: Java Installation

Installed Java for Keycloak:
```bash
# Install Java JDK
sudo dnf install java-21-openjdk-devel -y

# Verify Java installation
java -version
```

## Verification

Verified all services were running correctly:
```bash
# Check service status
systemctl status httpd
systemctl status php-fpm
systemctl status mariadb
systemctl status firewalld

# Check firewall rules
sudo firewall-cmd --list-all
```

## Server Information
- **IP Address**: 167.172.205.191
- **Hostname**: xyushman
- **OS**: Rocky Linux 10
- **Region**: SFO2
- **VPC**: default-sfo2

At this point, the basic server setup was complete with all essential services installed, configured, and secured. The server was ready for application deployment and Keycloak installation.

---

**Next Step**: [Keycloak Installation and Configuration](02-keycloak-configuration.md)