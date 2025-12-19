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

sudo dnf module install -y php:remi-8.4/minimal
sudo dnf install -y php-gd php-mysql php-mbstring php-bcmath php-xml php-curl php-zip php-intl php-sqlite3
sudo systemctl enable --now php-fpm
```
