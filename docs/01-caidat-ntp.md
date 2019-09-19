# Cài đặt NTP Server
---
## Mô hình

### Lưu ý:
- Node NTP sử dụng OS CENTOS 7

### Quy hoạch
![](/images/01-caidat-ntp/ip_planning_nginx_ntp.png)

### Mô hình triển khai
![](/images/01-caidat-ntp/mohinh_nginx_ntp.png)

### Lưu ý:
- Node NTP Server có hostname `tluntp01`

## Chuẩn bị

### Cập nhật hệ điều hành

```
yum install epel-release -y
yum update -y
```

### Cấu hình chuẩn bị

Thiết lập Hostname
```
hostnamectl set-hostname tluntp01
```

Cấu hình Network

```
echo "Setup IP eth0"
nmcli c modify eth0 ipv4.addresses 192.168.20.54/24
nmcli c modify eth0 ipv4.gateway 192.168.20.1
nmcli c modify eth0 ipv4.dns 8.8.8.8
nmcli c modify eth0 ipv4.method manual
nmcli con mod eth0 connection.autoconnect yes

echo "Setup IP eth1"
nmcli c modify eth1 ipv4.addresses 192.168.21.54/24
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

Khởi động lại
```
init 6
```

### Thiết lập NTP Server

Cài đặt dịch vụ chrony
```
yum -y install chrony
```

Cấu hình Chrony
```
sed -i 's/server 0.centos.pool.ntp.org iburst/ \
server 1.vn.pool.ntp.org iburst \
server 0.asia.pool.ntp.org iburst \
server 3.asia.pool.ntp.org iburst/g' /etc/chrony.conf
sed -i 's/server 1.centos.pool.ntp.org iburst/#/g' /etc/chrony.conf
sed -i 's/server 2.centos.pool.ntp.org iburst/#/g' /etc/chrony.conf
sed -i 's/server 3.centos.pool.ntp.org iburst/#/g' /etc/chrony.conf
sed -i 's/#allow 192.168.0.0\/16/allow 192.168.20.0\/24/g' /etc/chrony.conf
```

Lưu ý:
- NTP Server sẽ cho phép các IP trong dải 192.168.20.x kết nối tới để đồng bộ thời gian.

Khởi động dịch vụ
```
systemctl enable chronyd.service
systemctl restart chronyd.service
chronyc sources
```

Tới đây, phần thiết lập NTP Server đã thành công, mời bạn chuyển sang bài tiếp theo: [Triển khai Nginx](/docs/02-caidat-nginx.md)


## Nguồn

https://news.cloud365.vn/chronyd-dich-vu-thay-the-ntpd-tren-unix/

https://news.cloud365.vn/cai-dat-chrony-tren-centos-rhel-7/