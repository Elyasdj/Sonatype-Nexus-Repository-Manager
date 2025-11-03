# 🚀 Sonatype Nexus Repository Manager

<div align="center">
  <img src="./public/nexus-architecture.svg" alt="Nexus Architecture" width="800"/>
  
  [![License](https://img.shields.io/badge/license-MIT-blue.svg)](LICENSE)
  [![Nexus Version](https://img.shields.io/badge/nexus-3.85.0--03-orange.svg)](https://www.sonatype.com/products/nexus-repository)
  [![Java](https://img.shields.io/badge/java-11+-red.svg)](https://openjdk.java.net/)
</div>

---

## 📋 Table of Contents

- [What is Sonatype Nexus?](#what-is-sonatype-nexus)
- [Use Cases](#use-cases)
- [Architecture Overview](#architecture-overview)
- [Quick Start](#quick-start)
- [Installation Guide](#installation-guide)
- [Configuration](#configuration)
- [Security & Hardening](#security--hardening)
- [Performance Tuning](#performance-tuning)
- [Monitoring](#monitoring)
- [Troubleshooting](#troubleshooting)
- [Contributing](#contributing)

---

## 🔍 What is Sonatype Nexus?

**Sonatype Nexus Repository Manager** is a universal artifact repository manager that stores and distributes software components. It acts as a central hub for managing binaries, build artifacts, and dependencies across your entire software development lifecycle.

**Key Features:**
- 📦 Universal repository support (Maven, npm, Docker, PyPI, apt, yum, etc.)
- 🔒 Enterprise-grade security and access control
- 🚄 Intelligent proxy and caching for external repositories
- 📊 Built-in monitoring and health checks
- 🔄 High availability and scalability

---

## 💡 Use Cases

<table>
<tr>
<td width="50%">

### 🏢 Enterprise Artifact Management
Centralize all development artifacts (JARs, WARs, containers) in one secure location with fine-grained access control.

### 🌐 Proxy External Repositories
Cache dependencies from Maven Central, npm registry, Docker Hub to reduce external bandwidth and improve build speed.

</td>
<td width="50%">

### 📦 Private Package Distribution
Host proprietary packages and libraries for internal distribution across development teams.

### 🔄 CI/CD Integration
Seamless integration with Jenkins, GitLab CI, GitHub Actions for automated artifact publishing and retrieval.

</td>
</tr>
</table>

---

## 🏗️ Architecture Overview

Our Nexus deployment follows a production-ready architecture:

```
┌─────────────┐
│   Internet  │
└──────┬──────┘
       │
┌──────▼──────────────────┐
│  Nginx Reverse Proxy    │  ← SSL/TLS Termination
│  (Port 443/8081)        │
└──────┬──────────────────┘
       │
┌──────▼──────────────────┐
│  Nexus Repository       │  ← Application Layer
│  (User: nexus)          │
│  Java 11+               │
└──────┬──────────────────┘
       │
┌──────▼──────────────────┐
│  File System Storage    │  ← Blob Stores
│  /opt/nexus/...         │
└─────────────────────────┘
```

---

## ⚡ Quick Start

```bash
# Clone the repository
git clone <YOUR_REPOSITORY_URL>
cd nexus-setup

# Run the installation script
sudo bash scripts/install-nexus.sh

# Start Nexus service
sudo systemctl start nexus

# Check status
sudo systemctl status nexus

# Access Web UI
# http://your-server-ip:8081
```

**Default Credentials:**
- Username: `admin`
- Password: Check `/opt/nexus/sonatype-work/nexus3/admin.password`

---

## 📥 Installation Guide

### Prerequisites
- RHEL/CentOS/Rocky Linux 8+ or Ubuntu 20.04+
- Minimum 4GB RAM (8GB+ recommended)
- Java 11 or higher
- 50GB+ disk space

### Step-by-Step Installation

Refer to the following configuration files in this repository:

1. **[Java Installation](docs/01-java-installation.md)**
2. **[Nexus Installation](docs/02-nexus-installation.md)**
3. **[Systemd Service Setup](docs/03-systemd-service.md)**
4. **[Nginx Reverse Proxy](docs/04-nginx-proxy.md)**

---

## ⚙️ Configuration

### Repository Types

| Type | Purpose | Example |
|------|---------|---------|
| **hosted** | Store your own artifacts | Internal libraries, Docker images |
| **proxy** | Cache external repos | Maven Central, npm registry |
| **group** | Combine multiple repos | Aggregate Maven repos |

### Create Local Repositories

#### For Debian/Ubuntu (apt)
See: `docs/apt-repository-setup.md`

#### For RHEL/Rocky (yum)
See: `docs/yum-repository-setup.md`

---

## 🔐 Security & Hardening

### SSL/TLS Configuration
```bash
# Generate self-signed certificate
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout /etc/nginx/ssl/nexus.key \
  -out /etc/nginx/ssl/nexus.crt
```

### Firewall Rules
```bash
# RHEL/Rocky
firewall-cmd --add-port=8081/tcp --permanent
firewall-cmd --reload

# Ubuntu
ufw allow 8081/tcp
ufw reload
```

### Access Control
- Enable anonymous access: **Disabled by default**
- Use LDAP/AD integration for enterprise auth
- Implement role-based access control (RBAC)

Configuration files: `docs/security-hardening.md`

---

## 🚀 Performance Tuning

### JVM Memory Settings
```properties
# /opt/nexus/bin/nexus.vmoptions
-Xms4G
-Xmx4G
-XX:MaxDirectMemorySize=4G
```

**Rule of Thumb:** Allocate 50-60% of total server RAM to Nexus

### Disk I/O Optimization
- **OS + Nexus binaries:** SSD/NVMe
- **sonatype-work directory:** Faster SSD/NVMe
- **Blob stores:** Large capacity SSD/SAN

### Network Tuning
See: `configs/sysctl.d/99-prod.conf`

### File Handle Limits
See: `configs/security/limits.conf`

Full tuning guide: `docs/performance-tuning.md`

---

## 📊 Monitoring

### Built-in Prometheus Metrics
Nexus provides native Prometheus integration at:
```
http://your-nexus:8081/service/metrics/prometheus
```

### Grafana Dashboard
Import dashboard ID: **16459** (Nexus Dashboard by KL)

Configuration: `docs/monitoring-setup.md`

---

## 🔧 Troubleshooting

### Common Issues

<details>
<summary><b>Nexus won't start</b></summary>

```bash
# Check logs
tail -f /opt/nexus/sonatype-work/nexus3/log/nexus.log

# Verify Java version
java -version

# Check disk space
df -h /opt/nexus
```
</details>

<details>
<summary><b>502 Bad Gateway (Nginx)</b></summary>

```bash
# Check Nginx logs
tail -f /var/log/nginx/error.log

# SELinux issues (RHEL)
setenforce 0  # Temporary
# Then properly configure SELinux policies

# Verify Nexus is listening
ss -tunlp | grep 8081
```
</details>

<details>
<summary><b>Out of Memory</b></summary>

Increase JVM heap size in `/opt/nexus/bin/nexus.vmoptions`
```properties
-Xms8G
-Xmx8G
```
</details>

---

## 📚 Additional Resources

- [Official Sonatype Documentation](https://help.sonatype.com/repomanager3)
- [Nexus Community](https://community.sonatype.com/)
- [Prometheus Metrics Guide](https://help.sonatype.com/en/prometheus.html)
- [Security Best Practices](https://help.sonatype.com/en/security-best-practices.html)



---

<div align="center">
  <sub>Create by Elyasdj</sub>
  <sub>Guide by Elyasdj</sub>
</div>
