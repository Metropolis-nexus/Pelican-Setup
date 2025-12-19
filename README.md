# System Setup

- Create `/etc/ssh/sshd_config.d/10-certs.conf`:

```
PermitRootLogin prohibit-password
```
- Restart sshd
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

```
sudo dnf install -y nginx
sudo systemctl enable --now nginx
```

- Install Remi's repository

```
sudo subscription-manager repos --enable codeready-builder-for-rhel-10-x86_64-rpms
sudo dnf install -y https://dl.fedoraproject.org/pub/epel/epel-release-latest-10.noarch.rpm
sudo dnf install -y https://rpms.remirepo.net/enterprise/remi-release-10.rpm
```

- Install Valkey 9.0

```
sudo dnf module install -y valkey:remi-9.0
sudo systemctl enable --now valkey
```

- Install PHP 8.4

```
# Remove hardened_malloc from being preloaded, as it breaks php-fpm
sudo rm /etc/ld.so.preload

# Copy pasting the extension list from the documentation
# Some may be included in php-common already, but we are being explicit with what's required here
# We will include cli despite of it already being a dependency since it's needed for management 
# The official documentation forgot to mention pear but we will also include it
sudo dnf install -y php84-php-{gd,mysql,mbstring,bcmath,xml,curl,zip,intl,sqlite3,fpm,cli,pear}

sudo systemctl enable --now php-fpm

sudo dnf module install -y composer:2
```

- Add `/etc/yum.repos.d/mariadb.repo`

```
[mariadb]
name = MariaDB
baseurl = https://rpm.mariadb.org/11.8/rhel/$releasever/$basearch
gpgkey= https://rpm.mariadb.org/RPM-GPG-KEY-MariaDB
gpgcheck=1
```

- Install MariaDB

```
# MariaDB-client is a dependency of MariaDB-server, but we are being explicit here
# since the client is needed to manage the server anyways
sudo dnf install -y MariaDB-server MariaDB-client
sudo systemctl enable --now mariadb
```

- Secure MariaDB

```
[root@pelican ~]# mariadb-secure-installation

Enter current password for root (enter for none):
Switch to unix_socket authentication [Y/n] y
Change the root password? [Y/n] n
Remove anonymous users? [Y/n] y
Remove test database and access to it? [Y/n] y
Reload privilege tables now? [Y/n] y
```

- Create MariaDB user

```
[root@pelican ~]# mariadb -u root

CREATE USER 'pelican'@'127.0.0.1' IDENTIFIED BY 'REDACTED';
CREATE DATABASE pelican;
GRANT ALL PRIVILEGES ON *.* TO 'pelican'@'127.0.0.1' WITH GRANT OPTION;
exit
```
