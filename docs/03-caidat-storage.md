# Triển khai dịch vụ Storage và Backup
---
## Mô hình

### Quy hoạch
![](/images/03-caidat-storage/ip_planning_storage_backup.png)

### Mô hình triển khai
![](/images/03-caidat-storage/mohinh_storage_backup.png)

### Lưu ý:
- Node Storage có hostname `tlustorage01`
- Node Backup có hostname `tlustorage02`
- Node Storage và Node Backup sử dụng OS CENTOS 7

## Chuẩn bị

### Trên node `Storage`
> Hostname `tlustorage01`

Cập nhật hệ điều hành

```
yum install epel-release -y
yum update -y
```

Thiết lập Hostname
```
hostnamectl set-hostname tlustorage01
```

Cấu hình Network
```
echo "Setup IP eth0"
nmcli c modify eth0 ipv4.addresses 192.168.20.41/24
nmcli c modify eth0 ipv4.gateway 192.168.20.1
nmcli c modify eth0 ipv4.dns 8.8.8.8
nmcli c modify eth0 ipv4.method manual
nmcli con mod eth0 connection.autoconnect yes


echo "Setup IP eth1"
nmcli c modify eth1 ipv4.addresses 192.168.21.41/24
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
NTP_SERVER_IP='192.168.20.54'
sed -i '/server/d' /etc/chrony.conf
echo "server $NTP_SERVER_IP iburst" >> /etc/chrony.conf
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

### Trên node `Backup`
> Hostname `tlustorage02`

Cập nhật hệ điều hành
```
yum install epel-release -y
yum update -y
```

Thiết lập Hostname
```
hostnamectl set-hostname tlustorage02
```

Cấu hình Network
```
echo "Setup IP eth0"
nmcli c modify eth0 ipv4.addresses 192.168.20.42/24
nmcli c modify eth0 ipv4.gateway 192.168.20.1
nmcli c modify eth0 ipv4.dns 8.8.8.8
nmcli c modify eth0 ipv4.method manual
nmcli con mod eth0 connection.autoconnect yes


echo "Setup IP eth1"
nmcli c modify eth1 ipv4.addresses 192.168.21.42/24
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
NTP_SERVER_IP='192.168.20.54'
sed -i '/server/d' /etc/chrony.conf
echo "server $NTP_SERVER_IP iburst" >> /etc/chrony.conf
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

## Triển khai dịch vụ NFS trên node Storage
> Thực hiện trên node `Storage` (hostname `tlustorage01`)

Cài đặt dịch vụ NFS, Rsync
```
yum install nfs-utils rsync httpd -y
```

Kiểm tra ổ data (vdb 1 TB)
```
[root@tlustorage01 ~]# lsblk 
NAME                    MAJ:MIN RM   SIZE RO TYPE MOUNTPOINT
sr0                      11:0    1  1024M  0 rom  
vda                     252:0    0   200G  0 disk 
├─vda1                  252:1    0   512M  0 part /boot
└─vda2                  252:2    0 199,5G  0 part 
  ├─VolGroup00-LogVol01 253:0    0 191,6G  0 lvm  /
  └─VolGroup00-LogVol00 253:1    0   7,9G  0 lvm  [SWAP]
vdb                     252:16   0  1000G  0 disk
```

Khởi tạo LVM trên ổ `vdb`, đồng thời định dạng ext4 trên LV được tạo ra
```
pvcreate /dev/vdb
vgcreate vg-moodle-data /dev/vdb 
lvcreate --yes -l 100%FREE -n lv-moodle-data vg-moodle-data
mkfs.ext4 -F /dev/vg-moodle-data/lv-moodle-data
```

Kiểm tra
```
[root@tlustorage01 ~]# lsblk 
NAME                                MAJ:MIN RM   SIZE RO TYPE MOUNTPOINT
sr0                                  11:0    1  1024M  0 rom  
vda                                 252:0    0   200G  0 disk 
├─vda1                              252:1    0   512M  0 part /boot
└─vda2                              252:2    0 199,5G  0 part 
  ├─VolGroup00-LogVol01             253:0    0 191,6G  0 lvm  /
  └─VolGroup00-LogVol00             253:1    0   7,9G  0 lvm  [SWAP]
vdb                                 252:16   0  1000G  0 disk 
└─vg--moodle--data-lv--moodle--data 253:2    0  1000G  0 lvm  
```

Tạo mới phân vùng lưu dữ liệu Moodle (`/var/moodledata`)
```
sudo mkdir -p /var/moodledata
mount /dev/vg-moodle-data/lv-moodle-data /var/moodledata

echo '/dev/vg-moodle-data/lv-moodle-data    /var/moodledata      ext4        defaults        0 0' >> /etc/fstab
```

Kiểm tra
```
[root@tlustorage01 ~]# findmnt -lo source,target,fstype,label,options,used -t ext4
SOURCE                                        TARGET          FSTYPE LABEL OPTIONS                    USED
/dev/vda1                                     /boot           ext4         rw,relatime,data=ordered 131,5M
/dev/mapper/vg--moodle--data-lv--moodle--data /var/moodledata ext4         rw,relatime,data=ordered    76M

[root@tlustorage01 ~]# df -Th
Filesystem                                    Type      Size  Used Avail Use% Mounted on
/dev/mapper/VolGroup00-LogVol01               xfs       192G  1,3G  191G   1% /
devtmpfs                                      devtmpfs  7,9G     0  7,9G   0% /dev
tmpfs                                         tmpfs     7,9G     0  7,9G   0% /dev/shm
tmpfs                                         tmpfs     7,9G  8,7M  7,9G   1% /run
tmpfs                                         tmpfs     7,9G     0  7,9G   0% /sys/fs/cgroup
/dev/vda1                                     ext4      488M  132M  321M  30% /boot
tmpfs                                         tmpfs     1,6G     0  1,6G   0% /run/user/0
/dev/mapper/vg--moodle--data-lv--moodle--data ext4      985G   77M  935G   1% /var/moodledata

```

Phân quyền thư mục chưa data
```
sudo chown -R apache:apache /var/moodledata
sudo chmod -R 755 /var/moodledata
```

Kiểm tra
```
[root@tlustorage01 ~]# ll /var/moodledata
total 16
drwxr-xr-x 2 apache apache 16384 11:54 16 Th09 lost+found
```

Khởi động dịch vụ NFS
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

Add Mount point
```
echo '/var/moodledata  192.168.21.31(rw,sync,no_root_squash,no_all_squash) 192.168.21.32(rw,no_root_squash) 192.168.21.33(rw,no_root_squash)' > /etc/exports
```

Share thư mục `/var/moodledata` thông qua NFS
```
exportfs -ra
```

Hoặc có thể khởi động lại dịch vụ NFS
```
systemctl restart nfs-server
```

## Triển khai node Backup
> Thực hiện trên node `Backup` (hostname `tlustorage02`)

Cài đặt Rsync
```
yum install rsync -y
```

Kiểm tra ổ data (vdb 1 TB)

```
[root@tlustorage02 ~]# lsblk 
NAME                    MAJ:MIN RM   SIZE RO TYPE MOUNTPOINT
sr0                      11:0    1  1024M  0 rom  
vda                     252:0    0   200G  0 disk 
├─vda1                  252:1    0   512M  0 part /boot
└─vda2                  252:2    0 199,5G  0 part 
  ├─VolGroup00-LogVol01 253:0    0 191,6G  0 lvm  /
  └─VolGroup00-LogVol00 253:1    0   7,9G  0 lvm  [SWAP]
vdb                     252:16   0  1000G  0 disk 
```

Khởi tạo LVM trên ổ `vdb`, đồng thời định dạng ext4 trên LV được tạo ra
```
pvcreate /dev/vdb
vgcreate vg-backup-data /dev/vdb 
lvcreate --yes -l 100%FREE -n lv-backup-data vg-backup-data
mkfs.ext4 -F /dev/vg-backup-data/lv-backup-data
```

Kiểm tra
```
[root@tlustorage02 ~]# lsblk
NAME                                MAJ:MIN RM   SIZE RO TYPE MOUNTPOINT
sr0                                  11:0    1  1024M  0 rom  
vda                                 252:0    0   200G  0 disk 
├─vda1                              252:1    0   512M  0 part /boot
└─vda2                              252:2    0 199,5G  0 part 
  ├─VolGroup00-LogVol01             253:0    0 191,6G  0 lvm  /
  └─VolGroup00-LogVol00             253:1    0   7,9G  0 lvm  [SWAP]
vdb                                 252:16   0  1000G  0 disk 
└─vg--backup--data-lv--backup--data 253:2    0  1000G  0 lvm  
```

Tạo mới phân vùng lưu dữ liệu backups
```
sudo mkdir -p /backups
mount /dev/vg-backup-data/lv-backup-data /backups
```

Thêm mount point
```
echo '/dev/vg-backup-data/lv-backup-data    /backups      ext4        defaults        0 0' >> /etc/fstab
```

Kiểm tra
```
[root@tlustorage02 ~]# findmnt -lo source,target,fstype,label,options,used -t ext4
SOURCE                                        TARGET   FSTYPE LABEL OPTIONS                    USED
/dev/vda1                                     /boot    ext4         rw,relatime,data=ordered 131,7M
/dev/mapper/vg--backup--data-lv--backup--data /backups ext4         rw,relatime,data=ordered    76M

[root@tlustorage02 ~]# df -Th
Filesystem                                    Type      Size  Used Avail Use% Mounted on
/dev/mapper/VolGroup00-LogVol01               xfs       192G  1,2G  191G   1% /
devtmpfs                                      devtmpfs  7,9G     0  7,9G   0% /dev
tmpfs                                         tmpfs     7,9G     0  7,9G   0% /dev/shm
tmpfs                                         tmpfs     7,9G  8,7M  7,9G   1% /run
tmpfs                                         tmpfs     7,9G     0  7,9G   0% /sys/fs/cgroup
/dev/vda1                                     ext4      488M  132M  321M  30% /boot
tmpfs                                         tmpfs     1,6G     0  1,6G   0% /run/user/0
/dev/mapper/vg--backup--data-lv--backup--data ext4      985G   77M  935G   1% /backups
```

Tổ chức thư mục chứa Backup
- `/backups/moodle/moodledata`: Chứa dữ liệu Data Moodle
- `/backups/moodle/database`: Chứa dữ liệu Database moodle

Tạo thư mục
```
mkdir -p /backups/moodle/moodledata
mkdir -p /backups/moodle/database
```

## Đồng bộ dữ liệu giữa Node Storage với Node Backup

Trong phần mình sẽ đồng bộ thư mục `/var/moodledata` trên node Storage (`tlustorage01`) tới `/backups/moodle/moodledata` Node Backup (`tlustorage02`)

Thực hiện trên node Storage (`tlustorage01`)
```
ssh-keggen
```
> press Enter với tất cả tham số

Kết quả
```
[root@tlustorage01 ~]# ssh-keygen 

Generating public/private rsa key pair.
Enter file in which to save the key (/root/.ssh/id_rsa): [ENTER]
Enter passphrase (empty for no passphrase): [ENTER]
Enter same passphrase again: [ENTER]
Your identification has been saved in /root/.ssh/id_rsa.
Your public key has been saved in /root/.ssh/id_rsa.pub.
...
```

Thực hiện trên node Backup (`tlustorage02`)
```
ssh-keggen
```
> press Enter với tất cả tham số

Kết quả
```
[root@tlustorage02 ~]# ssh-keygen 

Generating public/private rsa key pair.
Enter file in which to save the key (/root/.ssh/id_rsa): [ENTER]
Enter passphrase (empty for no passphrase): [ENTER]
Enter same passphrase again: [ENTER]
Your identification has been saved in /root/.ssh/id_rsa.
Your public key has been saved in /root/.ssh/id_rsa.pub.
...
```


Chuyến keypair từ node Storage (`tlustorage01`) sang node Backup (`tlustorage02`)
> Thực hiện trên node Storage
```
ssh-copy-id root@192.168.20.42
```

Kết quả (Thao tác có yêu cầu nhập mật khẩu)
```
[root@tlustorage01 ~]# ssh-copy-id root@192.168.20.42

/usr/bin/ssh-copy-id: INFO: Source of key(s) to be installed: "/root/.ssh/id_rsa.pub"
/usr/bin/ssh-copy-id: INFO: attempting to log in with the new key(s), to filter out any that are already installed

/usr/bin/ssh-copy-id: WARNING: All keys were skipped because they already exist on the remote system.
		(if you think this is a mistake, you may want to use -f option)
```

Thử đồng bộ dữ liệu giữa Storage với node Backup
```
rsync -azP --delete /var/moodledata/ root@192.168.21.42:/backups/moodle/moodledata
```

> Lưu ý: Nếu thác tác không cần nhập mật khẩu tức thành công.

Đưa câu lệnh vào crontab
```
echo '0 2 * * * rsync -azP --delete /var/moodledata/ root@192.168.21.42:/backups/moodle/moodledata' >> /etc/crontab
systemctl restart crond
```

Kiểm tra
```
[root@tlustorage01 ~]# ls /var/moodledata/

total 1756
drwxrwsrwx   3 root   apache    4096 15:53 16 Th09 cache
drwxrwsrwx  67 root   apache    4096 14:04 17 Th09 filedir
drwxrwsrwx   2 root   apache    4096 15:49 16 Th09 lang
drwxrwsrwx   9 root   apache    4096 14:08 18 Th09 localcache
drwxrwsrwx 139 apache apache    4096 12:00 18 Th09 lock
drwxr-xr-x   2 apache apache   16384 11:54 16 Th09 lost+found
drwxrwsrwx   4 apache apache    4096 12:01 17 Th09 models
drwxrwsrwx   2 root   apache    4096 15:50 16 Th09 muc
drwxrwsrwx   2 apache apache 1744896 14:08 18 Th09 sessions
drwxrwsrwx   6 root   apache    4096 13:58 17 Th09 temp
-rwxr-xr-x   1 apache apache       0 13:56 16 Th09 test001.txt
-rwxr-xr-x   1 apache apache       0 13:56 16 Th09 test002.txt
-rwxr-xr-x   1 apache apache       0 13:56 16 Th09 test003.txt
-rw-r--r--   1 root   root         0 15:09 16 Th09 test3.txt
drwxrwsrwx   2 apache apache    4096 00:55 18 Th09 trashdir
```

```
[root@tlustorage02 ~]# ls /backups/moodle/moodledata
total 1556
drwxrwsrwx   3 root   48    4096 15:53 16 Th09 cache
drwxrwsrwx  67 root   48    4096 14:04 17 Th09 filedir
drwxrwsrwx   2 root   48    4096 15:49 16 Th09 lang
drwxrwsrwx   9 root   48    4096 14:08 18 Th09 localcache
drwxrwsrwx 139   48   48    4096 12:00 18 Th09 lock
drwxr-xr-x   2   48   48    4096 11:54 16 Th09 lost+found
drwxrwsrwx   4   48   48    4096 12:01 17 Th09 models
drwxrwsrwx   2 root   48    4096 15:50 16 Th09 muc
drwxrwsrwx   2   48   48 1552384 14:08 18 Th09 sessions
drwxrwsrwx   6 root   48    4096 13:58 17 Th09 temp
-rwxr-xr-x   1   48   48       0 13:56 16 Th09 test001.txt
-rwxr-xr-x   1   48   48       0 13:56 16 Th09 test002.txt
-rwxr-xr-x   1   48   48       0 13:56 16 Th09 test003.txt
-rw-r--r--   1 root root       0 15:09 16 Th09 test3.txt
drwxrwsrwx   2   48   48    4096 00:55 18 Th09 trashdir
```

Tới đây, phần triển khai dịch vụ Storage và Backup đã thành công, mời bạn chuyển sang bài tiếp theo: [Triển khai Database](/docs/04-caidat-database.md)

## Nguồn

https://news.cloud365.vn/nfs-network-file-system/