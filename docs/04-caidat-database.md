# Triển khai dịch vụ Database
---
## Mô hình

### Quy hoạch
![](/images/04-caidat-database/ip_planning_database.png)

### Mô hình triển khai
![](/images/04-caidat-database/mohinh_database.png)

### Lưu ý:
- Node Database Master có hostname `tludatabase01`
- Node Database Slave Replication có hostname `tludatabase02`
- Node Master và Slave sử dụng OS CENTOS 7

## Chuẩn bị

### Trên node `Master`
> Hostname `tludatabase01`

Cập nhật hệ điều hành
```
yum install epel-release -y
yum update -y
```


Thiết lập Hostname
```
hostnamectl set-hostname tludatabase01
```

Cấu hình Network
```
echo "Setup IP eth0"
nmcli c modify eth0 ipv4.addresses 192.168.20.44/24
nmcli c modify eth0 ipv4.gateway 192.168.20.1
nmcli c modify eth0 ipv4.dns 8.8.8.8
nmcli c modify eth0 ipv4.method manual
nmcli con mod eth0 connection.autoconnect yes


echo "Setup IP eth1"
nmcli c modify eth1 ipv4.addresses 192.168.21.44/24
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

### Trên node `Slave`
> Hostname `tludatabase02`

Cập nhật hệ điều hành
```
yum install epel-release -y
yum update -y
```

Thiết lập Hostname
```
hostnamectl set-hostname tludatabase02
```

Cấu hình Network
```
echo "Setup IP eth0"
nmcli c modify eth0 ipv4.addresses 192.168.20.45/24
nmcli c modify eth0 ipv4.gateway 192.168.20.1
nmcli c modify eth0 ipv4.dns 8.8.8.8
nmcli c modify eth0 ipv4.method manual
nmcli con mod eth0 connection.autoconnect yes


echo "Setup IP eth1"
nmcli c modify eth1 ipv4.addresses 192.168.21.45/24
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

## Triển khai mô hình Database Master Slave

### Thực hiện trên node Master
> Hostname `tludatabase01`

Cài đặt Database 10.2
```
echo '[mariadb]
name = MariaDB
baseurl = http://yum.mariadb.org/10.2/centos7-amd64
gpgkey=https://yum.mariadb.org/RPM-GPG-KEY-MariaDB
gpgcheck=1' >> /etc/yum.repos.d/MariaDB.repo

yum -y update

yum install -y mariadb mariadb-server
```

Cấu hình database
```
cp /etc/my.cnf /etc/my.cnf.org

cat > /etc/my.cnf << EOF
[mysqld]
# Cau hinh Log
slow_query_log                  = 1
slow_query_log_file             = /var/log/mariadb/slow.log
long_query_time                 = 5
log_error                       = /var/log/mariadb/error.log

## Su dung khi debug
# general_log_file                = /var/log/mariadb/mysql.log
# general_log                     = 1

# Tunning
max_connections = 6000

# Cau hinh Master DB
log-bin
server_id=1
log-basename=master
binlog-format=row
binlog-do-db=moodle
bind-address=192.168.20.44

# Tunning rotate log
binlog-row-image=minimal
expire_logs_days=7


[client-server]
!includedir /etc/my.cnf.d
EOF

mkdir -p /var/log/mariadb/
chown -R mysql:mysql /var/log/mariadb/
```

Khởi tạo dịch vụ
```
systemctl start mariadb
systemctl enable mariadb
```

Kiểm tra
```
[root@tludatabase01 ~]# systemctl status mariadb
● mariadb.service - MariaDB 10.2.27 database server
   Loaded: loaded (/usr/lib/systemd/system/mariadb.service; enabled; vendor preset: disabled)
  Drop-In: /etc/systemd/system/mariadb.service.d
           └─migrated-from-my.cnf-settings.conf
   Active: active (running) since T2 2019-09-16 14:16:19 +07; 7min ago
     Docs: man:mysqld(8)
           https://mariadb.com/kb/en/library/systemd/
 Main PID: 10306 (mysqld)
   Status: "Taking your SQL requests now..."
   CGroup: /system.slice/mariadb.service
           └─10306 /usr/sbin/mysqld

Th09 16 14:16:18 tludatabase01 systemd[1]: Starting MariaDB 10.2.27 database server...
Th09 16 14:16:19 tludatabase01 mysqld[10306]: 2019-09-16 14:16:19 139833879525568 [Note] /us......
Th09 16 14:16:19 tludatabase01 systemd[1]: Started MariaDB 10.2.27 database server.
Hint: Some lines were ellipsized, use -l to show in full.
```

Tạo mới database cho Moodle
```
mysql -u root

CREATE DATABASE moodle DEFAULT CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
CREATE USER 'moodleuser'@'localhost' IDENTIFIED BY 'Cloud3652019a@';
GRANT ALL PRIVILEGES ON moodle.* TO 'moodleuser'@'localhost' IDENTIFIED BY 'Cloud3652019a@' WITH GRANT OPTION;

CREATE USER 'moodleuser'@'%' IDENTIFIED BY 'Cloud3652019a@';
GRANT ALL PRIVILEGES ON moodle.* TO 'moodleuser'@'%' IDENTIFIED BY 'Cloud3652019a@' WITH GRANT OPTION;

CREATE USER 'moodleuser'@'moodle01' IDENTIFIED BY 'Cloud3652019a@';
GRANT ALL PRIVILEGES ON moodle.* TO 'moodleuser'@'moodle01' IDENTIFIED BY 'Cloud3652019a@' WITH GRANT OPTION;

CREATE USER 'moodleuser'@'moodle02' IDENTIFIED BY 'Cloud3652019a@';
GRANT ALL PRIVILEGES ON moodle.* TO 'moodleuser'@'moodle02' IDENTIFIED BY 'Cloud3652019a@' WITH GRANT OPTION;

CREATE USER 'moodleuser'@'moodle03' IDENTIFIED BY 'Cloud3652019a@';
GRANT ALL PRIVILEGES ON moodle.* TO 'moodleuser'@'moodle03' IDENTIFIED BY 'Cloud3652019a@' WITH GRANT OPTION;

FLUSH PRIVILEGES;
EXIT;
```

Khóa DB, phục vụ quá trình cấu hình DB Slave
```
mysql -u root

FLUSH TABLES WITH READ LOCK;
SET GLOBAL read_only = ON;

EXIT;
```

Tạo Slave user, phục vụ tiến trình replication tại node Slave kết nối tới node Master
```
mysql -u root

CREATE USER 'slave'@'localhost' IDENTIFIED BY 'Cloud3652019a@';
GRANT REPLICATION SLAVE ON *.* TO slave IDENTIFIED BY 'Cloud3652019a@' WITH GRANT OPTION;
FLUSH PRIVILEGES;
SHOW MASTER STATUS;

EXIT;
```

Kiêm tra giá trị
> Lưu ý kết quả, sẽ có giá trị sử dụng trong các bước tiếp theo
```
mysql -u root
SHOW MASTER STATUS;
EXIT;
```

Kết quả
```
MariaDB [(none)]> SHOW MASTER STATUS;
+-------------------+----------+--------------+------------------+
| File              | Position | Binlog_Do_DB | Binlog_Ignore_DB |
+-------------------+----------+--------------+------------------+
| master-bin.000001 |     3463 | moodle       |                  |
+-------------------+----------+--------------+------------------+
1 row in set (0.00 sec)
```

Lưu ý:
- 2 giá trị: `File = master-bin.000001` và `Position = 3463` sẽ được sử dụng cho cấu hình Node Slave

Dump database `moodle` (Database moodle sẽ được replicate tới node Slave)
```
mysqldump -u root -p moodle > moodle_raw.sql
```

Chuyển file database `moodle` sang node Database Slave (`tludatabase02`)
> Thao tác cần nhập mật khẩu
```
scp moodle_raw.sql root@192.168.20.45:/root/
```

Cập nhật các bảng hệ thống
```
mysql_upgrade -u root -p
```

LƯU Ý:
- Để mở khóa Database, lưu ý bước mở khóa DB sẽ được thực hiện sau khi cấu hình xong node Slave (`tludatabase02`)

```
mysql -u root

UNLOCK TABLES;
SET GLOBAL read_only = OFF;

EXIT;
```

Nếu cấu hình slave thành công, tại master node
```
MariaDB [(none)]> show processlist;
+----+-------------+---------------------+------+-------------+------+-----------------------------------------------------------------------+------------------+----------+
| Id | User        | Host                | db   | Command     | Time | State                                                                 | Info             | Progress |
+----+-------------+---------------------+------+-------------+------+-----------------------------------------------------------------------+------------------+----------+
|  1 | system user |                     | NULL | Daemon      | NULL | InnoDB purge worker                                                   | NULL             |    0.000 |
|  3 | system user |                     | NULL | Daemon      | NULL | InnoDB purge coordinator                                              | NULL             |    0.000 |
|  2 | system user |                     | NULL | Daemon      | NULL | InnoDB purge worker                                                   | NULL             |    0.000 |
|  4 | system user |                     | NULL | Daemon      | NULL | InnoDB purge worker                                                   | NULL             |    0.000 |
|  5 | system user |                     | NULL | Daemon      | NULL | InnoDB shutdown handler                                               | NULL             |    0.000 |
| 29 | slave       | (LƯU Ý SẼ CÓ THỂM TRƯỜNG NÀY) tludatabase02:54516 | NULL | Binlog Dump |  301 | Master has sent all binlog to slave; waiting for binlog to be updated | NULL             |    0.000 |
| 31 | root        | localhost           | NULL | Query       |    0 | init                                                                  | show processlist |    0.000 |
+----+-------------+---------------------+------+-------------+------+-----------------------------------------------------------------------+------------------+----------+
```

### Thực hiện trên node Slave
> Hostname `tludatabase02`


Cài đặt MariaDB 10.2
```
echo '[mariadb]
name = MariaDB
baseurl = http://yum.mariadb.org/10.2/centos7-amd64
gpgkey=https://yum.mariadb.org/RPM-GPG-KEY-MariaDB
gpgcheck=1' >> /etc/yum.repos.d/MariaDB.repo

yum -y update

yum install -y mariadb mariadb-server
```

Cấu hình database
```
cp /etc/my.cnf /etc/my.cnf.org

cat > /etc/my.cnf << EOF
[mysqld]
# Cau hinh Log
slow_query_log                  = 1
slow_query_log_file             = /var/log/mariadb/slow.log
long_query_time                 = 5
log_error                       = /var/log/mariadb/error.log

## Su dung khi debug
# general_log_file                = /var/log/mariadb/mysql.log
# general_log                     = 1

# Tunning
max_connections = 6000

# Cau hinh Slave DB
server_id=2
replicate-do-db=moodle

[client-server]
!includedir /etc/my.cnf.d
EOF

mkdir -p /var/log/mariadb/
chown -R mysql:mysql /var/log/mariadb/
```

Khởi động dịch vụ
```
systemctl start mariadb
systemctl enable mariadb
```

Tạo mới Database
> Database được replication từ database Master

```
mysql -u root

CREATE DATABASE moodle DEFAULT CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
CREATE USER 'moodleuser'@'localhost' IDENTIFIED BY 'Cloud3652019a@';
GRANT ALL PRIVILEGES ON moodle.* TO 'moodleuser'@'localhost' IDENTIFIED BY 'Cloud3652019a@' WITH GRANT OPTION;
FLUSH PRIVILEGES;
EXIT;
```

Load database
```
cd /root/
mysql -u root -p moodle < moodle_raw.sql
```
Lưu ý:
- File `moodle_raw.sql` có được từ bước node `tludatabase01`

Cập nhật các bảng hệ thống
```
mysql_upgrade -u root -p
```

Cấu hình Slave Replicate
```
mysql -u root

CHANGE MASTER TO
  MASTER_HOST='192.168.20.44',
  MASTER_USER='slave',
  MASTER_PASSWORD='Cloud3652019a@',
  MASTER_PORT=3306,
  MASTER_LOG_FILE='master-bin.000001',
  MASTER_LOG_POS=3463,
  MASTER_CONNECT_RETRY=10,
  MASTER_USE_GTID=slave_pos;
EXIT;
```

Lưu ý các giá trị:
- `MASTER_HOST`: Địa chỉ node Master;
- `MASTER_USER`, MASTER_PASSWORD user Slave được tạo tại Master;
- `MASTER_PORT`: Port kết nối;
- `MASTER_LOG_FILE`: File log bin của Master, có được từ bước `SHOW MASTER STATUS`;
- `MASTER_LOG_POS`: Giá trị Position SHOW MASTER STATUS;

Chạy Slave và kiểm tra trạng thái
```
mysql -u root

START SLAVE;
SHOW SLAVE STATUS\G;

EXIT;
```

Kết quả
```
MariaDB [(none)]> SHOW SLAVE STATUS\G;
*************************** 1. row ***************************
               Slave_IO_State: Waiting for master to send event
                  Master_Host: 192.168.20.44
                  Master_User: slave
                  Master_Port: 3306
                Connect_Retry: 10
              Master_Log_File: master-bin.000001
          Read_Master_Log_Pos: 3463
               Relay_Log_File: tludatabase02-relay-bin.000002
                Relay_Log_Pos: 3763
        Relay_Master_Log_File: master-bin.000001
             Slave_IO_Running: Yes
            Slave_SQL_Running: Yes
              Replicate_Do_DB: moodle
          Replicate_Ignore_DB: 
           Replicate_Do_Table: 
       Replicate_Ignore_Table: 
      Replicate_Wild_Do_Table: 
  Replicate_Wild_Ignore_Table: 
                   Last_Errno: 0
                   Last_Error: 
                 Skip_Counter: 0
          Exec_Master_Log_Pos: 3463
              Relay_Log_Space: 4080
              Until_Condition: None
               Until_Log_File: 
                Until_Log_Pos: 0
           Master_SSL_Allowed: No
           Master_SSL_CA_File: 
           Master_SSL_CA_Path: 
              Master_SSL_Cert: 
            Master_SSL_Cipher: 
               Master_SSL_Key: 
        Seconds_Behind_Master: 0
Master_SSL_Verify_Server_Cert: No
                Last_IO_Errno: 0
                Last_IO_Error: 
               Last_SQL_Errno: 0
               Last_SQL_Error: 
  Replicate_Ignore_Server_Ids: 
             Master_Server_Id: 1
               Master_SSL_Crl: 
           Master_SSL_Crlpath: 
                   Using_Gtid: Slave_Pos
                  Gtid_IO_Pos: 0-1-17
      Replicate_Do_Domain_Ids: 
  Replicate_Ignore_Domain_Ids: 
                Parallel_Mode: conservative
                    SQL_Delay: 0
          SQL_Remaining_Delay: NULL
      Slave_SQL_Running_State: Slave has read all relay log; waiting for the slave I/O thread to update it
```
- `Slave_IO_Running: Yes`, `Slave_SQL_Running: Yes` tức đã cấu hình thành công


Chuyển DB Slave sang dạng readonly, bảo toàn dữ liệu DB Slave
> Lưu ý: Nếu ghi đồng thời vào db `moodle` master lẫn db `moodle` slave, 2 db sẽ không thể đồng bộ.

```
SET GLOBAL read_only = ON;
```

Truy cập Node Master, mở khóa DB (Đã có hướng dẫn tại cấu hình Master)

Lưu ý:
> Vấn đề rotate log cho file bin log trên node Master

Mô tả:
- Thiết lập Rotate file Bin log trên node Master disable mặc định, dẫn tới file log này liên tục tăng cho đến khi full dung lượng của ổ. Vì vậy, chúng ta phải enable cấu hình rotate này lên.
- Trong mô hình, mình đã enable tham số này trong cấu hình /etc/my.conf tại node master

Kiểm tra trên node Master
```
cat /etc/my.cnf
...
# Tunning rotate log
binlog-row-image=minimal
expire_logs_days=7
```

## Vấn đề
> Rotate `binlog` chỉ xảy ra khi node __database khởi động lại__ dịch vụ hoặc __đưa ra chỉ thị rotate__.

Mô tả
- `binlog-row-image=minimal`: Ghi ít dữ liệu vào file bin log ít nhất có thể
- `expire_logs_days=7`: Giữ log trong vòng 7 ngày. Tự động rotate log mỗi khi khởi động mỗi khi node __database khởi động lại__ dịch vụ hoặc __đưa ra chỉ thị rotate__.

Cách đưa ra chỉ thị rotate thủ công
```
mysql -u root -p

### Kiểm tra các file Binary logs

MariaDB [(none)]> SHOW BINARY LOGS;
+-------------------+------------+
| Log_name          | File_size  |
+-------------------+------------+
| master-bin.000014 | 1073742621 |
| master-bin.000015 | 1019995810 |
| master-bin.000016 |    2659428 |
| master-bin.000017 |      13326 |
| master-bin.000018 |       1650 |
+-------------------+------------+

### Cắt Log

MariaDB [(none)]> FLUSH LOGS;
Query OK, 0 rows affected (0.02 sec)

MariaDB [(none)]> SHOW BINARY LOGS;
+-------------------+------------+
| Log_name          | File_size  |
+-------------------+------------+
| master-bin.000014 | 1073742621 |
| master-bin.000015 | 1019995810 |
| master-bin.000016 |    2659428 |
| master-bin.000017 |      13326 |
| master-bin.000018 |     263429 |
| master-bin.000019 |       2913 |
+-------------------+------------+
6 rows in set (0.00 sec)

### Xóa tới vị trí chỉ định

MariaDB [(none)]> PURGE BINARY LOGS TO 'master-bin.000016';
Query OK, 0 rows affected (0.52 sec)

MariaDB [(none)]> SHOW BINARY LOGS;
+-------------------+-----------+
| Log_name          | File_size |
+-------------------+-----------+
| master-bin.000016 |   2659428 |
| master-bin.000017 |     13326 |
| master-bin.000018 |    263429 |
| master-bin.000019 |    180953 |
+-------------------+-----------+
4 rows in set (0.00 sec)
```

Giải pháp khác:
- Có thể đặt crontab tự động khởi động lại MariaDB theo chu kỳ

## Nguồn

https://thesubtlepath.com/blog/mysql/binary-log-growth-handling-in-mysql/