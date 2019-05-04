# Triển khai HA Moodle NFS Pace Haproxy
---
## Chuẩn bị

### Tại moodle01

Cấu hình Hostname
```
echo "10.10.10.94 moodle01" >> /etc/hosts
echo "10.10.10.95 moodle02" >> /etc/hosts
echo "10.10.10.96 moodle03" >> /etc/hosts
```

Cấu hình Network
```
echo "Setup IP eth0"
nmcli c modify eth0 ipv4.addresses 10.10.10.94/24
nmcli c modify eth0 ipv4.gateway 10.10.10.1
nmcli c modify eth0 ipv4.dns 8.8.8.8
nmcli c modify eth0 ipv4.method manual
nmcli con mod eth0 connection.autoconnect yes

echo "Setup IP eth1"
nmcli c modify eth1 ipv4.addresses 10.10.11.94/24
nmcli c modify eth1 ipv4.method manual
nmcli con mod eth1 connection.autoconnect yes
```

Cài đặt gói
```
yum install epel-release -y
yum update -y
```

Tắt SELinux, Firewalld
```
sed -i 's/SELINUX=enforcing/SELINUX=disabled/g' /etc/sysconfig/selinux
sed -i 's/SELINUX=enforcing/SELINUX=disabled/g' /etc/selinux/config
systemctl stop firewalld
systemctl disable firewalld
```

### Tạo moodle02

Cấu hình Hostname
```
echo "10.10.10.94 moodle01" >> /etc/hosts
echo "10.10.10.95 moodle02" >> /etc/hosts
echo "10.10.10.96 moodle03" >> /etc/hosts
```

Cấu hình Network
```
echo "Setup IP eth0"
nmcli c modify eth0 ipv4.addresses 10.10.10.95/24
nmcli c modify eth0 ipv4.gateway 10.10.10.1
nmcli c modify eth0 ipv4.dns 8.8.8.8
nmcli c modify eth0 ipv4.method manual
nmcli con mod eth0 connection.autoconnect yes

echo "Setup IP eth1"
nmcli c modify eth1 ipv4.addresses 10.10.11.95/24
nmcli c modify eth1 ipv4.method manual
nmcli con mod eth1 connection.autoconnect yes
```

Cài đặt gói
```
yum install epel-release -y
yum update -y
```

Tắt SELinux, Firewalld
```
sed -i 's/SELINUX=enforcing/SELINUX=disabled/g' /etc/sysconfig/selinux
sed -i 's/SELINUX=enforcing/SELINUX=disabled/g' /etc/selinux/config
systemctl stop firewalld
systemctl disable firewalld
```

### Tạo moodle03

Cấu hình Hostname
```
echo "10.10.10.94 moodle01" >> /etc/hosts
echo "10.10.10.95 moodle02" >> /etc/hosts
echo "10.10.10.96 moodle03" >> /etc/hosts
```

Cấu hình Network
```
echo "Setup IP eth0"
nmcli c modify eth0 ipv4.addresses 10.10.10.96/24
nmcli c modify eth0 ipv4.gateway 10.10.10.1
nmcli c modify eth0 ipv4.dns 8.8.8.8
nmcli c modify eth0 ipv4.method manual
nmcli con mod eth0 connection.autoconnect yes

echo "Setup IP eth1"
nmcli c modify eth1 ipv4.addresses 10.10.11.96/24
nmcli c modify eth1 ipv4.method manual
nmcli con mod eth1 connection.autoconnect yes
```

Cài đặt gói
```
yum install epel-release -y
yum update -y
```

Tắt SELinux, Firewalld
```
sed -i 's/SELINUX=enforcing/SELINUX=disabled/g' /etc/sysconfig/selinux
sed -i 's/SELINUX=enforcing/SELINUX=disabled/g' /etc/selinux/config
systemctl stop firewalld
systemctl disable firewalld
```

## Cài đặt

### Install HTTPD

sudo yum install httpd -y

cat /etc/httpd/conf/httpd.conf | grep 'Listen 80'
sed -i "s/Listen 80/Listen 10.10.10.94:80/g" /etc/httpd/conf/httpd.conf

sed -i "s/Listen 80/Listen 10.10.10.95:80/g" /etc/httpd/conf/httpd.conf

sed -i "s/Listen 80/Listen 10.10.10.96:80/g" /etc/httpd/conf/httpd.conf

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

yum install -y galera rsync

systemctl stop mariadb
systemctl disable mariadb

### Log mariadb

cp /etc/my.cnf /etc/my.cnf.org

echo '[mysqld]
slow_query_log                  = 1
slow_query_log_file             = /var/log/mariadb/slow.log
long_query_time                 = 5
log_error                       = /var/log/mariadb/error.log
general_log_file                = /var/log/mariadb/mysql.log
general_log                     = 1

[client-server]
!includedir /etc/my.cnf.d' > /etc/my.cnf


mkdir -p /var/log/mariadb/
chown -R mysql. /var/log/mariadb/

### Tại moodle01

```
cp /etc/my.cnf.d/server.cnf /etc/my.cnf.d/server.cnf.bak

echo '[server]
[mysqld]
bind-address=10.10.10.94

[galera]
wsrep_on=ON
wsrep_provider=/usr/lib64/galera/libgalera_smm.so
#add your node ips here
wsrep_cluster_address="gcomm://10.10.11.94,10.10.11.95,10.10.11.96"
binlog_format=row
default_storage_engine=InnoDB
innodb_autoinc_lock_mode=2
#Cluster name
wsrep_cluster_name="moodle_cluster"
# Allow server to accept connections on all interfaces.
bind-address=10.10.10.94
# this server ip, change for each server
wsrep_node_address="10.10.11.94"
# this server name, change for each server
wsrep_node_name="moodle01"
wsrep_sst_method=rsync
[embedded]
[mariadb]
[mariadb-10.2]
' > /etc/my.cnf.d/server.cnf
```

### Tại moodle02

```
cp /etc/my.cnf.d/server.cnf /etc/my.cnf.d/server.cnf.bak

echo '[server]
[mysqld]
bind-address=10.10.10.95

[galera]
wsrep_on=ON
wsrep_provider=/usr/lib64/galera/libgalera_smm.so
#add your node ips here
wsrep_cluster_address="gcomm://10.10.11.94,10.10.11.95,10.10.11.96"
binlog_format=row
default_storage_engine=InnoDB
innodb_autoinc_lock_mode=2
#Cluster name
wsrep_cluster_name="moodle_cluster"
# Allow server to accept connections on all interfaces.
bind-address=10.10.10.95
# this server ip, change for each server
wsrep_node_address="10.10.11.95"
# this server name, change for each server
wsrep_node_name="moodle02"
wsrep_sst_method=rsync
[embedded]
[mariadb]
[mariadb-10.2]
' > /etc/my.cnf.d/server.cnf
```

### Tại moodle03

```
cp /etc/my.cnf.d/server.cnf /etc/my.cnf.d/server.cnf.bak

echo '[server]
[mysqld]
bind-address=10.10.10.96

[galera]
wsrep_on=ON
wsrep_provider=/usr/lib64/galera/libgalera_smm.so
#add your node ips here
wsrep_cluster_address="gcomm://10.10.11.94,10.10.11.95,10.10.11.96"
binlog_format=row
default_storage_engine=InnoDB
innodb_autoinc_lock_mode=2
#Cluster name
wsrep_cluster_name="moodle_cluster"
# Allow server to accept connections on all interfaces.
bind-address=10.10.10.96
# this server ip, change for each server
wsrep_node_address="10.10.11.96"
# this server name, change for each server
wsrep_node_name="moodle03"
wsrep_sst_method=rsync
[embedded]
[mariadb]
[mariadb-10.2]
' > /etc/my.cnf.d/server.cnf
```

### moodle01
galera_new_cluster
systemctl enable mariadb

### moodle02, moodle03
systemctl start mariadb
systemctl enable mariadb

### Kiểm tra

mysql -u root -e "SHOW STATUS LIKE 'wsrep_cluster_size'"
mysql -u root -e "SHOW STATUS LIKE 'wsrep%'"

### DATABASE
mysql -u root -p

CREATE DATABASE moodle DEFAULT CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
CREATE USER 'moodleuser'@'localhost' IDENTIFIED BY 'Cloud365a@123';
GRANT ALL PRIVILEGES ON moodle.* TO 'moodleuser'@'localhost' IDENTIFIED BY 'Cloud365a@123' WITH GRANT OPTION;

CREATE USER 'moodleuser'@'%' IDENTIFIED BY 'Cloud365a@123';
GRANT ALL PRIVILEGES ON moodle.* TO 'moodleuser'@'%' IDENTIFIED BY 'Cloud365a@123' WITH GRANT OPTION;

CREATE USER 'moodleuser'@'moodle01' IDENTIFIED BY 'Cloud365a@123';
GRANT ALL PRIVILEGES ON moodle.* TO 'moodleuser'@'moodle01' IDENTIFIED BY 'Cloud365a@123' WITH GRANT OPTION;

CREATE USER 'moodleuser'@'moodle02' IDENTIFIED BY 'Cloud365a@123';
GRANT ALL PRIVILEGES ON moodle.* TO 'moodleuser'@'moodle02' IDENTIFIED BY 'Cloud365a@123' WITH GRANT OPTION;

CREATE USER 'moodleuser'@'moodle03' IDENTIFIED BY 'Cloud365a@123';
GRANT ALL PRIVILEGES ON moodle.* TO 'moodleuser'@'moodle03' IDENTIFIED BY 'Cloud365a@123' WITH GRANT OPTION;

FLUSH PRIVILEGES;
EXIT;


## Haproxy
http://www.haproxy.org/download/1.8/src/haproxy-1.8.20.tar.gz

Bước 1: Cài đặt HAProxy 1.8

sudo yum install wget socat -y
wget http://cbs.centos.org/kojifiles/packages/haproxy/1.8.1/5.el7/x86_64/haproxy18-1.8.1-5.el7.x86_64.rpm 
yum install haproxy18-1.8.1-5.el7.x86_64.rpm -y

echo 'global
    log         127.0.0.1 local2
    chroot      /var/lib/haproxy
    pidfile     /var/run/haproxy.pid
    maxconn     4000
    user        haproxy
    group       haproxy
    daemon
    stats socket /var/lib/haproxy/stats
defaults
    mode                    http
    log                     global
    option                  httplog
    option                  dontlognull
    option http-server-close
    option forwardfor       except 127.0.0.0/8
    option                  redispatch
    retries                 3
    option  http-server-close
    timeout http-request    10s
    timeout queue           1m
    timeout connect         10s
    timeout client          1m
    timeout server          1m
    timeout http-keep-alive 10s
    timeout check           10s
    maxconn                 3000

listen stats
    bind :8080
    mode http
    stats enable
    stats uri /stats
    stats realm HAProxy\ Statistics
    stats admin if TRUE
    stats auth admin:portal123

listen galera
    bind 10.10.10.97:3306
    balance source
    mode tcp
    option httpchk
    option tcpka
    option tcplog
    option clitcpka
    option srvtcpka
    timeout client 28801s
    timeout server 28801s 
    server moodle01 10.10.10.94:3306 check port 9200  inter 5s fastinter 2s rise 3 fall 3 
    server moodle02 10.10.10.95:3306 check port 9200  inter 5s fastinter 2s rise 3 fall 3 backup
    server moodle03 10.10.10.96:3306 check port 9200  inter 5s fastinter 2s rise 3 fall 3 backup

listen web-backend
    bind 10.10.10.97:80
    balance leastconn
    cookie SERVERID insert indirect nocache
    mode  http
    option  forwardfor
    option  httpchk GET / HTTP/1.0
    option  httpclose
    option  httplog
    timeout  client 3h
    timeout  server 3h
    server moodle01 10.10.10.94:80  weight 1 check cookie s1
    server moodle02 10.10.10.95:80  weight 1 check cookie s2
    server moodle03 10.10.10.96:80  weight 1 check cookie s3' > /etc/haproxy/haproxy.cfg


Bước 2: Cầu hình Plugin check Galera

## Cấu hình log Haproxy đẩy log ra file riêng

sed -i "s/#\$ModLoad imudp/\$ModLoad imudp/g" /etc/rsyslog.conf
sed -i "s/#\$UDPServerRun 514/\$UDPServerRun 514/g" /etc/rsyslog.conf
echo '$UDPServerAddress 127.0.0.1' >> /etc/rsyslog.conf
echo 'local2.*    /var/log/haproxy.log' > /etc/rsyslog.d/haproxy.conf
systemctl restart rsyslog

echo 'net.ipv4.ip_nonlocal_bind = 1' >> /etc/sysctl.conf

sudo systemctl stop haproxy


## PHP

yum install -y http://rpms.remirepo.net/enterprise/remi-release-7.rpm
yum install -y yum-utils
yum-config-manager --enable remi-php72

yum install php php-common php-xmlrpc php-soap php-mysql php-dom php-mbstring php-gd php-ldap php-pdo php-json php-xml php-zip php-curl php-mcrypt php-pear php-intl setroubleshoot-server -y

## NFS

### NFS SERVER
yum install nfs-utils -y

sudo mkdir /var/moodledata
sudo chown -R apache:apache /var/moodledata
sudo chmod -R 755 /var/moodledata

systemctl enable rpcbind
systemctl enable nfs-server
systemctl enable nfs-lock
systemctl enable nfs-idmap
systemctl start rpcbind
systemctl start nfs-server
systemctl start nfs-lock
systemctl start nfs-idmap

echo '/var/moodledata  10.10.11.94(rw,sync,no_root_squash,no_all_squash) 10.10.11.95(rw,no_root_squash) 10.10.11.96(rw,no_root_squash)' > /etc/exports

systemctl restart nfs-server

### NFS CLIENT

mkdir -p /var/moodledata
mount -t nfs 10.10.11.94:/var/moodledata /var/moodledata


Kiểm tra
df -kh

echo '10.10.11.94:/var/moodledata /var/moodledata   nfs defaults 0 0' >> /etc/fstab

## Moodle

cd
yum install -y wget

wget https://download.moodle.org/download.php/direct/stable36/moodle-latest-36.tgz

sudo tar -zxvf moodle-latest-36.tgz -C /var/www/html

sudo chown -R root:root /var/www/html/moodle


### virual host
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

### Install Moodle

sudo /usr/bin/php /var/www/html/moodle/admin/cli/install.php --chmod=2777 --lang=en --wwwroot=http://10.10.10.94 --dataroot=/var/moodledata \
--dbtype=mariadb --dbhost=10.10.10.94 --dbname=moodle --dbuser=moodleuser --dbpass=Cloud365a@123 \
--fullname=MoodleNH --shortname=MNH \
--adminuser=admin --adminpass=Cloud365a@123 --adminemail=thanhnb@nhanhoa.com.vn \
--agree-license


sudo /usr/bin/php /var/www/html/moodle/admin/cli/install.php --chmod=2777 --lang=en --wwwroot=http://10.10.10.95 --dataroot=/var/moodledata \
--dbtype=mariadb --dbhost=10.10.10.94 --dbname=moodle --dbuser=moodleuser --dbpass=Cloud365a@123 \
--fullname=MoodleNH --shortname=MNH \
--adminuser=admin --adminpass=Cloud365a@123 --adminemail=thanhnb@nhanhoa.com.vn \
--agree-license --skip-database

sudo /usr/bin/php /var/www/html/moodle/admin/cli/install.php --chmod=2777 --lang=en --wwwroot=http://10.10.10.96 --dataroot=/var/moodledata \
--dbtype=mariadb --dbhost=10.10.10.94 --dbname=moodle --dbuser=moodleuser --dbpass=Cloud365a@123 \
--fullname=MoodleNH --shortname=MNH \
--adminuser=admin --adminpass=Cloud365a@123 --adminemail=thanhnb@nhanhoa.com.vn \
--agree-license --skip-database


Modify permissions to /var/www/html/config.php

sudo chmod o+r /var/www/html/moodle/config.php


Setup a cron job

sudo crontab -u apache -e
* * * * *    /usr/bin/php /var/www/html/moodle/admin/cli/cron.php >/dev/null

sudo systemctl restart httpd.service


> http://10.10.10.94/ (admin/Cloud365a@123)


## Pacemaker

yum -y install pacemaker pcs

systemctl start pcsd 
systemctl enable pcsd


echo "Cloud365a@123" | passwd --stdin hacluster

pcs cluster auth moodle01 moodle02 moodle03


pcs cluster setup --name ha_cluster moodle01 moodle02 moodle03



pcs cluster start --all
pcs cluster enable --all

pcs property set stonith-enabled=false
pcs property set no-quorum-policy=ignore
pcs property set default-resource-stickiness="INFINITY"


pcs resource create Virtual_IP ocf:heartbeat:IPaddr2 ip=10.10.10.97 cidr_netmask=24 op monitor interval=30s

pcs resource create Loadbalancer_HaProxy systemd:haproxy op monitor timeout="5s" interval="5s"


pcs resource create Web_Cluster systemd:httpd op monitor timeout="5s" interval="5s"
pcs resource clone Web_Cluster globally-unique=true


pcs constraint order Virtual_IP then Web_Cluster-clone
pcs constraint order Virtual_IP then Loadbalancer_HaProxy

pcs constraint colocation add Virtual_IP Loadbalancer_HaProxy INFINITY



## Chuyển config Moodle về VIP

sed -i "s/10.10.10.94/10.10.10.97/g" /var/www/html/moodle/config.php
sed -i "s/10.10.10.95/10.10.10.97/g" /var/www/html/moodle/config.php
sed -i "s/10.10.10.96/10.10.10.97/g" /var/www/html/moodle/config.php

sudo systemctl restart httpd.service
