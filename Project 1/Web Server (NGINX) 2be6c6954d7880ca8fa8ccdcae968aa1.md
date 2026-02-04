# Web Server (NGINX)

# Configuring LVMs

LVMs allow you to create flexible, resizable partitions spread over several disks. Perfect for isolated websites, as each site has its own independent storage.

### Check free space on disk

Before creating an LVM, you need to see if there is any space available on the disk (here /dev/sda)

```bash
sudo fdisk -l /dev/sda
```

Example output:

```bash
Disk /dev/sda: 931.51 GiB
Device     Boot      Start        End    Sectors   Size Id Type
/dev/sda1             2048    2099199    2097152     1G 83 Linux
/dev/sda2          2099200 1213974527 1211875328 577.9G 8e Linux LVM
/dev/sda3  *    1213974528 1215023103    1048576   512M  6 FAT16
```

### Create a new dedicated LVM partition

We create a partition **type LVM (8e)**:

```bash
sudo fdisk /dev/sda
```

In `fdisk`:

```bash
n  -> new partition
p  -> primary
4  -> partition number
Enter -> start
Enter -> end(max)
t -> Change partition type
4 -> partition 4
8e -> Linux LVM
w -> save partitioning
```

Reload the partition table:

```bash
sudo partprobe
```

### Initialize the Physical Volume

PV is the lowest layer of LVM. It represents the raw usable capacity.

```bash
sudo pvcreate /dev/sda4
```

### Create a Volume Group

A VG is a “pool” which groups together one or more Physical Volumes and which is used to create Logical Volumes.

```bash
sudo vgcreate vg_web /dev/sda4
```

We can add another disk later, LVM will automatically merge the space.

### Create Logical Volumes

Each site will have its own isolated volume. In the event of corruption, mishandling, deletion: **no impact on other sites**.

```bash
sudo lvcreate -n site1 -L 20G vg_web
sudo lvcreate -n site2 -L 20G vg_web
```

### Format LVs

Ext4 is stable, ideal for a web server.

```bash
sudo mkfs.ext4 /dev/vg_web/site1
sudo mkfs.ext4 /dev/vg_web/site2
```

### Create site directories

Each site will have its dedicated folder:

```bash
sudo mkdir -p /var/www/{site1,site2}
```

### Mount LVs permanently via `fstab`

Check UUIDs:

```bash
sudo blkid /dev/vg_web/site{1,2}

/dev/vg_web/site1: UUID="fd551bef-9b33-4076-9bd6-5976df4137ba" TYPE="ext4"
/dev/vg_web/site2: UUID="6c4d26da-12fb-40c3-a90e-2fd9baff036c" TYPE="ext4"
```

Modify `/etc/fstab` and add:

```bash
UUID=fd551bef-9b33-4076-9bd6-5976df4137ba  /var/www/site1  ext4  defaults  0 2
UUID=6c4d26da-12fb-40c3-a90e-2fd9baff036c  /var/www/site2  ext4  defaults  0 2
```

Then go up:

```bash
sudo mount -a
```

Check with:

```bash
df -h | grep /var/www

/dev/mapper/vg_web-site1   20G   24K   19G   1% /var/www/site1
/dev/mapper/vg_web-site2   20G   24K   19G   1% /var/www/site2
```

### Creating test HTML pages

Allows you to validate that the NGINX VHosts will point to the right place.

```bash
sudo mkdir -p /var/www/{site1,site2}/html
sudo touch /var/www/{site1,site2}/html/index.html

echo '<h1>Bienvenue sur site1.infra.labo</h1>' | sudo tee /var/www/site1/html/index.html
echo '<h1>Bienvenue sur site2.infra.labo</h1>' | sudo tee /var/www/site2/html/index.html
```

### Adjust SELinux contexts

```bash
sudo restorecon -Rv /var/www/{site1,site2}
```

Check :

```bash
ls -Z /var/www/site{1,2}

/var/www/site1:
unconfined_u:object_r:httpd_sys_content_t:s0 html   system_u:object_r:httpd_sys_content_t:s0 lost+found

/var/www/site2:
unconfined_u:object_r:httpd_sys_content_t:s0 html   system_u:object_r:httpd_sys_content_t:s0 lost+found
```

# Managing internal TLS certificates

### Generate private keys

We generate the private keys of the different sites and also a generic key.

```bash
sudo openssl genrsa -out /etc/pki/tls/private/site1.infra.labo.key 4096
sudo openssl genrsa -out /etc/pki/tls/private/site2.infra.labo.key 4096
```

Assign permissions:

```bash
sudo chmod 600 /etc/pki/tls/private/*.key
```

### Generate the CSR (Certificate Signing Request) of the sites

```bash
sudo openssl req -new \\
  -key /etc/pki/tls/private/site1.infra.labo.key \\
  -out /tmp/site1.infra.labo.csr \\
  -subj "/C=BE/ST=Wallonie/L=Ciney/O=Infra/OU=Infra-Web/CN=site1.infra.labo" \\
  -addext "subjectAltName=DNS:site1.infra.labo,DNS:www.site1.infra.labo"
```

```bash
sudo openssl req -new \\
  -key /etc/pki/tls/private/site2.infra.labo.key \\
  -out /tmp/site2.infra.labo.csr \\
  -subj "/C=BE/ST=Wallonie/L=Ciney/O=Infra/OU=Infra-Web/CN=site2.infra.labo" \\
  -addext "subjectAltName=DNS:site2.infra.labo,DNS:www.site2.infra.labo"
```

### Sign certificates from the intermediate CA

```bash
openssl ca -extensions server_cert -notext \\
  -in /etc/pki/CA/intermediate/csr/site1.infra.labo.csr \\
  -out /etc/pki/CA/intermediate/certs/site1.infra.labo.crt
```

```bash
openssl ca -extensions server_cert -notext \\
  -in /etc/pki/CA/intermediate/csr/site2.infra.labo.csr \\
  -out /etc/pki/CA/intermediate/certs/site2.infra.labo.crt
```

### Building chains of trust

Nginx requires:

- Server certificate
- Intermediate certificate
- Root Certificate

To do this, we will concatenate the different certificates into a “chain of trust”:

```bash
cat /etc/pki/CA/intermediate/certs/site1.infra.labo.crt \\
		/etc/pki/CA/intermediate/certs/ca-chain.crt \\
    > /home/admin/site1-fullchain.crt
```

```bash
cat /etc/pki/CA/intermediate/certs/site2.infra.labo.crt \\
		/etc/pki/CA/intermediate/certs/ca-chain.crt \\
    > /home/admin/site2-fullchain.crt
```

### Copy certificates to Nginx server

Place the different certificates in`/etc/pki/tls/certs/`, then give them the correct SELinux context:

```bash
sudo restorecon -Rv /etc/pki/tls
```

# Installing NGINX

### Installation

```bash
sudo dnf install nginx -y
sudo systemctl enable --now nginx
```

Verification :

```bash
sudo systemctl status nginx
```

### Open HTTP(S) ports

```bash
sudo firewall-cmd --permanent --zone=public --add-service=http
sudo firewall-cmd --permanent --zone=public --add-service=https
sudo firewall-cmd --reload
```

Verification :

```bash
sudo firewall-cmd --list-all
```

# Configuring VirtualHosts

For site 1, create a file`/etc/nginx/conf.d/site1.infra.labo.conf` :

```bash
# HTTPS Redirection 
server {
    listen 80;
    server_name site1.infra.labo www.site1.infra.labo;
    return 301 https://$host$request_uri;
}

# HTTPS Configuration
server {
    listen 443 ssl http2;
    server_name site1.infra.labo www.site1.infra.labo;

    root /var/www/site1/html;
    index index.html;

    access_log /var/log/nginx/site1_access.log;
    error_log  /var/log/nginx/site1_error.log;

    ssl_certificate     /etc/pki/tls/certs/site1-fullchain.crt;
    ssl_certificate_key /etc/pki/tls/private/site1.infra.labo.key;

    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_prefer_server_ciphers on;

		error_page 404 /404.html;
    location = /404.html {
    }
    
    location / {
        try_files $uri $uri/ =404;
    }
}
```

For site 2, create a file`/etc/nginx/conf.d/site2.infra.labo.conf` :

```bash
# HTTPS Redirection
server {
    listen 80;
    server_name site2.infra.labo www.site2.infra.labo;
    return 301 https://$host$request_uri;
}

# HTTPS Configuration
server {
    listen 443 ssl http2;
    server_name site2.infra.labo www.site2.infra.labo;

    root /var/www/site2/html;
    index index.html;

    access_log /var/log/nginx/site2_access.log;
    error_log  /var/log/nginx/site2_error.log;

    ssl_certificate     /etc/pki/tls/certs/site2-fullchain.crt;
    ssl_certificate_key /etc/pki/tls/private/site2.infra.labo.key;

    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_prefer_server_ciphers on;
    
    error_page 404 /404.html;
    location = /404.html {
    }

    location / {
        try_files $uri $uri/ =404;
    }
}
```

### Test and reload NGINX

```bash
sudo nginx -t
sudo systemctl reload nginx
```

# Default error page

### Generate the generic private key

We generate the private keys of the different sites and also a generic key.

```bash
sudo openssl genrsa -out /etc/pki/tls/private/web.key 4096
```

### Generate the CSR (Certificate Signing Request)

```bash
sudo openssl req -new \\
  -key /etc/pki/tls/private/web.key \\
  -out /tmp/web.csr \\
  -subj "/C=BE/ST=Wallonie/L=Ciney/O=Infra/OU=Infra-Web/CN=web.infra.labo" \\
  -addext "subjectAltName=DNS:web.infra.labo,IP:172.20.3.80"
```

### Sign the certificate from the intermediate CA

```bash
sudo openssl ca -extensions server_cert -notext \\
  -in /etc/pki/CA/intermediate/csr/web.csr \\
  -out /etc/pki/CA/intermediate/certs/web.crt
```

### Building the chain of trust

To do this, we will concatenate the different certificates into a “chain of trust”:

```bash
cat /etc/pki/CA/intermediate/certs/web.crt \\
		/etc/pki/CA/intermediate/certs/ca-chain.crt \\
    > /home/admin/web-fullchain.crt
```

### Copy certificates to Nginx server

Place the different certificates in `/etc/pki/tls/certs/`, then give them the correct SELinux context:

```bash
sudo restorecon -Rv /etc/pki/tls
```

### Create error pages

Create a default folder which will contain the error pages

```bash
sudo mkdir /var/www/default
```

Next, create a page `404.html` and place it in this folder

### Configure the default page

Edit file `/etc/nginx/nginx.conf` :

```bash
server {
    listen 80 default_server;
    server_name _;

    return 301 https://$host$request_uri;
}
server {
    listen        443 ssl default_server;
    server_name  _;

    root /var/www/default;
    index index.html;

    ssl_certificate     /etc/pki/tls/certs/web-fullchain.crt;
    ssl_certificate_key /etc/pki/tls/private/web.key;

    ssl_protocols TLSv1.2 TLSv1.3;

    error_page 404 /404.html;
    location = /404.html {
    }

    error_page 500 502 503 504 /50x.html;
    location = /50x.html {
    }

    location ~ /\\. {
        deny all;
    }

    location / {
            try_files $uri $uri/ /index.html;
    }
}
```

### Adjust SELinux contexts

```bash
sudo restorecon -Rv /var/www/default
```