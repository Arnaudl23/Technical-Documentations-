# Server WEB

```arduino
Installer Apache (httpd) :
 
sudo dnf install httpd -y
sudo systemctl enable --now httpd
sudo systemctl enable httpd
sudo systemctl start httpd

sudo firewall-cmd --permanent --add-service=http
sudo firewall-cmd --reload

Création d'un répertoire :

mkdir -p /var/www/html/wordpress

Désactiver la page par défaut d’Apache :

sudo mv /etc/httpd/conf.d/welcome.conf /etc/httpd/conf.d/welcome.conf.disabled
sudo systemctl restart httpd

```

```arduino
Configurer un VirtualHost :

Créer /etc/httpd/conf.d/wordpress.conf

<VirtualHost *:80>
        ServerName wp.cqa.lan
        ServerAlias www.wp.cqa.lan
        ServerAdmin admin@.cqa.lan

        <Directory /var/www/html/wordpress>
                AllowOverride all
                Require all granted
        </Directory>

        DocumentRoot /var/www/html/wordpress
        ErrorLog /var/log/httpd/cqa.lan_error.log
        CustomLog /var/log/httpd/wordpress_access.log combined
</VirtualHost>

sudo systemctl restart httpd
```

```arduino
Installation WordPress : 

cd /tmp
wget https://wordpress.org/latest.zip
tar -xvzf latest.tar.gz
sudo cp -rf wordpress/* /var/www/html/wordpress/

Définir les permissions :

chown -R apache:apache /var/www/html/wordpress
chmod -R 755 /var/www/html/wordpress

Installation MariaDB : 

sudo dnf install mariadb-server mariadb -y
sudo systemctl enable mariadb
sudo systemctl start mariadb

Entrez dans MariaDB : 

mariadb -u root -p

CREATE DATABASE wp2025_cqalan;
CREATE USER 'wpuser'@'localhost' IDENTIFIED BY 'motdepasse';
GRANT ALL PRIVILEGES ON wordpress.* TO 'wpuser'@'localhost';
FLUSH PRIVILEGES;
EXIT;
```
