https://docs.moodle.org/36/en/Installing_Moodle
https://www.vultr.com/docs/how-to-install-moodle-3-3-x-on-centos-7

## Cấu hình proxy

echo "proxy=http://10.10.10.86:3142" >> /etc/yum.conf

sudo yum install epel-release -y
sudo yum update -y


sed -i 's/SELINUX=enforcing/SELINUX=disabled/g' /etc/sysconfig/selinux
sed -i 's/SELINUX=enforcing/SELINUX=disabled/g' /etc/selinux/config
systemctl stop firewalld
systemctl disable firewalld

sudo init 6

## Install PHP

sudo yum install httpd -y
sudo sed -i 's/^/#&/g' /etc/httpd/conf.d/welcome.conf
sudo sed -i "s/Options Indexes FollowSymLinks/Options FollowSymLinks/" /etc/httpd/conf/httpd.conf

sudo systemctl start httpd.service
sudo systemctl enable httpd.service


echo '[mariadb]
name = MariaDB
baseurl = http://yum.mariadb.org/10.2/centos7-amd64
gpgkey=https://yum.mariadb.org/RPM-GPG-KEY-MariaDB
gpgcheck=1' >> /etc/yum.repos.d/MariaDB.repo
yum -y update

## Install MariaDB
yum install -y mariadb mariadb-server

systemctl start mariadb
systemctl enable mariadb

mysql -u root -p

CREATE DATABASE moodle DEFAULT CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
CREATE USER 'moodleuser'@'localhost' IDENTIFIED BY 'Cloud365a@123';
GRANT ALL PRIVILEGES ON moodle.* TO 'moodleuser'@'localhost' IDENTIFIED BY 'Cloud365a@123' WITH GRANT OPTION;
FLUSH PRIVILEGES;
EXIT;

## Install PHP

yum install -y http://rpms.remirepo.net/enterprise/remi-release-7.rpm
yum install -y yum-utils
yum-config-manager --enable remi-php72

yum install php php-common php-xmlrpc php-soap php-mysql php-dom php-mbstring php-gd php-ldap php-pdo php-json php-xml php-zip php-curl php-mcrypt php-pear php-intl setroubleshoot-server -y


## Down source moodle

cd
yum install -y wget

wget https://download.moodle.org/download.php/direct/stable36/moodle-latest-36.tgz

sudo tar -zxvf moodle-latest-36.tgz -C /var/www/html

sudo chown -R root:root /var/www/html/moodle


## Setup a dedicated data directory for Moodle

sudo mkdir /var/moodledata
sudo chown -R apache:apache /var/moodledata
sudo chmod -R 755 /var/moodledata


## Setup virtual host

cat <<EOF | sudo tee -a /etc/httpd/conf.d/moodle.conf
<VirtualHost *:80>

ServerAdmin thanhnb@nhanhoa.com.vn
DocumentRoot /var/www/html/moodle/
ServerName moodle.nhanhoa.local
ServerAlias www.moodle.nhanhoa.local
<Directory /var/www/html/moodle/>
    Options FollowSymLinks
    AllowOverride All
    Order allow,deny
    allow from all
</Directory>

ErrorLog /var/log/httpd/moodle.example.com-error_log
CustomLog /var/log/httpd/moodle.example.com-access_log common

</VirtualHost>
EOF


## Install Moodle

sudo /usr/bin/php /var/www/html/moodle/admin/cli/install.php

```
[root@moodle_1186 ~]# sudo /usr/bin/php /var/www/html/moodle/admin/cli/install.php
                                 .-..-.
   _____                         | || |
  /____/-.---_  .---.  .---.  .-.| || | .---.
  | |  _   _  |/  _  \/  _  \/  _  || |/  __ \
  * | | | | | || |_| || |_| || |_| || || |___/
    |_| |_| |_|\_____/\_____/\_____||_|\_____)

Moodle 3.6.3+ (Build: 20190501) command line installation program
-------------------------------------------------------------------------------
== Choose a language ==
en - English (en)
? - Available language packs
type value, press Enter to use default value (en)
: en
-------------------------------------------------------------------------------
== Data directories permission ==
type value, press Enter to use default value (2777)
:
-------------------------------------------------------------------------------
== Web address ==
type value
: http://10.10.11.86
-------------------------------------------------------------------------------
== Data directory ==
type value, press Enter to use default value (/var/www/html/moodledata)
: /var/moodledata
-------------------------------------------------------------------------------
== Choose database driver ==
mysqli
mariadb
type value, press Enter to use default value (mysqli)
: mariadb
-------------------------------------------------------------------------------
== Database host ==
type value, press Enter to use default value (localhost)
:
-------------------------------------------------------------------------------
== Database name ==
type value, press Enter to use default value (moodle)
:
-------------------------------------------------------------------------------
== Tables prefix ==
type value, press Enter to use default value (mdl_)
:
-------------------------------------------------------------------------------
== Database port ==
type value, press Enter to use default value ()
:
-------------------------------------------------------------------------------
== Unix socket ==
type value, press Enter to use default value ()
:
-------------------------------------------------------------------------------
== Database user ==
type value, press Enter to use default value (root)
: moodleuser
-------------------------------------------------------------------------------
== Database password ==
type value
: Cloud365a@123
-------------------------------------------------------------------------------
== Full site name ==
type value
: Moodle NhanHoa
-------------------------------------------------------------------------------
== Short name for site (eg single word) ==
type value
: moodle
-------------------------------------------------------------------------------
== Admin account username ==
type value, press Enter to use default value (admin)
: admin
-------------------------------------------------------------------------------
== New admin user password ==
type value
: Cloud365a@123
-------------------------------------------------------------------------------
== New admin user email address ==
type value, press Enter to use default value ()
: thanhnb@nhanhoa.com.vn
-------------------------------------------------------------------------------
== Upgrade key (leave empty to not set it) ==
type value
:
-------------------------------------------------------------------------------
== Copyright notice ==
Moodle  - Modular Object-Oriented Dynamic Learning Environment
Copyright (C) 1999 onwards Martin Dougiamas (http://moodle.com)

This program is free software: you can redistribute it and/or modify
it under the terms of the GNU General Public License as published by
the Free Software Foundation, either version 3 of the License, or
(at your option) any later version.

This program is distributed in the hope that it will be useful,
but WITHOUT ANY WARRANTY; without even the implied warranty of
MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.

See the Moodle License information page for full details:
http://docs.moodle.org/dev/License

Have you read these conditions and understood them?
type y (means yes) or n (means no)
: y


Installation completed successfully. => DONE
```


Modify permissions to /var/www/html/config.php

sudo chmod o+r /var/www/html/moodle/config.php


Setup a cron job

sudo crontab -u apache -e
* * * * *    /usr/bin/php /var/www/html/moodle/admin/cli/cron.php >/dev/null

## Restart Apache

sudo systemctl restart httpd.service


## Truy cập
> http://10.10.11.86/ (admin/Cloud365a@123)

> Done

