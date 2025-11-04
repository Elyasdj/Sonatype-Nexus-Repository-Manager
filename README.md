# 📦 Sonatype Nexus Repository Manager - Complete Setup Guide

A comprehensive production-ready guide for installing, configuring, and optimizing Nexus Repository Manager

[![Nexus](https://img.shields.io/badge/Nexus-3.85.0-blue?logo=sonatype)](https://www.sonatype.com/products/nexus-repository)
[![nginx](https://img.shields.io/badge/nginx-1.26.3-green.svg)](https://nginx.org/download/nginx-1.26.3.tar.gz)
[![Java](https://img.shields.io/badge/Java-21.0.9-orange?logo=oracle)](https://www.oracle.com/java/)

---

## 📑 Table of Contents

- [Introduction](#-introduction)
- [Architecture](#-architecture)
- [Prerequisites](#-prerequisites)
- [Installation & Configuration](#-installation--configuration)
  - [Install Java](#1-install-java)
  - [Install Nexus](#2-install-nexus)
  - [Create Systemd Service](#3-create-systemd-service)
  - [Configure Nginx](#4-configure-nginx)
  - [Security & Hardening](#5-security--hardening)
- [Repository Management](#-repository-management)
- [Monitoring](#-monitoring)
- [Performance Tuning](#-performance-tuning)
- [Troubleshooting](#-troubleshooting)

---

## 🔍 Introduction

### What is Sonatype Nexus?
A Repository Manager for storing, managing, and distributing software artifacts (packages, libraries, containers).

### Use Cases
- **Private Repository**: Store and manage organization's internal packages
- **Proxy Repository**: Cache dependencies from public repositories (Maven Central, npm, Docker Hub)
- **Group Repository**: Combine multiple repositories into a single endpoint
- **Artifact Management**: Manage versions and artifact lifecycle
- **Offline Environment**: Support for air-gapped and offline environments
- **Multi-Format Support**: Maven, npm, Docker, PyPI, NuGet, APT, YUM, and more

### Repository Types

| Type | Description |
|------|-------------|
| **Hosted** | Store organization's internal artifacts |
| **Proxy** | Cache artifacts from external sources |
| **Group** | Combine hosted and proxy repositories |

---

## 🏗️ Architecture

### Our Nexus Structure

<img width="1131" height="1081" alt="Untitled Diagram drawio" src="https://github.com/user-attachments/assets/6867a314-fda6-418a-9890-f8cf19e7c20e" />

---

## ⚙️ Prerequisites

- **OS**: RockyLinux10.0 or Ubuntu 24.04.3
- **RAM**: Minimum 4GB 
- **CPU**: Minimum 4 Cores

---

## 🚀 Installation & Configuration

### 1. Install Java

```bash
# Set hostname
sudo hostnamectl set-hostname Nexus_server

# Install Java
sudo yum install java -y 

# Verify version
java -version
```

---

### 2. Install Nexus

#### Create nexus User

```bash
# Create system user without home directory
useradd -M -d /opt/nexus -s /bin/bash -r nexus

# Grant sudo access (for administrative operations)
echo "nexus ALL=(ALL) NOPASSWD: ALL" > /etc/sudoers.d/nexus
```

#### Download and Install Nexus

```bash
# Create installation directory
mkdir /opt/nexus

# Download Nexus
wget https://download.sonatype.com/nexus/3/nexus-3.85.0-03-linux-x86_64.tar.gz

# Extract files
tar -xzf nexus-3.85.0-03-linux-x86_64.tar.gz -C /opt/nexus --strip-components=1

# Set file ownership
chown -R nexus:nexus /opt/nexus
```

#### Configure JVM and User

```bash
# JVM memory settings
vim /opt/nexus/bin/nexus.vmoptions
# Content in nexus.vmoptions

# Set run user
vim /opt/nexus/bin/nexus.rc
# Add: run_as_user="nexus"
```

#### Manual Startup (Testing)

```bash
# Start Nexus with nexus user
sudo -u nexus sh -c "cd /opt/nexus && ./bin/nexus start"

# View logs
tail -f /opt/nexus/sonatype-work/nexus3/log/nexus.log

# Always check port
ss -tunpla | grep 8081

# Always open firewall port
firewall-cmd --add-port=8081/tcp --permanent
firewall-cmd --reload
```

#### Temporary Access
`http://your-ip:8081`

#### Stop Nexus
```bash
/opt/nexus/bin/nexus stop
```

---

### 3. Create Systemd Service

Create systemd service for automatic Nexus management

```bash
# Create service file
vim /usr/lib/systemd/system/nexus.service
# Content in nexus.service

# Reload systemd
systemctl daemon-reload

# Enable and start service
systemctl enable --now nexus

# Always check status
systemctl status nexus
```

---

### 4. Configure Nginx

Use Nginx as Reverse Proxy with TLS/SSL

#### Install Nginx

```bash
# Install Nginx
dnf install nginx -y  
```

#### Configure Virtual Host

```bash
# Create configuration file
vim /etc/nginx/conf.d/nexus.conf
# Content in nexus.conf

# Test configuration
nginx -t

# Restart
systemctl restart nginx

# Enable and start service
systemctl enable --now nginx

# Always check status
systemctl status nginx.service
```

#### Get Initial Admin Password

```bash
cat /opt/nexus/sonatype-work/nexus3/admin.password
```

**Note**: This file is deleted after first login.

---

### 5. Security & Hardening

#### Create SSL/TLS Certificate

```bash
# Create directory
mkdir -p /etc/nginx/ssl
cd /etc/nginx/ssl

# Generate self-signed certificate
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout /etc/nginx/ssl/nexus.edj.com.key \
  -out /etc/nginx/ssl/nexus.edj.com.crt

# Set permissions
chmod 600 /etc/nginx/ssl/nexus.*
chown root:root /etc/nginx/ssl/nexus.*
```

**Note**: For production, use Let's Encrypt or a trusted CA.

---

## 📊 Repository Management

### Create Repository

#### APT Repository (Debian/Ubuntu)

```bash
# Check OS version
cat /etc/os-release

# Backup original sources
sudo mv /etc/apt/sources.list.d/ubuntu.sources /etc/apt/sources.list.d/ubuntu.sources.bak

# Create new sources
sudo vim /etc/apt/sources.list.d/ubuntu.sources
# Content in ubuntu.sources
```

**If you really must bypass GPG**

```bash
#Edit your source list
sudo vim /etc/apt/sources.list.d/ubuntu.sources

#Change
deb [trusted=yes] https://nexus.edj.com/repository/apt_security_ubuntu24/ noble-security main restricted universe multiverse

#Save and run
sudo apt update
```

**Path in Nexus UI:**
```
Settings > Repositories > Create Repository > apt (proxy)
```

**Difference between apt(hosted) and apt(proxy):**
- **apt(hosted)**: For storing your own packages
- **apt(proxy)**: For caching Ubuntu/Debian packages

#### YUM Repository (RedHat/CentOS)

```bash
# Check OS version
cat /etc/os-release

# Create new repo
sudo vim /etc/yum.repos.d/local.repo
# Content in local.repo
```

**Path in Nexus UI:**
```
Settings > Repositories > Create Repository > yum (proxy)
```

---

## 📈 Monitoring

### Prometheus Integration

**Important Note**: No nexus-exporter needed - Nexus has native Prometheus support.

#### Endpoint
`https://nexus.edj.com/service/rest/metrics/prometheus`

#### Grafana Dashboard
**Dashboard ID**: `16459` - Infra / Nexus (Nexus Dashboard by KL)

**Official Documentation**: [Sonatype Prometheus Docs](https://help.sonatype.com/en/prometheus.html)

---

## ⚡ Performance Tuning

### Nexus Optimization

#### 1. Change Machine ID

**Why?** Prevent conflicts in cloned environments

```bash
# Show current Machine ID
cat /etc/machine-id
# or
hostnamectl | grep "Machine ID"

# Generate new Machine ID
sudo rm -rfv /etc/machine-id
sudo systemd-machine-id-setup
```

#### 2. Memory Settings

**Note**: Nexus is RAM and Disk I/O intensive - 90% of performance issues originate here.

##### JVM Settings

```bash
vim /opt/nexus/bin/nexus.vmoptions
```

**Recommendations:**
- Minimum: 4GB RAM + 4 CPU Cores
- **Never allocate more than 50-60% of total server RAM to Nexus**

##### Disk Isolation

**Best Practice:**

| Disk | Content | Type |
|------|---------|------|
| **Disk 1** | OS + Nexus Binary | SSD/NVMe |
| **Disk 2** | `sonatype-work` Directory | SSD/NVMe (Faster) |
| **Disk 3** | `Blob Stores` Directory | SSD/SAN (High Capacity) |

---

### OS Tuning

#### 3. Increase File Handles

```bash
vim /etc/security/limits.conf
```

Add:
```
root    soft   nofile   65536
root    hard   nofile   65536
nexus   soft   nofile   65536
nexus   hard   nofile   65536
nginx   soft   nofile   65536
nginx   hard   nofile   65536
```

**Note**: Ensure nexus and nginx run with appropriate users.

#### 4. Network Tuning

```bash
vim /etc/sysctl.d/99-prod.conf
```

Add:
```bash
# Increase maximum number of open files system-wide
fs.file-max = 2097152

# Increase incoming connection queue (for Nginx)
net.core.somaxconn = 65536

# Fast reuse of sockets in TIME-WAIT mode
net.ipv4.tcp_tw_reuse = 1

# Increase local port range
net.ipv4.ip_local_port_range = 1024 65000
```

Apply changes:
```bash
sysctl -p
```

#### 5. Manage SELinux (RedHat)

**Default Mode**: Enforcing

```bash
# Disable temporarily
setenforce 0

# Re-enable after configuration
setenforce 1
```

**Note**: For production, define proper policies instead of disabling SELinux.

#### 6. Firewall Management

```bash
# Open required ports
firewall-cmd --add-port=8081/tcp --permanent  # Nexus
firewall-cmd --add-port=443/tcp --permanent   # Nginx HTTPS
firewall-cmd --reload

# Check rules
firewall-cmd --list-all
```

---

## 🔧 Troubleshooting

### Check Logs

```bash
# Nexus logs
tail -f /opt/nexus/sonatype-work/nexus3/log/nexus.log

# Nginx logs
tail -f /var/log/nginx/error.log
tail -f /var/log/nginx/access.log
```

### SELinux Issues (RedHat)

```bash
# Check SELinux status
sestatus

# Disable temporarily for testing
setenforce 0

# Re-enable
setenforce 1
```

### AppArmor Issues (Debian/Ubuntu)

```bash
# Edit Nginx profile
vim /etc/apparmor.d/usr.sbin.nginx
```

Add:
```
network inet stream,
network inet6 stream,
```

Apply changes:
```bash
sudo apparmor_parser -r /etc/apparmor.d/usr.sbin.nginx
```

### Test Access in Offline Environment

```bash
# Configure hosts
sudo vim /etc/hosts
```

Add:
```
192.168.56.155 nexus.edj.com
```

Test:
```bash
# RedHat
curl -Ik https://nexus.edj.com

# Debian
curl -Ik https://nexus.edj.com
```

### Common Issues

| Issue | Probable Cause | Solution |
|-------|----------------|----------|
| Port 8081 not accessible | Firewall or SELinux | Open port in firewall |
| Nginx 502 Bad Gateway | Nexus not ready | Wait for Nexus to fully start |
| HTTPS access fails | SSL certificate issue | Check ownership and permissions |

---

## ✅ Installation Verification

### Check Services

```bash
# Nexus
systemctl status nexus.service
ss -tunpla | grep 8081

# Nginx
systemctl status nginx.service
ss -tunpla | grep 443
```

### Test Endpoints

```bash
# Nexus (direct)
curl -I http://localhost:8081

# Nexus (via Nginx)
curl -Ik https://nexus.edj.com

# Metrics (for Prometheus)
curl -k https://nexus.edj.com/service/metrics/prometheus
```

### Access Web UI

```
URL: https://nexus.edj.com
Username: admin
Password: [read from admin.password file]
```

---

## 📄 Configuration Files

### Required Files

| File | Path | Purpose |
|------|------|---------|
| `nexus.vmoptions` | `/opt/nexus/bin/` | JVM settings |
| `nexus.rc` | `/opt/nexus/bin/` | Run user |
| `nexus.service` | `/usr/lib/systemd/system/` | Systemd unit |
| `nexus.conf` | `/etc/nginx/conf.d/` | Nginx virtual host |
| `ubuntu.sources` | `/etc/apt/sources.list.d/` | APT repository (Debian) |
| `local.repo` | `/etc/yum.repos.d/` | YUM repository (RedHat) |
| `99-prod.conf` | `/etc/sysctl.d/` | Kernel tuning |

**Note**: Configuration file templates are included in this repository.

---

## 💡 Pro Tips

### Systemd Service File Path

- **Recommended Path**: `/usr/lib/systemd/system/`
- **Why?**: Using `systemctl mask` on services in `/etc/systemd/system/` can cause issues

### Default Ports

| Service | Port | Protocol |
|---------|------|----------|
| Nexus Web UI | 8081 | HTTP |
| Nginx Proxy | 80/443 | HTTP/HTTPS |

### Blob Store Management

- Each repository can have a separate blob store
- Blob stores can be moved to separate disks
- Use File-Based blob stores (S3 for cloud)

---

## 🔐 Security

### Security Checklist

- ✅ Change default `admin` password
- ✅ Enable HTTPS with valid certificate
- ✅ Restrict network access (firewall)
- ✅ Enable authentication for anonymous access
- ✅ Configure RBAC (Role-Based Access Control)
- ✅ Enable audit logging
- ✅ Regular backups of blob stores and database
- ✅ Keep Nexus updated

### Backup

```bash
# Important directories
/opt/nexus/sonatype-work/   # Nexus data
/opt/nexus/bin/             # Configuration files
```

**Recommendation**: Use application-level backup:
```
Admin Panel > System > Tasks > Create Task > Admin - Backup H2 Database
```

---

## 📚 Useful Resources

- [Sonatype Nexus Documentation](https://help.sonatype.com/repomanager3)
- [Best Practices Guide](https://help.sonatype.com/repomanager3/planning-your-implementation)
- [Prometheus Metrics](https://help.sonatype.com/en/prometheus.html)

---

## 📞 Support

For issues and questions:
- Official Documentation: https://help.sonatype.com
- Community Forums: https://community.sonatype.com

---

<div align="center">
  <sub>Created by Elyasdj</sub>
</div>
<div align="center">
  <sub>Guided by Mahdi Sardari</sub>
</div>
