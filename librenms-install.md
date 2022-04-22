## Install Required Packages
apt install software-properties-common
add-apt-repository universe
apt update
apt install acl curl composer fping git graphviz imagemagick mariadb-client mariadb-server mtr-tiny nginx-full nmap php7.4-cli php7.4-curl php7.4-fpm php7.4-gd php7.4-gmp php7.4-json php7.4-mbstring php7.4-mysql php7.4-snmp php7.4-xml php7.4-zip rrdtool snmp snmpd whois unzip python3-pymysql python3-dotenv python3-redis python3-setuptools python3-systemd python3-pip

## Add librenms user:
useradd librenms -d /opt/librenms -M -r -s "$(which bash)"

## Download LibreNMS and Set permissions
cd /opt
git clone https://github.com/librenms/librenms.git

chown -R librenms:librenms /opt/librenms
chmod 771 /opt/librenms
setfacl -d -m g::rwx /opt/librenms/rrd /opt/librenms/logs /opt/librenms/bootstrap/cache/ /opt/librenms/storage/
setfacl -R -m g::rwx /opt/librenms/rrd /opt/librenms/logs /opt/librenms/bootstrap/cache/ /opt/librenms/storage/

##  PHP dependencies
su - librenms
./scripts/composer_wrapper.php install --no-dev
exit

## Timezone:
vi /etc/php/7.4/fpm/php.ini
vi /etc/php/7.4/cli/php.ini
timedatectl set-timezone Etc/UTC

## Configure MariaDB:
vi /etc/mysql/mariadb.conf.d/50-server.cnf
innodb_file_per_table=1
lower_case_table_names=0

systemctl enable mariadb
systemctl restart mariadb

mysql -u root

CREATE DATABASE librenms CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
CREATE USER 'librenms'@'localhost' IDENTIFIED BY 'password';
GRANT ALL PRIVILEGES ON librenms.* TO 'librenms'@'localhost';
FLUSH PRIVILEGES;
exit

## Configure PHP-FPM:
cp /etc/php/7.4/fpm/pool.d/www.conf /etc/php/7.4/fpm/pool.d/librenms.conf
vi /etc/php/7.4/fpm/pool.d/librenms.conf

Change [www] to [librenms]:
[librenms]

Change user and group to "librenms":
user = librenms
group = librenms

Change listen to a unique name:
listen = /run/php-fpm-librenms.sock

## Configure Web Server:
vi /etc/nginx/conf.d/librenms.conf

server {
 listen      80;
 server_name librenms.example.com;
 root        /opt/librenms/html;
 index       index.php;

 charset utf-8;
 gzip on;
 gzip_types text/css application/javascript text/javascript application/x-javascript image/svg+xml text/plain text/xsd text/xsl text/xml image/x-icon;
 location / {
  try_files $uri $uri/ /index.php?$query_string;
 }
 location ~ [^/]\.php(/|$) {
  fastcgi_pass unix:/run/php-fpm-librenms.sock;
  fastcgi_split_path_info ^(.+\.php)(/.+)$;
  include fastcgi.conf;
 }
 location ~ /\.(?!well-known).* {
  deny all;
 }
}

rm /etc/nginx/sites-enabled/default
systemctl restart nginx
systemctl restart php7.4-fpm

## Enable lnms
ln -s /opt/librenms/lnms /usr/bin/lnms
cp /opt/librenms/misc/lnms-completion.bash /etc/bash_completion.d/

## Snmpd
cp /opt/librenms/snmpd.conf.example /etc/snmp/snmpd.conf
vi /etc/snmp/snmpd.conf

curl -o /usr/bin/distro https://raw.githubusercontent.com/librenms/librenms-agent/master/snmp/distro
chmod +x /usr/bin/distro
systemctl enable snmpd
systemctl restart snmpd

## Job and logrotate
cp /opt/librenms/librenms.nonroot.cron /etc/cron.d/librenms
cp /opt/librenms/misc/librenms.logrotate /etc/logrotate.d/librenms

## Web installer
http://librenms.example.com/install
