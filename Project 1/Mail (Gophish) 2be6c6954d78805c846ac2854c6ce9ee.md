# Mail (Gophish)

# Installing and configuring Gophish + Postfix on Rocky Linux 9

---

## 1. Objective

Set up a phishing awareness platform by installing **Gophish** on Rocky Linux 9, and configure **Postfix** to send test emails.

The aim is to send campaigns, track clicks, manage landing pages and analyze user behavior.

---

## 2. Hardware used

- Rocky Linux 9 (server or VM)
- Root / sudo access
- Internet connection
- Gophish v0.12.1
- SMTP server: Postfix

---

## 3. Introduction

Gophish is an open-source framework for creating phishing campaigns in a secure environment.

It is installed in `/opt/gophish`, and a systemd service is configured to ensure automatic startup.

Postfix is used as a local SMTP server, enabling e-mails to be sent directly from Gophish.

---

# 4. Configuration

---

## 4.1 Installing and preparing Gophish

### ➤ Install unzip

```bash
sudo dnf install unzip -y
```

Installs the `unzip` tool needed to extract the Gophish archive.

### ➤ Download Gophish

```
curl -L -o gophish.zip https://github.com/gophish/gophish/releases/download/v0.12.1/gophish-v0.12.1-linux-64bit.zip
```

- `curl` downloads a file.
- `L` follows redirections.
- `o gophish.zip` names the downloaded file.

---

### ➤ Create installation directory

```
sudo mkdir /opt/gophish
cd /opt/gophish
```

Creates the folder for the application and moves to it.

---

### ➤ Extract archive

```
sudo unzip /home/admin/gophish.zip
```

Unzips the downloaded archive into the current directory.

---

### ➤ Create systemd service

```bash
sudo vi /etc/systemd/system/gophish.service
```

**File contents :**

```
[Unit]
Description=Gophish Phishing Framework
After=network.target

[Service]
WorkingDirectory=/opt/gophish
ExecStart=/opt/gophish/gophish
Restart=always
User=root

[Install]
WantedBy=multi-user.target
```

**Description :**

- `ExecStart` → start Gophish binary
- `Restart=always` → restarts in the event of a crash
- `WorkingDirectory` → Gophish folder
- Service launched after network (network.target)

---

### ➤ Set permissions

```
sudo chmod 755 /opt
sudo chmod -R 755 /opt/gophish
sudo chmod +x /opt/gophish/gophish
```

- `755` gives the necessary permissions.
- `+x` makes the executable usable.

---

### ➤ Open firewall ports

```
sudo firewall-cmd --add-port=3333/tcp --permanent
sudo firewall-cmd --add-port=80/tcp --permanent
sudo firewall-cmd --reload
```

- Port **3333**: admin interface
- Port **80**: landing pages

---

### ➤ Launch Gophish

```
./gophish
```

Test manual launch.

---

### ➤ Activate and check service

```bash
sudo systemctl start --now gophish
sudo systemctl status gophish
```

- Starts immediately
- Checks if service is running

---

## 4.2 Installing the Postfix server

### ➤ Install Postfix

```
sudo dnf install postfix -y
```

Installs the local mail server.

---

### ➤ Activate Postfix

```
sudo systemctl enable --now postfix
```

Activates and starts Postfix immediately.

---

# 5. Check

### ➤ Check Gophish

```bash
sudo systemctl status gophish
```

Expected result :

`Active: active (running)`

---

### ➤ Check open ports

```bash
sudo ss -tulnp | grep gophish
```

Should display :

- Port 3333 (admin)
- Port 80 (landing pages)

---

### ➤ Access Web interface

Navigate to :

```
https://IP_SERVEUR:3333
```

---

### ➤ Check Postfix

```
sudo systemctl status postfix
```

### ➤ Email logs

```
sudo tail -n 20 /var/log/maillog
```

---

# 6. Validation

The configuration is validated if:

✔ Gophish starts automatically

✔ admin interface works via HTTPS on port 3333

✔ HTTP port 80 responds for landing pages

✔ Postfix sends emails correctly

✔ A test campaign works (send → click → capture → reporting)