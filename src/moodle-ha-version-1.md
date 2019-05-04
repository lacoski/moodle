# Tài liệu hướng dẫn triển khai Moodle HA
---
## Mục lục

Tài liệu gồm 7 phần:
- Phần 1: Cài đặt MariaDB Galera
- Phần 2: Triển khai HAProxy
- Phần 3: Triển khai NFS
- Phần 4: Triển khai Apache
- Phần 5: Cài đặt PHP
- Phần 6: Triển khai Moodle
- Phần 7: Triển khai Pacemaker

## Mô hình

### Mô hình triển khai

![](/images/moodle-ha-version1/pic1.png)

### Cấu hình

```
HOSTNAME    moodle01
CPU         4 core
RAM         4 GB
Disk        vda: os
Network     eth0: network access (10.10.10.94)
Network     eth1: network internal (10.10.11.94)

HOSTNAME    moodle02
CPU         4 core
RAM         4 GB
Disk        vda: os
Network     eth0: network access (10.10.10.95)
Network     eth1: network internal (10.10.11.95)

HOSTNAME    moodle03
CPU         4 core
RAM         4 GB
Disk        vda: os
Network     eth0: network access (10.10.10.96)
Network     eth1: network internal (10.10.11.96)
```

## Chuẩn bị

### Tại moodle01

Cấu hình Hostname
```
hostnamectl set-hostname moodle01
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
init 6
```

### Tạo moodle02

Cấu hình Hostname
```
hostnamectl set-hostname moodle02
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
init 6
```

### Tạo moodle03

Cấu hình Hostname
```
hostnamectl set-hostname moodle03
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
init 6
```

## Cài đặt

### Phần 1: Cài đặt MariaDB Galera

#### Mô hình kiến trúc

![](/images/moodle-ha-version1/pic4.png)

#### Bước 1: Cài đặt MariaDB 10.2

> Thực hiện trên tất cả các node

Thêm repo
```
echo '[mariadb]
name = MariaDB
baseurl = http://yum.mariadb.org/10.2/centos7-amd64
gpgkey=https://yum.mariadb.org/RPM-GPG-KEY-MariaDB
gpgcheck=1' >> /etc/yum.repos.d/MariaDB.repo
yum -y update
```

Cài đặt gói MariaDB
```
yum install -y mariadb mariadb-server

yum install -y galera rsync

systemctl stop mariadb
systemctl disable mariadb
```

Cấu hình Log

```
cp /etc/my.cnf /etc/my.cnf.org

cat > /etc/my.cnf << EOF
[mysqld]
slow_query_log                  = 1
slow_query_log_file             = /var/log/mariadb/slow.log
long_query_time                 = 5
log_error                       = /var/log/mariadb/error.log
general_log_file                = /var/log/mariadb/mysql.log
general_log                     = 1

[client-server]
!includedir /etc/my.cnf.d
EOF

mkdir -p /var/log/mariadb/
chown -R mysql. /var/log/mariadb/
```
#### Bước 2: Cấu hình Galera

__Tại moodle01__

```
cp /etc/my.cnf.d/server.cnf /etc/my.cnf.d/server.cnf.bak

cat > /etc/my.cnf.d/server.cnf << EOF
[server]
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
EOF
```

__Tại moodle02__

```
cp /etc/my.cnf.d/server.cnf /etc/my.cnf.d/server.cnf.bak

cat > /etc/my.cnf.d/server.cnf << EOF
[server]
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
EOF
```

__Tại moodle03__

```
cp /etc/my.cnf.d/server.cnf /etc/my.cnf.d/server.cnf.bak

cat > /etc/my.cnf.d/server.cnf << EOF
[server]
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
EOF
```

#### Bước 3: Khởi động Cluster

__Thực hiện trên moodle01__

```
galera_new_cluster
systemctl enable mariadb
```

__Thực hiện trên moodle02 và moodle03__

```
systemctl start mariadb
systemctl enable mariadb
```

#### Bước 4: Kiểm tra cluster

Kiểm tra số node thuộc Cluster
```
mysql -u root -e "SHOW STATUS LIKE 'wsrep_cluster_size'"
```

Kết quả
```
[root@moodle01 ~]# mysql -u root -e "SHOW STATUS LIKE 'wsrep_cluster_size'"
+--------------------+-------+
| Variable_name      | Value |
+--------------------+-------+
| wsrep_cluster_size | 3     |
+--------------------+-------+
```

Kiểm tra các thành viên thuộc cluster
```
[root@moodle01 ~]# mysql -u root -e "SHOW STATUS LIKE 'wsrep_incoming_addresses'"
+--------------------------+----------------------------------------------------+
| Variable_name            | Value                                              |
+--------------------------+----------------------------------------------------+
| wsrep_incoming_addresses | 10.10.10.96:3306,10.10.10.94:3306,10.10.10.95:3306 |
+--------------------------+----------------------------------------------------+
```

### Phần 2: Triển khai HAProxy

#### Bước 1: Cài đặt HAProxy 1.8
```
sudo yum install wget socat -y
wget http://cbs.centos.org/kojifiles/packages/haproxy/1.8.1/5.el7/x86_64/haproxy18-1.8.1-5.el7.x86_64.rpm 
yum install haproxy18-1.8.1-5.el7.x86_64.rpm -y
```

#### Bước 2: Cấu hình HAProxy

```
cp /etc/haproxy/haproxy.cfg /etc/haproxy/haproxy.cfg.bak

cat > /etc/haproxy/haproxy.cfg << EOF
global
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
    stats auth admin:Cloud365a@123

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
    server moodle03 10.10.10.96:80  weight 1 check cookie s3
EOF
```

#### Bước 3: Cầu hình Plugin check Galera

```
yum install git xinetd -y
git clone https://github.com/lacoski/percona-clustercheck.git
cp percona-clustercheck/clustercheck /usr/local/bin


cat << EOF >> /etc/xinetd.d/mysqlchk
service mysqlchk
{
      disable = no
      flags = REUSE
      socket_type = stream
      port = 9200
      wait = no
      user = nobody
      server = /usr/local/bin/clustercheck
      log_on_failure += USERID
      only_from = 0.0.0.0/0
      per_source = UNLIMITED
}
EOF

echo 'mysqlchk 9200/tcp # MySQL check' >> /etc/services

mysql -uroot

GRANT PROCESS ON *.* TO 'clustercheckuser'@'localhost' IDENTIFIED BY 'clustercheckpassword!';
FLUSH PRIVILEGES;
exit

systemctl restart xinetd
systemctl enable xinetd
```

#### Bước 4: Cấu hình Log HAProxy
```
sed -i "s/#\$ModLoad imudp/\$ModLoad imudp/g" /etc/rsyslog.conf
sed -i "s/#\$UDPServerRun 514/\$UDPServerRun 514/g" /etc/rsyslog.conf
echo '$UDPServerAddress 127.0.0.1' >> /etc/rsyslog.conf
echo 'local2.*    /var/log/haproxy.log' > /etc/rsyslog.d/haproxy.conf
systemctl restart rsyslog

echo 'net.ipv4.ip_nonlocal_bind = 1' >> /etc/sysctl.conf
```

#### Bước 5: Tắt dịch vụ HAProxy, dịch vụ HAProxy sẽ được quản lý bởi Pacemaker
```
sudo systemctl stop haproxy
```

Lưu ý:
- Khi chưa triển khai dịch vụ Pacemaker, không thể chạy được HAproxy

### Phần 3: Triển khai Apache

#### Tại moodle01
Cài đặt
```
sudo yum install httpd -y
```

Cấu hình HTTP
```
cat /etc/httpd/conf/httpd.conf | grep 'Listen 80'
sed -i "s/Listen 80/Listen 10.10.10.94:80/g" /etc/httpd/conf/httpd.conf
cat /etc/httpd/conf/httpd.conf | grep 'Listen'

sudo sed -i 's/^/#&/g' /etc/httpd/conf.d/welcome.conf
sudo sed -i "s/Options Indexes FollowSymLinks/Options FollowSymLinks/" /etc/httpd/conf/httpd.conf
```

Tắt dịch vụ HTTP, dịch vụ HTTP sẽ được quản lý bởi Pacemaker
```
sudo systemctl stop httpd.service
sudo systemctl disable httpd.service
```

#### Tại moodle02

Cài đặt
```
sudo yum install httpd -y
```

Cấu hình HTTP
```
cat /etc/httpd/conf/httpd.conf | grep 'Listen 80'
sed -i "s/Listen 80/Listen 10.10.10.95:80/g" /etc/httpd/conf/httpd.conf
cat /etc/httpd/conf/httpd.conf | grep 'Listen'

sudo sed -i 's/^/#&/g' /etc/httpd/conf.d/welcome.conf
sudo sed -i "s/Options Indexes FollowSymLinks/Options FollowSymLinks/" /etc/httpd/conf/httpd.conf
```

Tắt dịch vụ HTTP, dịch vụ HTTP sẽ được quản lý bởi Pacemaker
```
sudo systemctl stop httpd.service
sudo systemctl disable httpd.service
```

#### Tại moodle03
Cài đặt
```
sudo yum install httpd -y
```

Cấu hình HTTP
```
cat /etc/httpd/conf/httpd.conf | grep 'Listen 80'
sed -i "s/Listen 80/Listen 10.10.10.96:80/g" /etc/httpd/conf/httpd.conf
cat /etc/httpd/conf/httpd.conf | grep 'Listen'

sudo sed -i 's/^/#&/g' /etc/httpd/conf.d/welcome.conf
sudo sed -i "s/Options Indexes FollowSymLinks/Options FollowSymLinks/" /etc/httpd/conf/httpd.conf
```

Tắt dịch vụ HTTP, dịch vụ HTTP sẽ được quản lý bởi Pacemaker
```
sudo systemctl stop httpd.service
sudo systemctl disable httpd.service
```

### Phần 4: Triển khai NFS

#### Mô hình kiến trúc

![](/images/moodle-ha-version1/pic3.png)

Lưu ý:
- Trong mô hình, mình sẽ sử dụng giải pháp NFS làm share storage giữa các node.
- Mình sẽ lựa chọn moodle01 làm node NFS Server, moodle02 vầ moodle03 làm node NFS Client
- Có thể lựa chọn 1 node khác để làm node NFS Server

#### Cấu hình NFS SERVER

> Thực hiện trên moodle01

Cài đặt
```
yum install nfs-utils -y
```

Tạo thư mục chia sẻ
```
sudo mkdir -p /var/moodledata
sudo chown -R apache:apache /var/moodledata
sudo chmod -R 755 /var/moodledata
```

Khởi tạo dịch vụ
```
systemctl enable rpcbind
systemctl enable nfs-server
systemctl enable nfs-lock
systemctl enable nfs-idmap
systemctl start rpcbind
systemctl start nfs-server
systemctl start nfs-lock
systemctl start nfs-idmap
```

Cấu hình chia sẻ NFS
```
echo '/var/moodledata  10.10.11.94(rw,sync,no_root_squash,no_all_squash) 10.10.11.95(rw,no_root_squash) 10.10.11.96(rw,no_root_squash)' > /etc/exports
```

Khởi động lại NFS
```
systemctl restart nfs-server
```

#### Cấu hình NFS CLIENT

> Thực hiện trên moodle02 và moodle03

Cài đặt
```
yum install nfs-utils -y
```

Tạo thư mục mount point
```
mkdir -p /var/moodledata
```

Mount NFS
```
mount -t nfs 10.10.11.94:/var/moodledata /var/moodledata
```

Kiểm tra
```
df -kh
```

Kết quả
```
[root@moodle02 ~]# df -kh
Filesystem                   Size  Used Avail Use% Mounted on
/dev/mapper/centos-root       50G  1.9G   49G   4% /
devtmpfs                     2.9G     0  2.9G   0% /dev
tmpfs                        2.9G     0  2.9G   0% /dev/shm
tmpfs                        2.9G  8.6M  2.9G   1% /run
tmpfs                        2.9G     0  2.9G   0% /sys/fs/cgroup
/dev/vda1                   1014M  219M  796M  22% /boot
/dev/mapper/centos-home       27G   33M   27G   1% /home
tmpfs                        581M     0  581M   0% /run/user/0
10.10.11.94:/var/moodledata   50G  1.9G   49G   4% /var/moodledata
```

Tự động mount khi khởi động OS
```
echo '10.10.11.94:/var/moodledata /var/moodledata   nfs defaults 0 0' >> /etc/fstab
```


### Phần 5: Cài đặt PHP

> Thực hiện trên tất cả các node

Lưu ý:
- Mình sẽ sử dụng phiên bản PHP 7.2

Cài đựat PHP và các gói hỗ trợ
```
yum install -y http://rpms.remirepo.net/enterprise/remi-release-7.rpm
yum install -y yum-utils
yum-config-manager --enable remi-php72

yum install php php-common php-xmlrpc php-soap php-mysql php-dom php-mbstring php-gd php-ldap php-pdo php-json php-xml php-zip php-curl php-mcrypt php-pear php-intl setroubleshoot-server -y
```

### Phần 6: Triển khai Moodle

#### Tại moodle01

Tạo Database và User
```
mysql -u root

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
```

Tải source
```
cd
yum install -y wget

wget https://download.moodle.org/download.php/direct/stable36/moodle-latest-36.tgz
```

Giải nén và phân quyền source
```
sudo tar -zxvf moodle-latest-36.tgz -C /var/www/html
sudo chown -R root:root /var/www/html/moodle
```

Cấu hình virtual host
```
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
```

Cài đặt Moodle
```
sudo /usr/bin/php /var/www/html/moodle/admin/cli/install.php --chmod=2777 --lang=en --wwwroot=http://10.10.10.94 --dataroot=/var/moodledata \
--dbtype=mariadb --dbhost=10.10.10.94 --dbname=moodle --dbuser=moodleuser --dbpass=Cloud365a@123 \
--fullname=MoodleNH --shortname=MNH \
--adminuser=admin --adminpass=Cloud365a@123 --adminemail=thanhnb@nhanhoa.com.vn \
--agree-license
```

Kết quả
```
[root@moodle01 ~]# sudo /usr/bin/php /var/www/html/moodle/admin/cli/install.php --chmod=2777 --lang=en --wwwroot=http://10.10.10.94 --dataroot=/var/moodledata \
> --dbtype=mariadb --dbhost=10.10.10.94 --dbname=moodle --dbuser=moodleuser --dbpass=Cloud365a@123 \
> --fullname=MoodleNH --shortname=MNH \
> --adminuser=admin --adminpass=Cloud365a@123 --adminemail=thanhnb@nhanhoa.com.vn \
> --agree-license
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
: 
-------------------------------------------------------------------------------
== Data directories permission ==
type value, press Enter to use default value (2777)
: 
-------------------------------------------------------------------------------
== Web address ==
type value, press Enter to use default value (http://10.10.10.94)
: 
-------------------------------------------------------------------------------
== Data directory ==
type value, press Enter to use default value (/var/moodledata)
: 
-------------------------------------------------------------------------------
== Choose database driver ==
 mysqli 
 mariadb 
type value, press Enter to use default value (mariadb)
: 
-------------------------------------------------------------------------------
== Database host ==
type value, press Enter to use default value (10.10.10.94)
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
type value, press Enter to use default value (moodleuser)
: 
-------------------------------------------------------------------------------
== Database password ==
type value, press Enter to use default value (Cloud365a@123)
: 
-------------------------------------------------------------------------------
== Full site name ==
type value, press Enter to use default value (MoodleNH)
: 
-------------------------------------------------------------------------------
== Short name for site (eg single word) ==
type value, press Enter to use default value (MNH)
: 
-------------------------------------------------------------------------------
== Admin account username ==
type value, press Enter to use default value (admin)
: 
-------------------------------------------------------------------------------
== New admin user password ==
type value
: Cloud365a@123
-------------------------------------------------------------------------------
== New admin user email address ==
type value, press Enter to use default value (thanhnb@nhanhoa.com.vn)
: 
-------------------------------------------------------------------------------
== Upgrade key (leave empty to not set it) ==
type value
: 
-------------------------------------------------------------------------------
== Setting up database ==
-->System

...
Installation completed successfully.
```

Chính sửa quyền trên file `/var/www/html/config.php`
```
sudo chmod o+r /var/www/html/moodle/config.php
```

Thiết lập crontab
```
sudo crontab -u apache -e
```
- Nội dung
  ```
  * * * * *    /usr/bin/php /var/www/html/moodle/admin/cli/cron.php >/dev/null
  ```

Khởi động lại httpd
```
sudo systemctl restart httpd.service
```
#### Tại moodle02
Giải nén và phân quyền source
```
sudo tar -zxvf moodle-latest-36.tgz -C /var/www/html
sudo chown -R root:root /var/www/html/moodle
```

Cấu hình virtual host
```
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
```

Cài đặt Moodle
```
sudo /usr/bin/php /var/www/html/moodle/admin/cli/install.php --chmod=2777 --lang=en --wwwroot=http://10.10.10.95 --dataroot=/var/moodledata \
--dbtype=mariadb --dbhost=10.10.10.94 --dbname=moodle --dbuser=moodleuser --dbpass=Cloud365a@123 \
--fullname=MoodleNH --shortname=MNH \
--adminuser=admin --adminpass=Cloud365a@123 --adminemail=thanhnb@nhanhoa.com.vn \
--agree-license --skip-database
```

Chính sửa quyền trên file `/var/www/html/config.php`
```
sudo chmod o+r /var/www/html/moodle/config.php
```

Thiết lập crontab
```
sudo crontab -u apache -e
```
- Nội dung
  ```
  * * * * *    /usr/bin/php /var/www/html/moodle/admin/cli/cron.php >/dev/null
  ```

Khởi động lại httpd
```
sudo systemctl restart httpd.service
```

#### Tại moodle03
Giải nén và phân quyền source
```
sudo tar -zxvf moodle-latest-36.tgz -C /var/www/html
sudo chown -R root:root /var/www/html/moodle
```

Cấu hình virtual host
```
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
```

Cài đặt Moodle
```
sudo /usr/bin/php /var/www/html/moodle/admin/cli/install.php --chmod=2777 --lang=en --wwwroot=http://10.10.10.96 --dataroot=/var/moodledata \
--dbtype=mariadb --dbhost=10.10.10.94 --dbname=moodle --dbuser=moodleuser --dbpass=Cloud365a@123 \
--fullname=MoodleNH --shortname=MNH \
--adminuser=admin --adminpass=Cloud365a@123 --adminemail=thanhnb@nhanhoa.com.vn \
--agree-license --skip-database
```

Chính sửa quyền trên file `/var/www/html/config.php`
```
sudo chmod o+r /var/www/html/moodle/config.php
```

Thiết lập crontab
```
sudo crontab -u apache -e
```
- Nội dung
  ```
  * * * * *    /usr/bin/php /var/www/html/moodle/admin/cli/cron.php >/dev/null
  ```

Khởi động lại httpd
```
sudo systemctl restart httpd.service
```


### Phần 7: Triển khai Pacemaker

#### Mô hình kiến trúc

![](/images/moodle-ha-version1/pic2.png)

#### Bước 1: Cài đặt

> Thực hiện trên tất cả các node

```
yum -y install pacemaker pcs

systemctl start pcsd 
systemctl enable pcsd
```

#### Bước 2: Thiết lập mật khẩu user hacluster

> Thực hiện trên tất cả các node

```
echo "Cloud365a@123" | passwd --stdin hacluster
```

Lưu ý:
- Mật khẩu user hacluster phải đồng bộ trên tất cả các node


#### Bước 3: Tạo Cluster

> Cấu hình trên moodle01

Chứng thực cluster (Chỉ thực thiện trên cấu hình trên một node duy nhất), nhập chính xác tài khoản user hacluster
```
pcs cluster auth moodle01 moodle02 moodle03
```

Kết quả
```
[root@moodle01 ~]# pcs cluster auth moodle01 moodle02 moodle03
Username: hacluster
Password: Cloud365a@123
moodle01: Authorized
moodle02: Authorized
moodle03: Authorized
```

Khởi tạo cấu hình ban đầu của Cluster
```
pcs cluster setup --name ha_cluster moodle01 moodle02 moodle03
```

Kết quả
```
[root@moodle01 ~]# pcs cluster setup --name ha_cluster moodle01 moodle02 moodle03

Destroying cluster on nodes: moodle01, moodle02, moodle03...
moodle03: Stopping Cluster (pacemaker)...
moodle01: Stopping Cluster (pacemaker)...
moodle02: Stopping Cluster (pacemaker)...
moodle03: Successfully destroyed cluster
moodle02: Successfully destroyed cluster
moodle01: Successfully destroyed cluster

Sending 'pacemaker_remote authkey' to 'moodle01', 'moodle02', 'moodle03'
moodle01: successful distribution of the file 'pacemaker_remote authkey'
moodle03: successful distribution of the file 'pacemaker_remote authkey'
moodle02: successful distribution of the file 'pacemaker_remote authkey'
Sending cluster config files to the nodes...
moodle01: Succeeded
moodle02: Succeeded
moodle03: Succeeded

Synchronizing pcsd certificates on nodes moodle01, moodle02, moodle03...
moodle01: Success
moodle02: Success
moodle03: Success
Restarting pcsd on the nodes in order to reload the certificates...
moodle01: Success
moodle02: Success
moodle03: Success
```

Khởi động cluster
```
pcs cluster start --all
```

Kết quả
```
[root@moodle01 ~]# pcs cluster start --all
moodle01: Starting Cluster (corosync)...
moodle02: Starting Cluster (corosync)...
moodle03: Starting Cluster (corosync)...
moodle01: Starting Cluster (pacemaker)...
moodle03: Starting Cluster (pacemaker)...
moodle02: Starting Cluster (pacemaker)...
```

Cho phép cluster khởi động cùng OS
```
pcs cluster enable --all
```

Kết quả
```
[root@moodle01 ~]# pcs cluster enable --all
moodle01: Cluster Enabled
moodle02: Cluster Enabled
moodle03: Cluster Enabled
```

#### Bước 4: Thiết lập Cluster

Cấu hình cơ chế bảo vệ trạng thái Cluster, tùy nhiên ta tạm bỏ qua cấu hình này
```
pcs property set stonith-enabled=false
```

Cho phép Cluster chạy kể cả khi mất quorum
```
pcs property set no-quorum-policy=ignore
```

Hạn chế Resource trong cluster chuyển node sau khi Cluster khởi động lại (1 số resource thay đổi node có thể dẫn tới dịch vụ bị down time như IP VIP của database)
```
pcs property set default-resource-stickiness="INFINITY"
```

Kiểm tra thiết lập cluster
```
pcs property list
```

Kết quả
```
[root@moodle01 ~]# pcs property list
Cluster Properties:
 cluster-infrastructure: corosync
 cluster-name: ha_cluster
 dc-version: 1.1.19-8.el7_6.4-c3c624ea3d
 default-resource-stickiness: INFINITY
 have-watchdog: false
 no-quorum-policy: ignore
 stonith-enabled: false
```

#### Bước 5: Tạo Resource

Tạo resource IP VIP Cluster
```
pcs resource create Virtual_IP ocf:heartbeat:IPaddr2 ip=10.10.10.97 cidr_netmask=24 op monitor interval=30s
```

Tạo resource HAProxy, cấu hình dạng (Active/Passive)
```
pcs resource create Loadbalancer_HaProxy systemd:haproxy op monitor timeout="5s" interval="5s"
```

Tạo Resource quản lý Apache (Active/Active)
```
pcs resource create Web_Cluster systemd:httpd op monitor timeout="5s" interval="5s"
pcs resource clone Web_Cluster globally-unique=true
```

Tạo ràng buộc Resource Virtal_IP chạy phải khởi động trước Resource Web_Cluster
```
pcs constraint order Virtual_IP then Web_Cluster-clone
```

Ràng buộc Resource Virtal_IP phải chạy cùng node với resource Loadbalancer_HaProxy
```
pcs constraint order Virtual_IP then Loadbalancer_HaProxy
```

Ràng buộc Resource Virtal_IP phải chạy cùng node với resource Loadbalancer_HaProxy
```
pcs constraint colocation add Virtual_IP Loadbalancer_HaProxy INFINITY
```

#### Bước 6: Chỉnh sửa lại cấu hình Moodle trên Pacemaker

> Thực hiện trên tất cả các node

Cấu hình Moodle
```
sed -i "s/10.10.10.94/10.10.10.97/g" /var/www/html/moodle/config.php
sed -i "s/10.10.10.95/10.10.10.97/g" /var/www/html/moodle/config.php
sed -i "s/10.10.10.96/10.10.10.97/g" /var/www/html/moodle/config.php
```

Khởi động lại HTTP
```
sudo systemctl restart httpd.service
```


## Kiểm tra

### Kiểm tra HAProxy Stats Page

![](/images/moodle-ha-version1/pic5.png)

### Truy cập Moodle HA

![](/images/moodle-ha-version1/pic6.png)

### Tắt nóng 1 node

Lưu ý:
- Mô hình chưa HA được node NFS Server (moodle01) nên chỉ có thể xảy lỗi moodle02 và moodle03
- Nếu có lỗi xảy ra tại NFS Server, hệ thống moodle sẽ xảy ra vấn đề

Tắt Node moodle02
```
[root@moodle02 ~]# init 0
```

Kiểm tra HAProxy
![](/images/moodle-ha-version1/pic7.png)

Kiểm tra trạng thái Cluster
```
[root@moodle01 ~]# pcs status
Cluster name: ha_cluster
Stack: corosync
Current DC: moodle03 (version 1.1.19-8.el7_6.4-c3c624ea3d) - partition with quorum
Last updated: Wed Sep 18 02:46:28 2019
Last change: Wed Sep 18 02:38:58 2019 by root via cibadmin on moodle01

3 nodes configured
5 resources configured

Online: [ moodle01 moodle03 ]
OFFLINE: [ moodle02 ]

Full list of resources:

 Virtual_IP	(ocf::heartbeat:IPaddr2):	Started moodle01
 Loadbalancer_HaProxy	(systemd:haproxy):	Started moodle01
 Clone Set: Web_Cluster-clone [Web_Cluster] (unique)
     Web_Cluster:0	(systemd:httpd):	Stopped
     Web_Cluster:1	(systemd:httpd):	Started moodle03
     Web_Cluster:2	(systemd:httpd):	Started moodle01

Failed Actions:
* Loadbalancer_HaProxy_start_0 on moodle03 'not running' (7): call=10, status=complete, exitreason='',
    last-rc-change='Wed Sep 18 02:38:37 2019', queued=0ms, exec=2072ms


Daemon Status:
  corosync: active/enabled
  pacemaker: active/enabled
  pcsd: active/enabled
```

- Trên HAProxy và Pacemaker đã báo moodle02 down, thử truy cập lại vào moodle, vẫn có thể truy cập bình thường

![](/images/moodle-ha-version1/pic8.png)