# Ansible

# 📘 Documentation — Installation & Configuration of Ansible for Splunk Universal Forwarder Deployment

## 🧩 1. Objective

Automate the installation, configuration, and management of the Splunk Universal Forwarder (UF) on Rocky Linux servers using Ansible.

---

## 🏗️ 2. Infrastructure Overview

### 🌐 Control Node (Ansible)

- Rocky Linux (PVE1)
- User: `admin`
- SSH keys stored in: `~/.ssh/`

### 🖥️ Managed Nodes

### **PVE1**

- 172.20.21.53 — NS1
- 172.20.21.67 — DHCP1
- 172.20.21.69 — TFTP1
- 172.20.21.89 — MX1

### **PVE2**

- 172.20.23.53 — NS2
- 172.20.23.67 — DHCP2
- 172.20.23.90 — CA2

---

## ⚙️ 3. Installing Ansible

### 🧲 Install required packages

```
sudo dnf install ansible python3-cryptography -y

```

### 📁 Create the Ansible workspace

```
mkdir ~/ansible-splunk
cd ~/ansible-splunk

```

---

## 🔑 4. SSH Access Setup

### 🗝️ Generate SSH key pair

```
ssh-keygen -t ed25519

```

### 🚀 Deploy your public key to each remote server

```
ssh-copy-id admin@<IP>

```

Repeat for all PVE1 & PVE2 machines.

---

## 📁 5. Ansible Inventory Files

### 📄 inventory_pve1.ini

```
[rocky_servers]
172.20.21.53 ansible_user=admin
172.20.21.67 ansible_user=admin
172.20.21.69 ansible_user=admin
172.20.21.89 ansible_user=admin

```

### 📄 inventory_pve2.ini

```
[rocky_servers_pve2]
172.20.23.53 ansible_user=admin
172.20.23.67 ansible_user=admin
172.20.23.90 ansible_user=admin

```

---

## 🛠️ 6. Working Playbook (install_splunk_uf.yml)

```yaml
---
- name: Install Splunk Universal Forwarder on Rocky Linux
  hosts: rocky_servers
  become: yes

  vars:
    splunk_user: "splunkfwd"
    splunk_indexer: "192.168.1.1:9997"
    splunk_rpm: "splunkforwarder-9.3.1-0b8d769cb912.x86_64.rpm"
    splunk_download_url: "<https://download.splunk.com/products/universalforwarder/releases/9.3.1/linux/>{{ splunk_rpm }}"

  tasks:

    - name: Install dependencies
      dnf:
        name:
          - wget
          - tar
        state: present

    - name: Create Splunk user
      user:
        name: "{{ splunk_user }}"
        shell: /bin/bash
        create_home: yes
        system: yes

    - name: Download Splunk UF RPM
      get_url:
        url: "{{ splunk_download_url }}"
        dest: "/tmp/{{ splunk_rpm }}"
        mode: '0644'

    - name: Install Splunk UF (disable GPG check)
      dnf:
        name: "/tmp/{{ splunk_rpm }}"
        state: present
        disable_gpg_check: true

    - name: Fix permissions on Splunk folder
      file:
        path: /opt/splunkforwarder
        owner: "{{ splunk_user }}"
        group: "{{ splunk_user }}"
        recurse: yes

    - name: Enable Splunk UF boot-start
      command: >
        /opt/splunkforwarder/bin/splunk enable boot-start
        -user {{ splunk_user }}
        --accept-license
        --answer-yes
        --no-prompt
      args:
        creates: /etc/systemd/system/SplunkForwarder.service

    - name: Add forward-server
      command: >
        /opt/splunkforwarder/bin/splunk add forward-server {{ splunk_indexer }}
      become_user: "{{ splunk_user }}"
      ignore_errors: yes

    - name: Configure inputs.conf
      copy:
        dest: /opt/splunkforwarder/etc/system/local/inputs.conf
        owner: "{{ splunk_user }}"
        group: "{{ splunk_user }}"
        mode: '0644'
        content: |
          [default]
          host = {{ ansible_default_ipv4.address }}

          [monitor:///var/log/messages]
          disabled = false

          [monitor:///var/log/secure]
          disabled = false

          [monitor:///var/log/maillog]
          disabled = false

          [monitor:///var/log/cron]
          disabled = false

          [monitor:///var/log/firewalld]
          disabled = false

          [monitor:///var/log/audit]
          disabled = false
          recursive = true

          [monitor:///var/log/kea]
          disabled = false
          recursive = true

    - name: Restart SplunkForwarder service
      systemd:
        name: SplunkForwarder
        state: restarted
        enabled: yes

```

---

## 🚀 7. Running the Playbook

### ▶️ Run on PVE1

```
ansible-playbook -i inventory_pve1.ini install_splunk_uf.yml --ask-become-pass

```

### ▶️ Run on PVE2

```
ansible-playbook -i inventory_pve2.ini install_splunk_uf.yml --ask-become-pass

```

---

## 🔍 8. Verification Steps

### 🟢 Check Universal Forwarder service

```
systemctl status SplunkForwarder

```

### 🟢 Confirm forwarding configuration

```
sudo -u splunkfwd /opt/splunkforwarder/bin/splunk list forward-server

```

### 🟢 Confirm logs are visible in Splunk

Search:

```
index=* host="<IP or hostname>"

```

### 🟢 Check connectivity to the indexer

```
sudo dnf install nmap-ncat -y
nc -zv 192.168.1.1 9997

```

---

## 📦 9. Important Notes

- SSH keys **must** be deployed before running Ansible.
- SELinux enforcing mode may block access to logs.
- `inputs.conf` determines exactly which files the UF sends.
- The UF may send either hostname or IP unless forced.

---

##
