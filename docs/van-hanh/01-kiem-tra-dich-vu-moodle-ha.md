# Hướng dẫn kiểm tra cụm HA Moodle

## Phần 1: Kiểm tra HAProxy Stats Page

http://192.168.20.30:8080/stats

Các trường hợp

### Cụm down:

![](/images/van-hanh/01-kiem-tra-dich-vu-moodle-ha/pic1.png)

- Trường hợp cả 3 node cùng down

### Cụm chạy không ổn định

![](/images/van-hanh/01-kiem-tra-dich-vu-moodle-ha/pic2.png)

- Chỉ có 1 node hoạt động (Moodle01)

### Cụm chạy ổn định

![](/images/van-hanh/01-kiem-tra-dich-vu-moodle-ha/pic3.png)

- Cả 3 node đều chạy

## Phần 2: Kiểm tra dịch vụ moodle trên từng node
> Bảo đảm dịch vụ Moodle trên từng node chạy ổn định.

Bước 1:
- Loại bỏ dịch vụ Apache ra khỏi PCS (Pacemaker)

```
pcs resource ban Web_Cluster-clone tlumoodle01
pcs resource ban Web_Cluster-clone tlumoodle02
pcs resource ban Web_Cluster-clone tlumoodle03
```

Bước 2:
- Bật dịch vụ Apache trên từng node
- LƯU Ý: Chỉ chạy Apache trên 1 node trong 1 thời điểm.
- Kiểm tra 3 thao tác: Đăng nhập, truy cập Course, down file bất kỳ từ moodle
- Nếu cả 3 thao tác bình thường => Node Moodle chạy ổn định

Khởi động dịch vụ Apache bằng tay
```
systemctl start httpd
```

Ví dụ
```
[root@tlumoodle01 ~]# systemctl start httpd
[root@tlumoodle02 ~]# systemctl start httpd
[root@tlumoodle03 ~]# systemctl start httpd
```

Bước 3: Khi test xong, cho phép PCS (Pacemaker) kiểm soát resource Http

Kiểm tra, tại thời điểm cả 3 node đang ở trạng thái `Stopped`
```
[root@tlumoodle01 ~]# pcs status
Cluster name: ha_cluster
Stack: corosync
Current DC: tlumoodle01 (version 1.1.19-8.el7_6.4-c3c624ea3d) - partition with quorum
Last updated: Thu Oct 10 09:34:32 2019
Last change: Mon Oct  7 17:56:04 2019 by root via cibadmin on tlumoodle01

3 nodes configured
5 resources configured

Online: [ tlumoodle01 tlumoodle02 tlumoodle03 ]

Full list of resources:

 Virtual_IP	(ocf::heartbeat:IPaddr2):	Started tlumoodle01
 Loadbalancer_HaProxy	(systemd:haproxy):	Started tlumoodle01
 Clone Set: Web_Cluster-clone [Web_Cluster] (unique)
     Web_Cluster:0	(systemd:httpd):	Stopped
     Web_Cluster:1	(systemd:httpd):	Stopped
     Web_Cluster:2	(systemd:httpd):	Stopped

Daemon Status:
  corosync: active/enabled
  pacemaker: active/enabled
  pcsd: active/enabled
```

Kiểm tra ràng buộc
```
[root@tlumoodle01 ~]# pcs constraint
Location Constraints:
  Resource: Web_Cluster-clone
    Disabled on: tlumoodle01 (score:-INFINITY) (role: Started)
    Disabled on: tlumoodle03 (score:-INFINITY) (role: Started)
    Disabled on: tlumoodle02 (score:-INFINITY) (role: Started)
Ordering Constraints:
  start Virtual_IP then start Web_Cluster-clone (kind:Mandatory)
  start Virtual_IP then start Loadbalancer_HaProxy (kind:Mandatory)
Colocation Constraints:
  Virtual_IP with Loadbalancer_HaProxy (score:INFINITY)
Ticket Constraints:
```

Cho phép resource chạy trong pcs
```
pcs resource clear Web_Cluster-clone tlumoodle01
pcs resource clear Web_Cluster-clone tlumoodle02
pcs resource clear Web_Cluster-clone tlumoodle03
```

Kiểm tra lại, tài thời điểm resource `Web_Cluster-clone` đã chạy trên cả 3 node
```
[root@tlumoodle01 ~]# pcs status
Cluster name: ha_cluster
Stack: corosync
Current DC: tlumoodle01 (version 1.1.19-8.el7_6.4-c3c624ea3d) - partition with quorum
Last updated: Thu Oct 10 09:35:59 2019
Last change: Thu Oct 10 09:35:55 2019 by root via crm_resource on tlumoodle01

3 nodes configured
5 resources configured

Online: [ tlumoodle01 tlumoodle02 tlumoodle03 ]

Full list of resources:

 Virtual_IP	(ocf::heartbeat:IPaddr2):	Started tlumoodle01
 Loadbalancer_HaProxy	(systemd:haproxy):	Started tlumoodle01
 Clone Set: Web_Cluster-clone [Web_Cluster] (unique)
     Web_Cluster:0	(systemd:httpd):	Started tlumoodle01
     Web_Cluster:1	(systemd:httpd):	Started tlumoodle02
     Web_Cluster:2	(systemd:httpd):	Started tlumoodle03

Daemon Status:
  corosync: active/enabled
  pacemaker: active/enabled
  pcsd: active/enabled
```


## Phần 3: Kiểm tra dịch vụ moodle khi chạy đồng thời 3 node

Câu lệnh
```
curl -I <IP NODE MOODLE>
```

Ví dụ
```
[root@tlumoodle01 ~]# curl -I 192.168.20.31
HTTP/1.1 303 See Other
Date: Thu, 10 Oct 2019 02:38:39 GMT
Server: Apache/2.4.6 (CentOS) PHP/7.2.22
X-Powered-By: PHP/7.2.22
Location: https://moodledev.thanglong.edu.vn
Content-Language: en
Content-Type: text/html; charset=UTF-8

[root@tlumoodle01 ~]# curl -I 192.168.20.32
HTTP/1.1 303 See Other
Date: Thu, 10 Oct 2019 02:42:00 GMT
Server: Apache/2.4.6 (CentOS) PHP/7.2.22
X-Powered-By: PHP/7.2.22
Location: https://moodledev.thanglong.edu.vn
Content-Language: en
Content-Type: text/html; charset=UTF-8

[root@tlumoodle01 ~]# curl -I 192.168.20.33
HTTP/1.1 303 See Other
Date: Thu, 10 Oct 2019 02:42:09 GMT
Server: Apache/2.4.6 (CentOS) PHP/7.2.22
X-Powered-By: PHP/7.2.22
Location: https://moodledev.thanglong.edu.vn
Content-Language: en
Content-Type: text/html; charset=UTF-8
```
- Nếu status code bằng `HTTP/1.1 303 See Other` + `Location: https://moodledev.thanglong.edu.vn` => Node hoạt động bình thường
- Lưu ý: Node hoạt động bình thường hay không phải bảo đảm đã thực hiện `Phần 2: Kiểm tra dịch vụ moodle trên từng node`

## Phần 4: Ví dụ khi 1 node trong cụm Moodle hoạt động không bình thường

Trạng thái trên HAProxy

![](/images/van-hanh/01-kiem-tra-dich-vu-moodle-ha/pic4.png)

- Node `moodle03` gặp vấn đề

Kiểm tra trển pcs
```
[root@tlumoodle01 ~]# pcs status
Cluster name: ha_cluster
Stack: corosync
Current DC: tlumoodle01 (version 1.1.19-8.el7_6.4-c3c624ea3d) - partition with quorum
Last updated: Thu Oct 10 09:47:36 2019
Last change: Thu Oct 10 09:35:55 2019 by root via crm_resource on tlumoodle01

3 nodes configured
5 resources configured

Online: [ tlumoodle01 tlumoodle02 tlumoodle03 ]

Full list of resources:

 Virtual_IP	(ocf::heartbeat:IPaddr2):	Started tlumoodle01
 Loadbalancer_HaProxy	(systemd:haproxy):	Started tlumoodle01
 Clone Set: Web_Cluster-clone [Web_Cluster] (unique)
     Web_Cluster:0	(systemd:httpd):	Started tlumoodle01
     Web_Cluster:1	(systemd:httpd):	Started tlumoodle02
     Web_Cluster:2	(systemd:httpd):	FAILED tlumoodle03

Failed Actions:
* Web_Cluster:2_monitor_5000 on tlumoodle03 'OCF_PENDING' (196): call=26, status=complete, exitreason='',
    last-rc-change='Thu Oct 10 09:47:27 2019', queued=0ms, exec=0ms


Daemon Status:
  corosync: active/enabled
  pacemaker: active/enabled
  pcsd: active/enabled
```

- Tại thời điểm node `tlumoodle03` đã chuyển sang trạng thái `FAILED`

Xử lý:
- Loại bỏ node `tlumoodle03` khởi cụm

```
pcs resource ban Web_Cluster-clone tlumoodle03
```

Kết quả
```
[root@tlumoodle01 ~]# pcs resource ban Web_Cluster-clone tlumoodle03

[root@tlumoodle01 ~]# pcs status
Cluster name: ha_cluster
Stack: corosync
Current DC: tlumoodle01 (version 1.1.19-8.el7_6.4-c3c624ea3d) - partition with quorum
Last updated: Thu Oct 10 09:49:10 2019
Last change: Thu Oct 10 09:49:06 2019 by root via crm_resource on tlumoodle01

3 nodes configured
5 resources configured

Online: [ tlumoodle01 tlumoodle02 tlumoodle03 ]

Full list of resources:

 Virtual_IP	(ocf::heartbeat:IPaddr2):	Started tlumoodle01
 Loadbalancer_HaProxy	(systemd:haproxy):	Started tlumoodle01
 Clone Set: Web_Cluster-clone [Web_Cluster] (unique)
     Web_Cluster:0	(systemd:httpd):	Started tlumoodle01
     Web_Cluster:1	(systemd:httpd):	Started tlumoodle02
     Web_Cluster:2	(systemd:httpd):	Stopped

Failed Actions:
* Web_Cluster:2_monitor_5000 on tlumoodle03 'OCF_PENDING' (196): call=26, status=complete, exitreason='',
    last-rc-change='Thu Oct 10 09:47:27 2019', queued=0ms, exec=0ms


Daemon Status:
  corosync: active/enabled
  pacemaker: active/enabled
  pcsd: active/enabled

```

- Sau khi dịch vụ `Web_Cluster-clone` trên `tlumoodle03` đã stop, truy cập `tlumoodle03` khởi động dịch vụ apache bằng tay

```
[root@tlumoodle03 moodle]# systemctl start httpd
[root@tlumoodle03 moodle]# systemctl status httpd
● httpd.service - The Apache HTTP Server
   Loaded: loaded (/usr/lib/systemd/system/httpd.service; enabled; vendor preset: disabled)
  Drop-In: /etc/systemd/system/httpd.service.d
           └─nofile_limit.conf
   Active: active (running) since T5 2019-10-10 09:50:42 +07; 8s ago
     Docs: man:httpd(8)
           man:apachectl(8)
  Process: 9813 ExecStop=/bin/kill -WINCH ${MAINPID} (code=exited, status=0/SUCCESS)
 Main PID: 10044 (httpd)
   Status: "Processing requests..."
   CGroup: /system.slice/httpd.service
           ├─10044 /usr/sbin/httpd -DFOREGROUND
           ├─10045 /usr/sbin/httpd -DFOREGROUND
           ├─10046 /usr/sbin/httpd -DFOREGROUND
           ├─10048 /usr/sbin/httpd -DFOREGROUND
           ├─10051 /usr/sbin/httpd -DFOREGROUND
           ├─10052 /usr/sbin/httpd -DFOREGROUND
           ├─10053 /usr/sbin/httpd -DFOREGROUND
           ├─10057 /usr/sbin/httpd -DFOREGROUND
           ├─10058 /usr/sbin/httpd -DFOREGROUND
           ├─10059 /usr/sbin/httpd -DFOREGROUND
           ├─10061 /usr/sbin/httpd -DFOREGROUND
           ├─10062 /usr/sbin/httpd -DFOREGROUND
           ├─10063 /usr/sbin/httpd -DFOREGROUND
           ├─10064 /usr/sbin/httpd -DFOREGROUND
           ├─10065 /usr/sbin/httpd -DFOREGROUND
           ├─10066 /usr/sbin/httpd -DFOREGROUND
           ├─10067 /usr/sbin/httpd -DFOREGROUND
           ├─10068 /usr/sbin/httpd -DFOREGROUND
           ├─10069 /usr/sbin/httpd -DFOREGROUND
           ├─10070 /usr/sbin/httpd -DFOREGROUND
           ├─10071 /usr/sbin/httpd -DFOREGROUND
           ├─10072 /usr/sbin/httpd -DFOREGROUND
           ├─10073 /usr/sbin/httpd -DFOREGROUND
           ├─10074 /usr/sbin/httpd -DFOREGROUND
           ├─10075 /usr/sbin/httpd -DFOREGROUND
           ├─10076 /usr/sbin/httpd -DFOREGROUND
           ├─10077 /usr/sbin/httpd -DFOREGROUND
           └─10078 /usr/sbin/httpd -DFOREGROUND

Th10 10 09:50:42 tlumoodle03 systemd[1]: Starting The Apache HTTP Server...
Th10 10 09:50:42 tlumoodle03 httpd[10044]: AH00558: httpd: Could not reliably determine the server's fully qualified domain name, using 192.168.20.33. Set the 'ServerName' directive globally to ...s this message
Th10 10 09:50:42 tlumoodle03 systemd[1]: Started The Apache HTTP Server.
Hint: Some lines were ellipsized, use -l to show in full.
```

Kiểm tra trạng thái dịch vụ moodle
```
[root@tlumoodle01 ~]# curl -I 192.168.20.33
HTTP/1.1 503 Service Unavailable
Date: Thu, 10 Oct 2019 02:51:18 GMT
Server: Apache/2.4.6 (CentOS) PHP/7.2.22
X-Powered-By: PHP/7.2.22
Connection: close
Content-Type: text/html; charset=UTF-8
```

Kiểm tra qua browser
```
http://192.168.20.33/
```

Kết quả

![](/images/van-hanh/01-kiem-tra-dich-vu-moodle-ha/pic5.png)

Lý do:
- Do config.php chưa chính xác (`/var/www/html/moodle/config.php`)

Cách fix:
- Chỉnh sửa lại config, bảo đảm mọi thứ chạy chính xác trên tlumoodle03

Khi fix thành công

Kiểm tra
```
[root@tlumoodle01 ~]# curl -I 192.168.20.33
HTTP/1.1 303 See Other
Date: Thu, 10 Oct 2019 02:57:16 GMT
Server: Apache/2.4.6 (CentOS) PHP/7.2.22
X-Powered-By: PHP/7.2.22
Location: https://moodledev.thanglong.edu.vn
Content-Language: en
Content-Type: text/html; charset=UTF-8
```

Kiểm tra Haproxy Status

![](/images/van-hanh/01-kiem-tra-dich-vu-moodle-ha/pic6.png)

- Tại thời điểm HAProxy sẽ cho phép node `moodle03` xử lý request.

Cho phép dịch vụ Apache chạy trọng PCS
```
pcs resource clear Web_Cluster-clone tlumoodle03
```

Kiểm tra 
```
[root@tlumoodle01 ~]# pcs status
Cluster name: ha_cluster
Stack: corosync
Current DC: tlumoodle01 (version 1.1.19-8.el7_6.4-c3c624ea3d) - partition with quorum
Last updated: Thu Oct 10 10:00:23 2019
Last change: Thu Oct 10 10:00:19 2019 by root via crm_resource on tlumoodle01

3 nodes configured
5 resources configured

Online: [ tlumoodle01 tlumoodle02 tlumoodle03 ]

Full list of resources:

 Virtual_IP	(ocf::heartbeat:IPaddr2):	Started tlumoodle01
 Loadbalancer_HaProxy	(systemd:haproxy):	Started tlumoodle01
 Clone Set: Web_Cluster-clone [Web_Cluster] (unique)
     Web_Cluster:0	(systemd:httpd):	Started tlumoodle01
     Web_Cluster:1	(systemd:httpd):	Started tlumoodle02
     Web_Cluster:2	(systemd:httpd):	Started tlumoodle03

Failed Actions:
* Web_Cluster:2_monitor_5000 on tlumoodle03 'OCF_PENDING' (196): call=26, status=complete, exitreason='',
    last-rc-change='Thu Oct 10 09:47:27 2019', queued=0ms, exec=0ms


Daemon Status:
  corosync: active/enabled
  pacemaker: active/enabled
  pcsd: active/enabled
```

Tại thời điểm dịch vụ Apache trên tất cả các node đã được quản lý bởi PCS
