# Cài đặt Nginx
---
## Mô hình

### Lưu ý:
- Node NTP sử dụng OS CENTOS 7

### Quy hoạch
![](/images/02-caidat-nginx/ip_planning_nginx_ntp.png)

### Mô hình triển khai
![](/images/02-caidat-nginx/mohinh_nginx_ntp.png)

### Lưu ý:
- Node Nginx có hostname `tlunginx01`

## Chuẩn bị

### Cập nhật hệ điều hành

```
yum install epel-release -y
yum update -y
```

### Cấu hình chuẩn bị

Thiết lập Hostname
```
hostnamectl set-hostname tlunginx01
```

Cấu hình Network

```
echo "Setup IP eth0"
nmcli c modify eth0 ipv4.addresses 192.168.20.53/24
nmcli c modify eth0 ipv4.gateway 192.168.20.1
nmcli c modify eth0 ipv4.dns 8.8.8.8
nmcli c modify eth0 ipv4.method manual
nmcli con mod eth0 connection.autoconnect yes

echo "Setup IP eth1"
nmcli c modify eth1 ipv4.addresses 192.168.21.53/24
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

Khởi động lại
```
init 6
```

### Secure Node Nginx

Vì node Nginx sẽ có IP Public, nên mình sẽ tiến hành Secure Node thông qua Iptable và fail2ban, thay đổi user được phép truy cập Node qua SSH.

Cài đặt Iptables
```
sudo yum install iptables-services -y
sudo systemctl start iptables
```

Khởi động dịch vụ Iptables
```
systemctl status iptables
systemctl enable iptables
```

Cài đặt fail2ban
```
yum install fail2ban -y 
```

Cấu hình fail2ban
```
> /etc/fail2ban/jail.conf

cat << EOF >> /etc/fail2ban/jail.conf
[DEFAULT]
ignoreip = 127.0.0.1
bantime = 600
findtime = 600
maxretry = 3
EOF
```

Lưu ý:
- Cấu hình tham số mặc định
- Bỏ qua ban ip localhost
- Thời gian ban: 10 phút

Cấu hình fail2ban cho ssh
```
cat << EOF >> /etc/fail2ban/jail.local
[sshd]
enabled  = true
filter   = sshd
action   = iptables[name=SSH, port=ssh, protocol=tcp]
logpath  = /var/log/auth.log
maxretry = 5
bantime = 3600
EOF
```

Lưu ý:
- Nếu truy cập sai mật khẩu SSH 5 lần, người dùng sẽ bị block trong 1 tiếng
- Ban bằng cơ chế IPtables

Kiểm tra bằng câu lệnh `iptables -L`

```bash
Chain INPUT (policy ACCEPT 0 packets, 0 bytes)
 pkts bytes target     prot opt in     out     source               destination         
  311 29167 f2b-SSH    tcp  --  any    any     anywhere             anywhere             tcp dpt:ssh
  480 40907 ACCEPT     all  --  any    any     anywhere             anywhere             state RELATED,ESTABLISHED
   11  1056 ACCEPT     icmp --  any    any     anywhere             anywhere            
    0     0 ACCEPT     all  --  lo     any     anywhere             anywhere            
    3   180 ACCEPT     tcp  --  any    any     anywhere             anywhere             state NEW tcp dpt:ssh
   10   600 REJECT     all  --  any    any     anywhere             anywhere             reject-with icmp-host-prohibited

Chain FORWARD (policy ACCEPT 0 packets, 0 bytes)
 pkts bytes target     prot opt in     out     source               destination         
    0     0 REJECT     all  --  any    any     anywhere             anywhere             reject-with icmp-host-prohibited

Chain OUTPUT (policy ACCEPT 119 packets, 16393 bytes)
 pkts bytes target     prot opt in     out     source               destination         

Chain f2b-SSH (1 references)
 pkts bytes target     prot opt in     out     source               destination         
  290 26487 RETURN     all  --  any    any     anywhere             anywhere            
```

Nếu cấu hình thành công, trên IPTable sẽ xuất hiện dòng
```
311 29167 f2b-SSH    tcp  --  any    any     anywhere             anywhere             tcp dpt:ssh

```

Kiểm tra trạng thái fail2ban
```
[root@tlunginx01 ~]# fail2ban-client status
Status
|- Number of jail:	1
`- Jail list:	sshd
```

Liệt kê danh sách IP bị ban
```
[root@tlunginx01 ~]# fail2ban-client status sshd
Status for the jail: sshd
|- Filter
|  |- Currently failed:	4
|  |- Total failed:	1664
|  `- File list:	/var/log/auth.log
`- Actions
   |- Currently banned:	14
   |- Total banned:	234
   `- Banned IP list:	222.186.52.124 112.85.42.171 122.195.200.148 42.59.249.169 49.88.112.78 222.186.31.144 182.99.115.144 222.186.31.136 153.36.242.143 222.186.42.15 221.167.9.20 49.88.112.90 129.213.172.170 222.186.30.165
```

Trong trường hợp muốn loại ban 1 ip chỉ định
```
fail2ban-client set sshd unbanip <IP_Mong_Muon>

# VD
fail2ban-client set sshd unbanip 127.0.0.1
```

Thêm mới user được phép truy cập SSH
```
useradd nhanhoa
passwd nhanhoa  

# Lưu ý đặt pass cho nhanhoa là: Cloud3652019a@
```

Chỉ cho phép user `nhanhoa` được truy cập thông qua SSH
```
cp /etc/ssh/sshd_config /etc/ssh/sshd_config_bak

sed -i 's|#PermitRootLogin yes|PermitRootLogin no|g' /etc/ssh/sshd_config
sed -i 's|#PermitEmptyPasswords no|PermitEmptyPasswords no|g' /etc/ssh/sshd_config

echo "AllowUsers nhanhoa" >> /etc/ssh/sshd_config
systemctl restart sshd
```

### Triển khai dịch vụ Nginx

Cài đặt Nginx
```
sudo yum install nginx -y
```

Bổ sung cấu hình IPTables
```
sudo iptables  -I INPUT 5 -p tcp -m multiport --dports 80,443 -m conntrack --ctstate NEW,ESTABLISHED -j ACCEPT
sudo iptables -A OUTPUT -p tcp -m multiport --dports 80,443 -m conntrack --ctstate ESTABLISHED -j ACCEPT
```

Lưu câu hình IPtables
```
service iptables save
```

Tunning Nginx
```
systemctl stop nginx

mkdir /etc/systemd/system/nginx.service.d
cat > /etc/systemd/system/nginx.service.d/nofile_limit.conf << EOF
[Service]
LimitNOFILE=100000
EOF
```

Tunning sysctl
```
cat >> /etc/sysctl.conf << EOF
vm.swappiness = 0
vm.max_map_count = 262144


net.ipv4.tcp_wmem = 4096 65536 33554432
net.ipv4.tcp_syn_retries = 3
net.ipv4.tcp_syncookies = 1
net.ipv4.tcp_synack_retries = 3
net.ipv4.tcp_slow_start_after_idle = 0
net.ipv4.tcp_rmem = 4096 87380 33554432
net.ipv4.tcp_max_tw_buckets = 5880000
net.ipv4.tcp_max_syn_backlog = 3240000
net.ipv4.tcp_max_orphans = 262144
net.ipv4.tcp_keepalive_probes = 5
net.ipv4.tcp_keepalive_intvl = 30
net.ipv4.tcp_fin_timeout = 10
net.ipv4.tcp_congestion_control = cubic


net.ipv4.neigh.default.gc_thresh3 = 450560
net.ipv4.neigh.default.gc_thresh2 = 450560
net.ipv4.neigh.default.gc_thresh1 = 225280
net.ipv4.neigh.default.gc_stale_time = 7200


net.ipv4.ip_local_port_range = 1024 65535
net.ipv4.ip_forward = 1


net.ipv4.conf.default.send_redirects = 0
net.ipv4.conf.default.secure_redirects = 0
net.ipv4.conf.default.accept_source_route = 0
net.ipv4.conf.default.accept_source_route = 0
net.ipv4.conf.default.accept_redirects = 0
net.ipv4.conf.all.send_redirects = 0
net.ipv4.conf.all.secure_redirects = 0
net.ipv4.conf.all.rp_filter = 1
net.ipv4.conf.all.log_martians = 0
net.ipv4.conf.all.accept_source_route = 0
net.ipv4.conf.all.accept_redirects = 0


net.core.wmem_max = 67108864
net.core.rmem_max = 67108864
net.core.rmem_default = 67108864
net.core.wmem_default = 67108864


net.ipv4.tcp_sack = 0
net.ipv4.tcp_dsack = 0
net.ipv4.tcp_fack = 0


net.core.netdev_max_backlog = 64000


net.core.default_qdisc = fq


kernel.randomize_va_space = 1
kernel.pid_max = 65536


kernel.msgmnb = 65536
kernel.msgmax = 65536


fs.nr_open = 4000000
fs.file-max = 4000000
EOF
```

Kích hoạt tham số Tunning
```
sysctl -p
```

Kết quả
```
[root@tlunginx01 ~]# sysctl -p
vm.swappiness = 0
vm.max_map_count = 262144
net.ipv4.tcp_wmem = 4096 65536 33554432
net.ipv4.tcp_syn_retries = 3
net.ipv4.tcp_syncookies = 1
net.ipv4.tcp_synack_retries = 3
net.ipv4.tcp_slow_start_after_idle = 0
net.ipv4.tcp_rmem = 4096 87380 33554432
net.ipv4.tcp_max_tw_buckets = 5880000
net.ipv4.tcp_max_syn_backlog = 3240000
net.ipv4.tcp_max_orphans = 262144
net.ipv4.tcp_keepalive_probes = 5
net.ipv4.tcp_keepalive_intvl = 30
net.ipv4.tcp_fin_timeout = 10
net.ipv4.tcp_congestion_control = cubic
net.ipv4.neigh.default.gc_thresh3 = 450560
net.ipv4.neigh.default.gc_thresh2 = 450560
net.ipv4.neigh.default.gc_thresh1 = 225280
net.ipv4.neigh.default.gc_stale_time = 7200
net.ipv4.ip_local_port_range = 1024 65535
net.ipv4.ip_forward = 1
net.ipv4.conf.default.send_redirects = 0
net.ipv4.conf.default.secure_redirects = 0
net.ipv4.conf.default.accept_source_route = 0
net.ipv4.conf.default.accept_source_route = 0
net.ipv4.conf.default.accept_redirects = 0
net.ipv4.conf.all.send_redirects = 0
net.ipv4.conf.all.secure_redirects = 0
net.ipv4.conf.all.rp_filter = 1
net.ipv4.conf.all.log_martians = 0
net.ipv4.conf.all.accept_source_route = 0
net.ipv4.conf.all.accept_redirects = 0
net.core.wmem_max = 67108864
net.core.rmem_max = 67108864
net.core.rmem_default = 67108864
net.core.wmem_default = 67108864
net.ipv4.tcp_sack = 0
net.ipv4.tcp_dsack = 0
net.ipv4.tcp_fack = 0
net.core.netdev_max_backlog = 64000
net.core.default_qdisc = fq
kernel.randomize_va_space = 1
kernel.pid_max = 65536
kernel.msgmnb = 65536
kernel.msgmax = 65536
fs.nr_open = 4000000
fs.file-max = 4000000
```

Cấu hình Nginx
```
cp /etc/nginx/nginx.conf /etc/nginx/nginx.conf.org

cat > /etc/nginx/nginx.conf << EOF
# For more information on configuration, see:
#   * Official English Documentation: http://nginx.org/en/docs/
#   * Official Russian Documentation: http://nginx.org/ru/docs/

user nginx;
worker_processes auto;
# Tunning
worker_rlimit_nofile 100000;

error_log /var/log/nginx/error.log;
pid /run/nginx.pid;

# Load dynamic modules. See /usr/share/nginx/README.dynamic.
include /usr/share/nginx/modules/*.conf;

events {
    # Tunning
    worker_connections 4000;
    use epoll;
    multi_accept on;
}

http {
    # Tunning
    open_file_cache max=200000 inactive=20s;
    open_file_cache_valid 30s;
    open_file_cache_min_uses 2;
    open_file_cache_errors on;
    access_log off;
    client_body_timeout 10;
    keepalive_timeout 30;
    keepalive_requests 100000;

    log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                      '$status $body_bytes_sent "$http_referer" '
                      '"$http_user_agent" "$http_x_forwarded_for"';

    access_log  /var/log/nginx/access.log  main;

    sendfile            on;
    tcp_nopush          on;
    tcp_nodelay         on;
    #keepalive_timeout   65;
    types_hash_max_size 2048;

    include             /etc/nginx/mime.types;
    default_type        application/octet-stream;

    # Load modular configuration files from the /etc/nginx/conf.d directory.
    # See http://nginx.org/en/docs/ngx_core_module.html#include
    # for more information.
    include /etc/nginx/conf.d/*.conf;

    server {
        listen       80 default_server;
        listen       [::]:80 default_server;
        server_name  _;
        root         /usr/share/nginx/html;

        # Load configuration files for the default server block.
        include /etc/nginx/default.d/*.conf;

        location / {
        }

        error_page 404 /404.html;
            location = /40x.html {
        }

        error_page 500 502 503 504 /50x.html;
            location = /50x.html {
        }
    }
}
EOF
```

Khởi chạy dịch vụ Nginx
```
systemctl restart nginx
systemctl enable nginx
```

Tới đây, phần thiết lập Nginx đã thành công, mời bạn chuyển sang bài tiếp theo: [Triển khai Storage](/docs/03-caidat-storage.md)

## Nguồn
https://blog.cloud365.vn/linux/fail2ban-cai-dat-cau-hinh/