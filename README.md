# System Setup

- Create `/etc/ssh/sshd_config.d/10-certs.conf`:

```
PermitRootLogin prohibit-password
```
- Restart sshd
- rsync TLS certificates to `/certs/`
- Add `/etc/yum.repos.d/nginx.repo`

```
[nginx-mainline]
name=nginx mainline repo
baseurl=http://nginx.org/packages/mainline/centos/10/x86_64/
gpgcheck=1
enabled=1
gpgkey=https://nginx.org/keys/nginx_signing.key
module_hotfixes=true
priority=1
```

- Install NGINX

```bash
sudo dnf install -y nginx
sudo systemctl enable --now nginx
```

- Add `/etc/nginx/conf.d/pelican.conf`:

```
server_tokens off;

server {
    listen 443 ssl default_server;

    root /var/www/pelican/public;
    index index.php;

    # allow larger file uploads and longer script runtimes
    client_max_body_size 100m;
    client_body_timeout 120s;

    sendfile off;

    ssl_certificate /etc/certs/fullchain.pem;
    ssl_certificate_key /etc/certs/privkey.pem;
    ssl_session_cache shared:SSL:10m;
    ssl_protocols TLSv1.3;
    ssl_prefer_server_ciphers on;

    location / {
        try_files $uri $uri/ /index.php?$query_string;
    }

    location ~ \.php$ {
        fastcgi_split_path_info ^(.+\.php)(/.+)$;
        fastcgi_pass unix:/var/run/php-fpm/pelican.sock;
        fastcgi_index index.php;
        include fastcgi_params;
        fastcgi_param PHP_VALUE "upload_max_filesize = 100M \n post_max_size=100M";
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
        fastcgi_param HTTP_PROXY "";
        fastcgi_intercept_errors off;
        fastcgi_buffer_size 16k;
        fastcgi_buffers 4 16k;
        fastcgi_connect_timeout 300;
        fastcgi_send_timeout 300;
        fastcgi_read_timeout 300;
        include /etc/nginx/fastcgi_params;
    }

    location ~ /\.ht {
        deny all;
    }
}
```

- Configure SELinux

```bash
sudo setsebool -P httpd_can_network_redis 1
```

- Install Remi's repository

```bash
sudo subscription-manager repos --enable codeready-builder-for-rhel-10-x86_64-rpms
sudo dnf install -y https://dl.fedoraproject.org/pub/epel/epel-release-latest-10.noarch.rpm
sudo dnf install -y https://rpms.remirepo.net/enterprise/remi-release-10.rpm
```

- Install Valkey 9.0

```bash
sudo dnf module install -y valkey:remi-9.0
sudo systemctl enable --now valkey
```

- Install PHP 8.4

```bash
# Remove hardened_malloc from being preloaded, as it breaks php-fpm
sudo rm /etc/ld.so.preload

sudo dnf module install -y php:remi-8.4/minimal

# Copy pasting the extension list from the documentation
# Some may be included in php-common already, but we are being explicit with what's required here
# The official documentation forgot to mention php-sodium, but it is needed for `composer install`,
# so we will include it
sudo dnf install -y php-{gd,mysql,mbstring,bcmath,xml,curl,zip,intl,sqlite3,fpm,sodium}

sudo dnf module install -y composer:2

sudo systemctl enable --now php-fpm
```

- Add `/etc/php-fpm.d/pelican.conf`

```
[pelican]

user = nginx
group = nginx

listen = /var/run/php-fpm/pelican.sock
listen.owner = nginx
listen.group = nginx
listen.mode = 0750

# Arbitary limits
# Taken from https://pterodactyl.io/community/installation-guides/panel/centos8.html#install-dependencies
pm = ondemand
pm.max_children = 9
pm.process_idle_timeout = 10s
pm.max_requests = 200
```

- Restart `php-fpm`

- Add `/etc/yum.repos.d/mariadb.repo`

```
[mariadb]
name = MariaDB
baseurl = https://rpm.mariadb.org/11.8/rhel/$releasever/$basearch
gpgkey= https://rpm.mariadb.org/RPM-GPG-KEY-MariaDB
gpgcheck=1
```

- Install MariaDB

```bash
# MariaDB-client is a dependency of MariaDB-server, but we are being explicit here
# since the client is needed to manage the server anyways
sudo dnf install -y MariaDB-server MariaDB-client
sudo systemctl enable --now mariadb
```

- Secure MariaDB

```
[root@pelican ~]# sudo mariadb-secure-installation

Enter current password for root (enter for none):
Switch to unix_socket authentication [Y/n] y
Change the root password? [Y/n] n
Remove anonymous users? [Y/n] y
Remove test database and access to it? [Y/n] y
Reload privilege tables now? [Y/n] y
```

- Switch to the root user

- Create MariaDB user

```
[root@pelican ~]# mariadb -u root

CREATE USER 'pelican'@'127.0.0.1' IDENTIFIED BY 'REDACTED';
CREATE DATABASE pelican;
GRANT ALL PRIVILEGES ON *.* TO 'pelican'@'127.0.0.1' WITH GRANT OPTION;
exit
```

- Setup Pelican

```bash
mkdir -p /var/www/pelican
cd /var/www/pelican
curl -L https://github.com/pelican-dev/panel/releases/latest/download/panel.tar.gz | tar -xzv
COMPOSER_ALLOW_SUPERUSER=1 composer install --no-dev --optimize-autoloader
php artisan p:environment:setup
chmod -R 755 storage/* bootstrap/cache/
sudo chown -R nginx:nginx /var/www/pelican
```

- Configure Pelican MariaDB
```
[root@pelican pelican]# php artisan p:environment:database

 Do you want to continue? (yes/no) [no]:
 > yes

 Database Driver [SQLite (recommended)]:
 > mariadb                 

 Database Host [127.0.0.1]:
 > 

 Database Port [3306]:
 > 

 Database Name [panel]:
 > pelican                               

 Database Username [pelican]:
 > 

 Database Password:
 >
```

- Configure Pelican Redis

```
[root@pelican pelican]# php artisan p:redis:setup                       

 Redis Host [127.0.0.1]:
 >              

 Redis User:
 > 

 Redis Password:
 > 

 Redis Port [6379]:
 > 

 Queue worker service name [pelican-queue]:
 > 

 Webserver User [www-data]:
 > nginx

 Webserver Group [www-data]:
 > nginx
```

- Edit `/var/www/pelican/.env` and set `APP_URL`.
