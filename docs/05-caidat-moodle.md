# Cài đặt Moodle Cluster
---
## Mô hình

### Quy hoạch
![](/images/05-caidat-moodle/ip_planning_moodle.png)

### Mô hình triển khai
![](/images/05-caidat-moodle/mohinh_moodle_only.png)

### Lưu ý:
- Node Moodle01 có hostname `tlumoodle01`
- Node Moodle02 có hostname `tlumoodle02`
- Node Moodle03 có hostname `tlumoodle03`
- Node Moodle01 và Moodle02, Moodle03 sử dụng OS CENTOS 7

### Yêu cầu
- Cần chuấn bị trước Database và Storage qua docs:

[Triển khai dịch vụ Database](/docs/04-caidat-database.md)

[Triển khai dịch vụ Storage và Backup](/docs/03-caidat-storage.md)

## Chuẩn bị

### Trên node `Moodle01`
> Hostname `tlumoodle01`

Cập nhật hệ điều hành
```
yum install epel-release -y
yum update -y
```

Thiết lập Hostname
```
hostnamectl set-hostname tlumoodle01
```

Cấu hình Network
```
echo "Setup IP eth0"
nmcli c modify eth0 ipv4.addresses 192.168.20.31/24
nmcli c modify eth0 ipv4.gateway 192.168.20.1
nmcli c modify eth0 ipv4.dns 8.8.8.8
nmcli c modify eth0 ipv4.method manual
nmcli con mod eth0 connection.autoconnect yes


echo "Setup IP eth1"
nmcli c modify eth1 ipv4.addresses 192.168.21.31/24
nmcli c modify eth1 ipv4.method manual
nmcli con mod eth1 connection.autoconnect yes
```

Tắt FirewallD và SELinux
```
sed -i 's/SELINUX=enforcing/SELINUX=disabled/g' /etc/sysconfig/selinux
sed -i 's/SELINUX=enforcing/SELINUX=disabled/g' /etc/selinux/config
systemctl stop firewalld
systemctl disable firewalld
```

Cấu hình CMD Log
```
curl -Lso- https://raw.githubusercontent.com/nhanhoadocs/scripts/master/Utilities/cmdlog.sh | bash
```

Cấu hình đồng bộ thời gian
```
timedatectl set-timezone Asia/Ho_Chi_Minh

yum -y install chrony
VIP_MGNT_IP='192.168.20.54'
sed -i '/server/d' /etc/chrony.conf
echo "server $VIP_MGNT_IP iburst" >> /etc/chrony.conf
systemctl enable chronyd.service
systemctl restart chronyd.service
chronyc sources
```

Lưu ý:
- Cấu hình đồng bộ thời gian với Node NTP Server

Cấu hình Host File
```
echo "192.168.20.31 tlumoodle01" >> /etc/hosts
echo "192.168.20.32 tlumoodle02" >> /etc/hosts
echo "192.168.20.33 tlumoodle03" >> /etc/hosts

echo "192.168.20.41 tlustorage01" >> /etc/hosts
echo "192.168.20.42 tlustorage02" >> /etc/hosts

echo "192.168.20.44 tludatabase01" >> /etc/hosts
echo "192.168.20.45 tludatabase02" >> /etc/hosts
```

Khởi động lại
```
init 6
```

### Trên node `Moodle02`
> Hostname `tlumoodle02`

Cập nhật hệ điều hành
```
yum install epel-release -y
yum update -y
```

Thiết lập Hostname
```
hostnamectl set-hostname tlumoodle02
```

Cấu hình Network
```
echo "Setup IP eth0"
nmcli c modify eth0 ipv4.addresses 192.168.20.32/24
nmcli c modify eth0 ipv4.gateway 192.168.20.1
nmcli c modify eth0 ipv4.dns 8.8.8.8
nmcli c modify eth0 ipv4.method manual
nmcli con mod eth0 connection.autoconnect yes


echo "Setup IP eth1"
nmcli c modify eth1 ipv4.addresses 192.168.21.32/24
nmcli c modify eth1 ipv4.method manual
nmcli con mod eth1 connection.autoconnect yes
```

Tắt FirewallD và SELinux
```
sed -i 's/SELINUX=enforcing/SELINUX=disabled/g' /etc/sysconfig/selinux
sed -i 's/SELINUX=enforcing/SELINUX=disabled/g' /etc/selinux/config
systemctl stop firewalld
systemctl disable firewalld
```

Cấu hình CMD Log
```
curl -Lso- https://raw.githubusercontent.com/nhanhoadocs/scripts/master/Utilities/cmdlog.sh | bash
```

Cấu hình đồng bộ thời gian
```
timedatectl set-timezone Asia/Ho_Chi_Minh

yum -y install chrony
VIP_MGNT_IP='192.168.20.54'
sed -i '/server/d' /etc/chrony.conf
echo "server $VIP_MGNT_IP iburst" >> /etc/chrony.conf
systemctl enable chronyd.service
systemctl restart chronyd.service
chronyc sources
```

Lưu ý:
- Cấu hình đồng bộ thời gian với Node NTP Server

Cấu hình Host File
```
echo "192.168.20.31 tlumoodle01" >> /etc/hosts
echo "192.168.20.32 tlumoodle02" >> /etc/hosts
echo "192.168.20.33 tlumoodle03" >> /etc/hosts

echo "192.168.20.41 tlustorage01" >> /etc/hosts
echo "192.168.20.42 tlustorage02" >> /etc/hosts

echo "192.168.20.44 tludatabase01" >> /etc/hosts
echo "192.168.20.45 tludatabase02" >> /etc/hosts
```

Khởi động lại
```
init 6
```

### Trên node `Moodle03`
> Hostname `tlumoodle03`

Cập nhật hệ điều hành
```
yum install epel-release -y
yum update -y
```

Thiết lập Hostname
```
hostnamectl set-hostname tlumoodle03
```

Cấu hình Network
```
echo "Setup IP eth0"
nmcli c modify eth0 ipv4.addresses 192.168.20.33/24
nmcli c modify eth0 ipv4.gateway 192.168.20.1
nmcli c modify eth0 ipv4.dns 8.8.8.8
nmcli c modify eth0 ipv4.method manual
nmcli con mod eth0 connection.autoconnect yes


echo "Setup IP eth1"
nmcli c modify eth1 ipv4.addresses 192.168.21.33/24
nmcli c modify eth1 ipv4.method manual
nmcli con mod eth1 connection.autoconnect yes
```

Tắt FirewallD và SELinux
```
sed -i 's/SELINUX=enforcing/SELINUX=disabled/g' /etc/sysconfig/selinux
sed -i 's/SELINUX=enforcing/SELINUX=disabled/g' /etc/selinux/config
systemctl stop firewalld
systemctl disable firewalld
```

Cấu hình CMD Log
```
curl -Lso- https://raw.githubusercontent.com/nhanhoadocs/scripts/master/Utilities/cmdlog.sh | bash
```

Cấu hình đồng bộ thời gian
```
timedatectl set-timezone Asia/Ho_Chi_Minh

yum -y install chrony
VIP_MGNT_IP='192.168.20.54'
sed -i '/server/d' /etc/chrony.conf
echo "server $VIP_MGNT_IP iburst" >> /etc/chrony.conf
systemctl enable chronyd.service
systemctl restart chronyd.service
chronyc sources
```

Lưu ý:
- Cấu hình đồng bộ thời gian với Node NTP Server

Cấu hình Host File
```
echo "192.168.20.31 tlumoodle01" >> /etc/hosts
echo "192.168.20.32 tlumoodle02" >> /etc/hosts
echo "192.168.20.33 tlumoodle03" >> /etc/hosts

echo "192.168.20.41 tlustorage01" >> /etc/hosts
echo "192.168.20.42 tlustorage02" >> /etc/hosts

echo "192.168.20.44 tludatabase01" >> /etc/hosts
echo "192.168.20.45 tludatabase02" >> /etc/hosts
```

Khởi động lại
```
init 6
```

## Cài đặt môi trường
> Thực hiện trên cả 3 node Moodle01 (`tlumoodle01`), Moodle02 (`tlumoodle02`), Moodle03 (`tlumoodle03`)


### Mount NFS

Dữ liệu moodle (`moodledata`) sẽ được sử dụng chung, chia sẽ bởi NFS Server. Thư mục `moodledata` trên NFS Server mình sẽ mount vào các node Moodle.

Cài đặt NFS Client
```
yum install nfs-utils -y
```

Mount thư mục chia sẻ
```
mkdir -p /var/moodledata
mount -t nfs 192.168.21.41:/var/moodledata /var/moodledata
```

Thêm mount point vào `/etc/fstab`
```
echo "192.168.21.41:/var/moodledata /var/moodledata   nfs defaults 0 0" >> /etc/fstab
```

Kiểm tra
```
df -Th
```

Kết quả
```
[root@tlumoodle01 ~]# df -Th
Filesystem                      Type      Size  Used Avail Use% Mounted on
/dev/mapper/VolGroup00-LogVol01 xfs       184G  1,2G  183G   1% /
devtmpfs                        devtmpfs   16G     0   16G   0% /dev
tmpfs                           tmpfs      16G     0   16G   0% /dev/shm
tmpfs                           tmpfs      16G  8,7M   16G   1% /run
tmpfs                           tmpfs      16G     0   16G   0% /sys/fs/cgroup
/dev/vda1                       ext4      488M  132M  321M  30% /boot
tmpfs                           tmpfs     3,2G     0  3,2G   0% /run/user/0
192.168.21.41:/var/moodledata   nfs4      985G   76M  935G   1% /var/moodledata
```


### Cài đặt HTTP

```
sudo yum install httpd -y
sudo sed -i 's/^/#&/g' /etc/httpd/conf.d/welcome.conf
sudo sed -i "s/Options Indexes FollowSymLinks/Options FollowSymLinks/" /etc/httpd/conf/httpd.conf

sudo systemctl start httpd.service
sudo systemctl enable httpd.service
```

### Cài đặt PHP

```
yum install -y http://rpms.remirepo.net/enterprise/remi-release-7.rpm
yum install -y yum-utils
yum-config-manager --enable remi-php72

yum install php php-common php-xmlrpc php-soap \
 php-mysql php-dom php-mbstring php-gd \
 php-ldap php-pdo php-json php-xml php-zip \
 php-curl php-mcrypt php-pear \
 php-intl setroubleshoot-server -y
```

### Triển khai HAProxy

Cài đặt HAproxy 1.8
```
sudo yum install wget socat -y
wget http://cbs.centos.org/kojifiles/packages/haproxy/1.8.1/5.el7/x86_64/haproxy18-1.8.1-5.el7.x86_64.rpm 
yum install haproxy18-1.8.1-5.el7.x86_64.rpm -y
```

Cấu hình HAProrxy
```
cp /etc/haproxy/haproxy.cfg /etc/haproxy/haproxy.cfg.org

cat > /etc/haproxy/haproxy.cfg << EOF
global
    log         127.0.0.1 local2
    chroot      /var/lib/haproxy
    pidfile     /var/run/haproxy.pid
    maxconn     15000
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
    maxconn                 15000

listen stats
    bind :8080
    mode http
    stats enable
    stats uri /stats
    stats realm HAProxy\ Statistics
    stats admin if TRUE
    stats auth admin:Cloud3652019a@

listen web-backend
    bind 192.168.20.30:80
    balance leastconn
    cookie SERVERID insert indirect nocache
    mode  http
    option  forwardfor
    option  httpchk GET / HTTP/1.0
    option  httpclose
    option  httplog
    timeout  client 3h
    timeout  server 3h
    server moodle01 192.168.20.31:80  weight 1 check cookie s1 
    server moodle02 192.168.20.32:80  weight 1 check cookie s2
    server moodle03 192.168.20.33:80  weight 1 check cookie s3
EOF
```

Cấu hình HAProxy Log
```
cp /etc/rsyslog.conf /etc/rsyslog.conf.org

echo '
\$ModLoad imuxsock
\$ModLoad imklog
\$ActionFileDefaultTemplate RSYSLOG_TraditionalFileFormat
\$FileOwner root
\$FileGroup adm
\$FileCreateMode 0640
\$DirCreateMode 0755
\$Umask 0022
$ModLoad imudp
$UDPServerRun 514
$UDPServerAddress 127.0.0.1
local2.*    /var/log/haproxy.log


auth,authpriv.*-/var/log/auth.log
daemon.*-/var/log/daemon.log
kern.*-/var/log/kern.log
cron.*-/var/log/cron.log
user.*-/var/log/user.log
mail.*-/var/log/mail.log
local7.*-/var/log/boot.log
local6.*-/var/log/cmdlog.log ' > /etc/rsyslog.conf

echo 'local2.*    /var/log/haproxy.log' > /etc/rsyslog.d/haproxy.conf
systemctl restart rsyslog
echo 'net.ipv4.ip_nonlocal_bind = 1' >> /etc/sysctl.conf
```

Tắt dịch vụ HAProxy, dịch vụ HAProxy sẽ được quản lý bởi Pacemaker
```
sudo systemctl stop haproxy
```

Lưu ý:
- Khi chưa triển khai dịch vụ Pacemaker, không thể khởi động được HAproxy

## Cài đăt Moodle

### Trên node `Moodle01`
> Hostname `tlumoodle01`

Tải mã nguồn Moodle version 3.7
```
cd
yum install -y wget

wget https://download.moodle.org/download.php/direct/stable37/moodle-latest-37.tgz
```

Giải nén vào thư mục `/var/www/html` + Phân quyền thư mục
```
sudo tar -xvzf moodle-latest-37.tgz -C /var/www/html
sudo chown -R root:root /var/www/html/moodle
```

Cấu hình Virtual Host
```
sed -i "s/Listen 80/Listen 192.168.20.31:80/g" /etc/httpd/conf/httpd.conf
cat /etc/httpd/conf/httpd.conf | grep 'Listen'

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
sudo /usr/bin/php /var/www/html/moodle/admin/cli/install.php --chmod=2777 \
 --lang=en --wwwroot=http://192.168.20.31 --dataroot=/var/moodledata \
 --dbtype=mariadb --dbhost=192.168.20.44 --dbname=moodle --dbuser=moodleuser --dbpass=Cloud3652019a@ \
 --fullname=MoodleNH --shortname=MNH \
 --adminuser=admin --adminpass=Cloud3652019a@ --adminemail=thanhnb@nhanhoa.com.vn \
 --agree-license
```

Kết quả
```
== Choose a language ==
en - English (en)
? - Available language packs
type value, press Enter to use default value (en)
: [ENTER]
-------------------------------------------------------------------------------
== Data directories permission ==
type value, press Enter to use default value (2777)
: [ENTER]
-------------------------------------------------------------------------------
== Web address ==
type value, press Enter to use default value (http://192.168.20.31)
: [ENTER]
-------------------------------------------------------------------------------
== Data directory ==
type value, press Enter to use default value (/var/moodledata)
: [ENTER]
-------------------------------------------------------------------------------
== Choose database driver ==
 mysqli 
 mariadb 
type value, press Enter to use default value (mariadb)
: [ENTER]
-------------------------------------------------------------------------------
== Database host ==
type value, press Enter to use default value (192.168.20.44)
: [ENTER]
-------------------------------------------------------------------------------
== Database name ==
type value, press Enter to use default value (moodle)
: [ENTER]
-------------------------------------------------------------------------------
== Tables prefix ==
type value, press Enter to use default value (mdl_)
: [ENTER]
-------------------------------------------------------------------------------
== Database port ==
type value, press Enter to use default value ()
: [ENTER]
-------------------------------------------------------------------------------
== Unix socket ==
type value, press Enter to use default value ()
: [ENTER]
-------------------------------------------------------------------------------
== Database user ==
type value, press Enter to use default value (moodleuser)
: [ENTER]
-------------------------------------------------------------------------------
== Database password ==
type value, press Enter to use default value (Cloud3652019a@)
: [ENTER]
-------------------------------------------------------------------------------
== Full site name ==
type value, press Enter to use default value (MoodleNH)
: [ENTER]
-------------------------------------------------------------------------------
== Short name for site (eg single word) ==
type value, press Enter to use default value (MNH)
: [ENTER]
-------------------------------------------------------------------------------
== Admin account username ==
type value, press Enter to use default value (admin)
: [ENTER]
-------------------------------------------------------------------------------
== New admin user password ==
type value
: Cloud3652019a@
-------------------------------------------------------------------------------
== New admin user email address ==
type value, press Enter to use default value (thanhnb@nhanhoa.com.vn)
: [ENTER]
-------------------------------------------------------------------------------
== Upgrade key (leave empty to not set it) ==
type value
: [ENTER]
-------------------------------------------------------------------------------
== Setting up database ==
-->System
...

Installation completed successfully.
```

> Lưu ý: Các node Moodle tiếp theo tương tự

Chỉnh sửa quyền trên file /var/www/html/config.php
```
sudo chmod o+r /var/www/html/moodle/config.php
```

Thiết lập crontab
```
sudo crontab -u apache -e

## Nội dung
* * * * *    /usr/bin/php /var/www/html/moodle/admin/cli/cron.php >/dev/null
```

Khởi động lại httpd
```
sudo systemctl restart httpd.service
```

Tunning OS phục vụ tăng khả năng xử lý tải HAProxy, Apache (Tùy chọn)
```
echo '
fs.file-max = 100000
fs.nr_open = 100000

net.ipv4.ip_local_port_range=1024 65535
net.ipv4.tcp_max_syn_backlog = 100000
net.core.netdev_max_backlog = 100000
net.core.somaxconn = 65534

net.ipv4.ip_local_port_range=1024 65535' >> /etc/sysctl.conf

echo '
* soft nofile 100000
* hard nofile 100000
root soft nofile 100000
root hard nofile 100000' >> /etc/security/limits.conf

ulimit -n 100000
ulimit -a

sysctl -p
```

Tunning Prefox Apache
```
echo '
KeepAlive On
MaxKeepAliveRequests 10
KeepAliveTimeout 5

<IfModule prefork.c>
    StartServers 20
    MinSpareServers 25
    MaxSpareServers 100
    ServerLimit 4096
    MaxClients 4096
    MaxRequestsPerChild 1000
</IfModule>' >> /etc/httpd/conf/httpd.conf 
```

### Trên node `Moodle02`
> Hostname `tlumoodle02`

Tải mã nguồn Moodle version 3.7
```
cd
yum install -y wget

wget https://download.moodle.org/download.php/direct/stable37/moodle-latest-37.tgz
```

Giải nén vào thư mục `/var/www/html` + Phân quyền thư mục
```
sudo tar -xvzf moodle-latest-37.tgz -C /var/www/html
sudo chown -R root:root /var/www/html/moodle
```

Cấu hình Virtual Host
```
sed -i "s/Listen 80/Listen 192.168.20.32:80/g" /etc/httpd/conf/httpd.conf
cat /etc/httpd/conf/httpd.conf | grep 'Listen'

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
sudo /usr/bin/php /var/www/html/moodle/admin/cli/install.php --chmod=2777 \
 --lang=en --wwwroot=http://192.168.20.32 --dataroot=/var/moodledata \
 --dbtype=mariadb --dbhost=192.168.20.44 --dbname=moodle --dbuser=moodleuser --dbpass=Cloud3652019a@ \
 --fullname=MoodleNH --shortname=MNH \
 --adminuser=admin --adminpass=Cloud3652019a@ --adminemail=thanhnb@nhanhoa.com.vn \
 --agree-license --skip-database
```

Chỉnh sửa quyền trên file /var/www/html/config.php
```
sudo chmod o+r /var/www/html/moodle/config.php
```

Thiết lập crontab
```
sudo crontab -u apache -e

## Nội dung
* * * * *    /usr/bin/php /var/www/html/moodle/admin/cli/cron.php >/dev/null
```

Khởi động lại httpd
```
sudo systemctl restart httpd.service
```

Tunning OS phục vụ tăng khả năng xử lý tải HAProxy, Apache (Tùy chọn)
```
echo '
fs.file-max = 100000
fs.nr_open = 100000

net.ipv4.ip_local_port_range=1024 65535
net.ipv4.tcp_max_syn_backlog = 100000
net.core.netdev_max_backlog = 100000
net.core.somaxconn = 65534

net.ipv4.ip_local_port_range=1024 65535' >> /etc/sysctl.conf

echo '
* soft nofile 100000
* hard nofile 100000
root soft nofile 100000
root hard nofile 100000' >> /etc/security/limits.conf

ulimit -n 100000
ulimit -a

sysctl -p
```

Tunning Prefox Apache
```
echo '
KeepAlive On
MaxKeepAliveRequests 10
KeepAliveTimeout 5

<IfModule prefork.c>
    StartServers 20
    MinSpareServers 25
    MaxSpareServers 100
    ServerLimit 4096
    MaxClients 4096
    MaxRequestsPerChild 1000
</IfModule>' >> /etc/httpd/conf/httpd.conf 
```

### Trên node `Moodle03`
> Hostname `tlumoodle03`

Tải mã nguồn Moodle version 3.7
```
cd
yum install -y wget

wget https://download.moodle.org/download.php/direct/stable37/moodle-latest-37.tgz
```

Giải nén vào thư mục `/var/www/html` + Phân quyền thư mục
```
sudo tar -xvzf moodle-latest-37.tgz -C /var/www/html
sudo chown -R root:root /var/www/html/moodle
```

Cấu hình Virtual Host
```
sed -i "s/Listen 80/Listen 192.168.20.33:80/g" /etc/httpd/conf/httpd.conf
cat /etc/httpd/conf/httpd.conf | grep 'Listen'

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
sudo /usr/bin/php /var/www/html/moodle/admin/cli/install.php --chmod=2777 \
 --lang=en --wwwroot=http://192.168.20.33 --dataroot=/var/moodledata \
 --dbtype=mariadb --dbhost=192.168.20.44 --dbname=moodle --dbuser=moodleuser --dbpass=Cloud3652019a@ \
 --fullname=MoodleNH --shortname=MNH \
 --adminuser=admin --adminpass=Cloud3652019a@ --adminemail=thanhnb@nhanhoa.com.vn \
 --agree-license --skip-database
```

Chỉnh sửa quyền trên file /var/www/html/config.php
```
sudo chmod o+r /var/www/html/moodle/config.php
```

Thiết lập crontab
```
sudo crontab -u apache -e

## Nội dung
* * * * *    /usr/bin/php /var/www/html/moodle/admin/cli/cron.php >/dev/null
```

Khởi động lại httpd
```
sudo systemctl restart httpd.service
```

Tunning OS phục vụ tăng khả năng xử lý tải HAProxy, Apache (Tùy chọn)
```
echo '
fs.file-max = 100000
fs.nr_open = 100000

net.ipv4.ip_local_port_range=1024 65535
net.ipv4.tcp_max_syn_backlog = 100000
net.core.netdev_max_backlog = 100000
net.core.somaxconn = 65534

net.ipv4.ip_local_port_range=1024 65535' >> /etc/sysctl.conf

echo '
* soft nofile 100000
* hard nofile 100000
root soft nofile 100000
root hard nofile 100000' >> /etc/security/limits.conf

ulimit -n 100000
ulimit -a

sysctl -p
```

Tunning Prefox Apache
```
echo '
KeepAlive On
MaxKeepAliveRequests 10
KeepAliveTimeout 5

<IfModule prefork.c>
    StartServers 20
    MinSpareServers 25
    MaxSpareServers 100
    ServerLimit 4096
    MaxClients 4096
    MaxRequestsPerChild 1000
</IfModule>' >> /etc/httpd/conf/httpd.conf 
```

## Triển khai Cluster Pacemaker

### Cài đặt Pacemaker
> Thực hiện trên cả 3 node Moodle01 (`tlumoodle01`), Moodle02 (`tlumoodle02`), Moodle03 (`tlumoodle03`)

Cài đặt
```
yum -y install pacemaker pcs
```

Khởi động dịch vụ
```
systemctl start pcsd 
systemctl enable pcsd
```

Thiết lập mật khẩu user `hacluster`
```
echo "Cloud3652019a@" | passwd --stdin hacluster
```
Lưu ý:
- Mật khẩu user hacluster phải đồng bộ trên tất cả các node

### Cấu hình Cluster Pacemaker
> Thực hiện trên Moodle01 (`tlumoodle01`)

Chứng thực cluster, nhập chính xác tài khoản user hacluster

```
pcs cluster auth tlumoodle01 tlumoodle02 tlumoodle03
```

Khởi tạo cấu hình ban đầu của Cluster
```
pcs cluster setup --name ha_cluster tlumoodle01 tlumoodle02 tlumoodle03
```

Khởi động cluster
```
pcs cluster start --all
```

Cho phép cluster khởi động cùng OS
```
pcs cluster enable --all
```

Thiết lập Cluster
```
pcs property set stonith-enabled=false
pcs property set no-quorum-policy=ignore
pcs property set default-resource-stickiness="INFINITY"
```

Kiểm tra thiết lập cluster
```
pcs property list
```


Tạo Resource IP VIP Cluster
```
pcs resource create Virtual_IP ocf:heartbeat:IPaddr2 ip=192.168.20.30 cidr_netmask=24 op monitor interval=30s
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

### Chỉnh sửa lại cấu hình Moodle trên Pacemaker
> Thực hiện trên cả 3 node Moodle01 (`tlumoodle01`), Moodle02 (`tlumoodle02`), Moodle03 (`tlumoodle03`)

```
sed -i "s/192.168.20.31/192.168.20.30/g" /var/www/html/moodle/config.php
sed -i "s/192.168.20.32/192.168.20.30/g" /var/www/html/moodle/config.php
sed -i "s/192.168.20.33/192.168.20.30/g" /var/www/html/moodle/config.php
```

## Kiểm tra

Kiểm tra trạng thái dịch vụ, Truy cập http://192.168.20.30:8080/stats (Mật khẩu: admin / Cloud3652019a@)

![](/images/05-caidat-moodle/kiem_tra_moodle_cluster_stats_page.png)

Nếu có thể truy cập tức HAProxy đa hoạt động. Để ý cá node `moodle01`, `moodle02`, `moodle03`, nếu các trường đều xanh tức dịch vụ Apache trên các node Moodle đã chạy.

Kiểm tra trạng thái cluster
```
[root@tlumoodle01 src]# pcs status
Cluster name: ha_cluster
Stack: corosync
Current DC: tlumoodle02 (version 1.1.19-8.el7_6.4-c3c624ea3d) - partition with quorum
Last updated: Wed Sep 18 17:43:01 2019
Last change: Tue Sep 17 17:03:13 2019 by root via crm_resource on tlumoodle01

3 nodes configured
5 resources configured

Online: [ tlumoodle01 tlumoodle02 tlumoodle03 ]

Full list of resources:

 Virtual_IP	(ocf::heartbeat:IPaddr2):	Started tlumoodle01
 Loadbalancer_HaProxy	(systemd:haproxy):	Started tlumoodle01
 Clone Set: Web_Cluster-clone [Web_Cluster] (unique)
     Web_Cluster:0	(systemd:httpd):	Started tlumoodle02
     Web_Cluster:1	(systemd:httpd):	Started tlumoodle03
     Web_Cluster:2	(systemd:httpd):	Started tlumoodle01

Daemon Status:
  corosync: active/enabled
  pacemaker: active/enabled
  pcsd: active/enabled
```

Resource `Web_Cluster-clone` phải chạy trên cả 3 node (`tlumoodle01`, `tlumoodle02`, `tlumoodle03`) => Tức Pacemaker đã hoạt động chính xác.

Tìm Node đang chạy resource `Loadbalancer_HaProxy` (`tlumoodle01`) thực hiện câu lệnh
```
[root@tlumoodle01 ~]# tail -n 1000 /var/log/haproxy.log | grep moodle01 | wc -l
258
[root@tlumoodle01 ~]# tail -n 1000 /var/log/haproxy.log | grep moodle02 | wc -l
509
[root@tlumoodle01 ~]# tail -n 1000 /var/log/haproxy.log | grep moodle03 | wc -l
223
```

Nếu các kết quả đều lớn hơn khác 0 tức dịch vụ Moodle Web trên từng node Moodle đã hoạt động tuy nhiên vẫn cần check thêm Moodle có hoạt động bình thường trên từng node. (Để chính xác nhất cần phải có traffic thực truy cập vào moodle)

Kiểm tra dịch vụ trên từng node
```
[root@tlumoodle01 ~]# curl -I 192.168.20.31
HTTP/1.1 303 See Other
Date: Wed, 18 Sep 2019 11:05:48 GMT
Server: Apache/2.4.6 (CentOS) PHP/7.2.22

[root@tlumoodle01 ~]# curl -I 192.168.20.32
HTTP/1.1 303 See Other
Date: Wed, 18 Sep 2019 11:05:48 GMT
Server: Apache/2.4.6 (CentOS) PHP/7.2.22

[root@tlumoodle01 ~]# curl -I 192.168.20.33
HTTP/1.1 303 See Other
Date: Wed, 18 Sep 2019 11:05:48 GMT
Server: Apache/2.4.6 (CentOS) PHP/7.2.22
```

- Nếu kết quả trả về khi CURL đều bằng 303 (`HTTP/1.1 303 See Other`), Moodle trên từng node đã hoạt động.

## Nguồn

https://blog.cloud365.vn/search/?q=High+Availability
