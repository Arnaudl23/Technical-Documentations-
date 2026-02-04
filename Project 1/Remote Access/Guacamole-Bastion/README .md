# Bastion (Guacamole)

# Guacamole installation

## Objective

This document describes the complete installation and advanced configuration of an **Apache Guacamole** server on **Rocky Linux 9**, including :

- compilation of **guacamole-server** with **libssh2**,
- deployment of the web client via **Tomcat**,
- **MariaDB** integration via the JDBC authentication module,
- implementation of a **Rocky Linux Jump Host** as SSH/RDP bastion.

This architecture makes it possible to centralize remote access to equipment (routers, MLS, firewalls, servers) in a **secure**, **standardized** and **scalable** way.

---

# 1. Preparation

## 1.1 Updating the machine

```bash
sudo dnf update -y
```

## 1.2 Install dependencies required for compilation

```bash
sudo dnf install -y gcc make curl
sudo dnf install -y libtool freerdp-devel cairo-devel libjpeg-turbo-devel
sudo dnf install -y openssl-devel uuid-devel
sudo dnf install -y cairo-devel freerdp-devel libjpeg-turbo-devel libpng-devel uuid-devel pango-devel libssh2-devel libtelnet-devel libvncserver-devel libwebp-devel libvorbis-devel pulseaudio-libs-devel openssl-devel libtool
```

---

# 2. Installing and compiling libssh2 from source

## 2.1 Downloading sources

```
cd /usr/local/src
sudo curl -O https://www.libssh2.org/download/libssh2-1.11.0.tar.gz
sudo tar xzf libssh2-1.11.0.tar.gz
cd libssh2-1.11.0
```

## 2.2 Compile libssh2

```bash
sudo ./configure --prefix=/usr/local
sudo make
sudo make install
```

## 2.3 Update libraries

```bash
sudo ldconfig
```

---

# 3. Installing and compiling Guacamole-server

## 3.1 Download sources

```
cd /usr/local/src
sudo curl -O https://apache.org/dyn/closer.lua/guacamole/1.5.3/source/guacamole-server-1.5.3.tar.gz
sudo tar xzf guacamole-server-1.5.3.tar.gz
cd guacamole-server-1.5.3
```

## 3.2 Compile Guacamole with compiled libssh2

```bash
sudo PKG_CONFIG_PATH="/usr/local/lib/pkgconfig" ./configure --with-ssh
sudo make
sudo make install
sudo ldconfig
```

## 3.3 Activate the `guacd` service

```bash
sudo systemctl enable --now guacd
```

### Create configuration file `/etc/guacamole/guacd.conf`

```
sudo vi /etc/guacamole/guacd.conf
```

Contents :

```
[server]
bind_host = 172.20.4.1
bind_port = 4822
```

Restart `guacd`:

```
sudo systemctl restart guacd
sudo systemctl enable guacd
```

---

# 4. Installing Tomcat and Guacamole Client

## 4.1 Installing Tomcat

```bash
sudo dnf install -y tomcat tomcat-webapps tomcat-admin-webapps
sudo systemctl enable --now tomcat
```

Permissions :

```bash
sudo chown tomcat:tomcat /etc/guacamole/guacamole.properties
sudo chmod 600 /etc/guacamole/guacamole.properties
```

## 4.2 Deploy `guacamole.war`

```
sudo cp guacamole.war /var/lib/tomcat/webapps/
sudo ln -s /etc/guacamole/ /usr/share/tomcat/.guacamole
```

---

# 5. MySQL configuration for Guacamole

## 5.1 MariaDB installation

```
sudo dnf install -y mariadb-server
sudo systemctl enable --now mariadb
```

Initialization :

```bash
sudo mysql_secure_installation
```

### Recommended parameters :

| Question | Answer | Explanation |
| --- | --- | --- |
| Enter current password | Enter | No password defined |
| unix_socket authentication ? | N | Requires real SQL password |
| Set root password ? | Y | Set strong password |
| Remove anonymous users ? | Y | Remove anonymous accounts |
| Disallow remote root login ? | Y | Disable remote root |
| Remove test database ? | Y | Clean |
| Reload privileges ? | Y | Apply |

## 5.2 Database and user creation

```
mysql -u root -p
CREATE DATABASE guacamole_db;
CREATE USER 'guacadmin'@'localhost' IDENTIFIED BY 'formation';
GRANT SELECT,INSERT,UPDATE,DELETE ON guacamole_db.* TO 'guacadmin'@'localhost';
FLUSH PRIVILEGES;
```

## 5.3 Import SQL schema

```bash
cat /etc/guacamole/schema/*.sql | mysql -uguacadmin -pformation guacamole_db
```

---

# 6. Configuring Guacamole

## 6.1 Modifying `guacamole.properties`

File: `/etc/guacamole/guacamole.properties`

```
mysql-hostname: localhost
mysql-database: guacamole_db
mysql-username: guacadmin
mysql-password: formation

guacd-hostname: localhost
guacd-port: 4822
```

Restart Tomcat :

```
sudo systemctl restart tomcat
```

---

# 7. JDBC authentication module

## 7.1 Download JDBC module

```
cd /usr/local/src
sudo curl -O https://dlcdn.apache.org/guacamole/1.5.3/binary/guacamole-auth-jdbc-1.5.3.tar.gz
sudo tar xzf guacamole-auth-jdbc-1.5.3.tar.gz
```

Install :

```
sudo mkdir -p /etc/guacamole/extensions
sudo mkdir -p /etc/guacamole/lib

sudo cp guacamole-auth-jdbc-1.5.3/mysql/guacamole-auth-jdbc-mysql-1.5.3.jar \
/etc/guacamole/extensions/
```

## 7.2 Install MySQL Connector/J

```
cd /usr/local/src
sudo curl -O https://dev.mysql.com/get/Downloads/Connector-J/mysql-connector-j-8.4.0.tar.gz
sudo tar xzf mysql-connector-j-8.4.0.tar.gz
cd mysql-connector-j-8.4.0
sudo cp mysql-connector-j-8.4.0.jar /etc/guacamole/lib/
```

Restart Tomcat :

```
sudo systemctl restart tomcat
```

---

# 8. Adding additional devices

## 8.1 Via Web interface

*(Standard method in Guacamole GUI*)

---

# 9. Jump Server Rocky Linux (SSH/RDP Proxy)

## 9.1 User creation

### 9.1.1 Guacamole user

In the web interface :

- Settings → Users
- Username: `mathieu`
- Password: `mathieu1`
- Permissions as required

### 9.1.2 Linux user on Rocky

```
sudo adduser jump
sudo passwd formation
```

Optional: avoid SSH key confirmation

```
sudo su - jump
ssh -o StrictHostKeyChecking=no 172.20.4.1
```

---

## 9.2 Configuring Guacamole to use Jump Host

Connection ID = 4 :

```
mysql -uguacadmin -pformation guacamole_db
```

Delete old settings :

```
DELETE FROM guacamole_connection_parameter WHERE connection_id = 4;
```

Add ProxyCommand :

```
INSERT INTO guacamole_connection_parameter (connection_id, parameter_name, parameter_value) VALUES
(4, 'hostname', '172.20.1.254'),
(4, 'port', '22'),
(4, 'username', 'mathieu'),
(4, 'password', 'mathieu1'),
(4, 'proxy-command', 'ssh -o StrictHostKeyChecking=no jump@172.20.4.1 -W %h:%p');
```

## 9.3 Adding devices (examples)

### SA1

```
DELETE FROM guacamole_connection_parameter WHERE connection_id=5;
INSERT INTO guacamole_connection_parameter VALUES
(5,'hostname','172.20.1.1'),
(5,'port','22'),
(5,'username','mathieu'),
(5,'password','mathieu1'),
(5,'proxy-command','ssh -o StrictHostKeyChecking=no jump@172.20.4.1 -W %h:%p');
```

*(Same structure for MLS2, SA2, R1, BBA1, BBA2, R3...)*

---

## Show available IDs

```bash
mysql -uguacadmin -pformation guacamole_db \
  -e "SELECT connection_id, connection_name FROM guacamole_connection;"
```

---

# Hardening

## SELinux

```
sudo setsebool -P tomcat_can_network_connect_db 1
sudo setsebool -P domain_can_mmap_files 1
sudo restorecon -Rv /var/lib/tomcat/webapps/
sudo restorecon -Rv /etc/guacamole/
```

## Secure Mount Configuration

Add to `/etc/fstab`:

```
tmpfs /tmp     tmpfs defaults,rw,nosuid,nodev,noexec,relatime,size=2G 0 0
tmpfs /dev/shm tmpfs defaults,rw,nosuid,nodev,noexec,relatime,size=2G 0 0
```

Then :

```
sudo systemctl daemon-reload
sudo mount /tmp
sudo mount /dev/shm
findmnt | grep -E "/tmp|/dev/shm"
```

### Validation

```
echo -e '#!/bin/bash\necho TEST_TMP_OK' > /tmp/t.sh
chmod +x /tmp/t.sh
/tmp/t.sh  # Permission denied
```

```
echo -e '#!/bin/bash\necho TEST_SHM_OK' > /dev/shm/t2.sh
chmod +x /dev/shm/t2.sh
/dev/shm/t2.sh  # Permission denied
```

## Disable unnecessary filesystems

Create :

```
sudo vi /etc/modprobe.d/filesystems-blacklist.conf
```

Add to :

```
install cramfs /bin/false
install freevxfs /bin/false
install hfs /bin/false
install hfsplus /bin/false
install jffs2 /bin/false
install udf /bin/false

blacklist cramfs
blacklist freevxfs
blacklist hfs
blacklist hfsplus
blacklist jffs2
blacklist udf
```

Reload :

```
sudo depmod -a
```

Validate :

```
modprobe --showconfig | grep -E "cramfs|freevxfs|hfs|hfsplus|jffs2|udf"
```
