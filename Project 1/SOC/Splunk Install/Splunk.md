# Splunk Install

# Splunk Server 9.3.1 Installation Guide

## Objective

Set up a complete **Splunk infrastructure** using two Rocky Linux 9.6 virtual machines (Splunk-Server and Splunk-Web), including:

- Installation and configuration of **Splunk Enterprise 9.3.1**
- Access to Splunk Web GUI from a dedicated VM
- Opening and validating required ports (8000, 8089, 9997)
- Integration of a **Splunk Developer license**
- Preparing the environment for **Universal Forwarders** and log ingestion
- User management (`root/admin: formation`) for both VMs
- Comprehensive documentation for reproducibility

**Final objective:** A functional, secure Splunk platform ready to collect, index, and analyze data from Linux/Windows servers and internal infrastructure.

---

## Hardware Used

- **Hypervisor:** PVE3 (Proxmox VE)
- **Version:** Proxmox VE 8.4
- **Role:** Hosting two Rocky Linux VMs dedicated to Splunk

---

## Virtual Machine Configuration

### VM 1 - Splunk-Server

- **OS:** Rocky Linux 9.6
- **CPU:** 10 vCores
- **RAM:** 64 GiB
- **Storage:** 500 GB
- **Root User:** `root / formation`
- **Splunk Admin User:** `admin / formation`
- **Role:** Splunk Enterprise instance (indexing, ingestion, core services)

---

# Splunk Enterprise 9.3.1 Installation Steps

This guide includes **SELinux configuration**, **KV Store setup**, and **automatic startup via systemd**.

---

## 1. Rocky Minimal Prerequisites

Install required packages and enable the firewall:

```bash
sudo dnf install -y wget curl tar firewalld policycoreutils-python-utils
sudo systemctl enable --now firewalld

```

---

## 2. System Update

```bash
sudo dnf upgrade --refresh -y

```

---

## 3. Download Splunk

```bash
wget -O splunk-9.3.1.rpm "https://download.splunk.com/products/splunk/releases/9.3.1/linux/splunk-9.3.1-0b8d769cb912.x86_64.rpm"
```

![image.png](image.png)

---

## 4. Clean Previous Installations (If Necessary)

Remove anything that could interfere with a clean installation:

```bash
sudo rm -rf /opt/splunk
sudo userdel splunk 2>/dev/null
sudo groupdel splunk 2>/dev/null
sudo mkdir -p /opt
sudo chown root:root /opt
sudo restorecon -RF /opt
```

---

## 5. Temporarily Disable SELinux

SELinux prevents automatic creation of the `splunk` user and `/opt/splunk` directory:

```bash
sudo setenforce 0
getenforce  # should display Permissive
```

![image.png](image%201.png)

---

## 6. Install Splunk

```bash
sudo rpm -i splunk-9.3.1.rpm

```

---

## 7. Re-enable SELinux

```bash
sudo setenforce 1
getenforce  # should display Enforcing
```

![image.png](image%202.png)

---

## 8. First Splunk Start and Admin Account Creation

```bash
cd /opt/splunk/bin
sudo ./splunk start --accept-license
```

![image.png](image%203.png)

Create the admin account:

```
Username: admin
Password: formation
```

![image.png](image%204.png)

---

## 9. Enable Splunk Auto-Start via systemd

### Create the systemd service

```bash
sudo tee /etc/systemd/system/splunk.service > /dev/null << 'EOF'
[Unit]
Description=Splunk Enterprise
After=network.target

[Service]
Type=forking
User=splunk
Group=splunk
ExecStart=/opt/splunk/bin/splunk start --no-prompt --accept-license
ExecStop=/opt/splunk/bin/splunk stop
Restart=on-failure

[Install]
WantedBy=multi-user.target
EOF

```

### Enable and start the service

```bash
sudo systemctl daemon-reload
sudo systemctl enable splunk
sudo systemctl start splunk
sudo systemctl status splunk

```

---

## 10. Verify KV Store

```bash
/opt/splunk/bin/splunk show kvstore-status

```

- `status: ready` → KV Store is operational
- `replicationStatus: KV store captain` → OK

If necessary, fix permissions:

```bash
sudo chown -R splunk:splunk /opt/splunk/var/lib/splunk/kvstore

```

---

## 11. Open Firewalld Ports

```bash
sudo firewall-cmd --add-port=8000/tcp --permanent    # Splunk Web
sudo firewall-cmd --add-port=8089/tcp --permanent    # Management
sudo firewall-cmd --add-port=9997/tcp --permanent    # Forwarders
sudo firewall-cmd --reload

```

---

## 12. Access Splunk Web

From your browser:

```
http://YOUR_SPLUNK_VM_IP:8000

```

Login:

```
Username: admin
Password: formation

```

---

## 13. Install Developer License

### Web UI Method (Recommended)

1. Splunk Web → `Settings` → `Licensing` → `Add license`
2. Upload the `.lic` file

### CLI Method (If Web GUI is elsewhere)

1. Copy license to the VM:

```bash
scp license.lic root@SPLUNK-SERVER:/opt/splunk/etc/licenses/

```

1. Add the license:

```bash
sudo /opt/splunk/bin/splunk add licenses /opt/splunk/etc/licenses/license.lic
sudo /opt/splunk/bin/splunk restart
```

![image.png](image%205.png)

---

✅ **Outcome with this setup:**

- Splunk automatically starts at VM boot
- KV Store is operational
- SELinux is active for system security
- Developer license is applied correctly

---
