# Ứng dụng Nginx làm Reverse Proxy cho Moodle Cluster
---
## Mô hình

### Mô hình đấu nối

![](/images/06-nginx-moodle/mohinh_nginx_moodle.png)

### Yêu cầu
- Cần chuẩn bị cụm Moodle và Nginx qua docs:

[Triển khai Moodle](/docs/05-caidat-moodle.md)

[Triển khai Nginx](/docs/02-caidat-nginx.md)

- Cấn có IP Public cho Nginx và 1 Domain trỏ về

### Lưu ý:
- Node Nginx có hostname `tlunginx01`
- Node Moodle01 có hostname `tlumoodle01`
- Node Moodle02 có hostname `tlumoodle02`
- Node Moodle03 có hostname `tlumoodle03`
- Lưu ý, Node Nginx có IP Public `171.244.3.112`, và mình đã trỏ domain `moodle.devopsviet.com`

![](/images/06-nginx-moodle/dns_moodle_check.png)

Kiểm tra, truy cập `http://171.244.3.112/`

![](/images/06-nginx-moodle/check_nginx_ip_public.png)

## Cấu hình Nginx làm Reverse Proxy cho Moodle

### Mô hình hoạt động

![](/images/06-nginx-moodle/traffic_nginx_moodle.png)

Giải thích:
- Nginx được sử dụng làm `Reverse Proxy` cho cụm, các traffic từ Internet sẽ qua Nginx để vào trong hệ thống Moodle
- Moodle Cluster sử dụng HAProxy làm LB cho cụm Moodle Web phía dưới
- Nginx sẽ điều hướng traffic từ internet xuống HAProxy Moodle thông qua `Moodle IP VIP`, HAProxy Moodle sẽ điều phối traffic xuống các node Moodle Web.

### Thực hiện trên node Nginx
> Hostname `tlunginx01`

Cấu hình VirtualHost
```
cat > /etc/nginx/conf.d/moodle.devopsviet.com.conf << EOF
server {
    server_name moodle.devopsviet.com;
    client_max_body_size 1024M;

        location / {
            proxy_set_header X-Real-IP \$remote_addr;
            proxy_set_header X-Forwarded-For \$proxy_add_x_forwarded_for;
            proxy_set_header Host \$http_host;
            proxy_set_header X-NginX-Proxy true;
            proxy_pass http://192.168.20.30:80;
        }
}
EOF
```

Kết quả
```
[root@tlunginx01 ~]# cat /etc/nginx/conf.d/moodle.devopsviet.com.conf
server {
    server_name moodle.devopsviet.com;
    client_max_body_size 1024M;

        location / {
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header Host $http_host;
            proxy_set_header X-NginX-Proxy true;
            proxy_pass http://192.168.20.30:80;
        }
}
```

Test và Reload lại trang thái Nginx
```
[root@tlunginx01 conf.d]# nginx -t
nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
nginx: configuration file /etc/nginx/nginx.conf test is successful
[root@tlunginx01 conf.d]# nginx -s reload
```

### Thực hiện trên các node Moodle
> Thực hiện trên cả 3 node Moodle01 (`tlumoodle01`), Moodle02 (`tlumoodle02`), Moodle03 (`tlumoodle03`)

File cấu hình mặc định
> Trên Node Moodle01 (`tlumoodle01`)

```
[root@tlumoodle01 ~]# cat /var/www/html/moodle/config.php
<?php  // Moodle configuration file

unset($CFG);
global $CFG;
$CFG = new stdClass();

$CFG->dbtype    = 'mariadb';
$CFG->dblibrary = 'native';
$CFG->dbhost    = '192.168.20.44';
$CFG->dbname    = 'moodle';
$CFG->dbuser    = 'moodleuser';
$CFG->dbpass    = 'Cloud3652019a@';
$CFG->prefix    = 'mdl_';
$CFG->dboptions = array (
  'dbpersist' => 0,
  'dbport' => '',
  'dbsocket' => '',
  'dbcollation' => 'utf8mb4_unicode_ci',
);

$CFG->wwwroot   = 'http://192.168.20.30';
$CFG->dataroot  = '/var/moodledata';
$CFG->admin     = 'admin';

$CFG->directorypermissions = 02777;

require_once(__DIR__ . '/lib/setup.php');

// There is no php closing tag in this file,
// it is intentional because it prevents trailing whitespace problems!
```

Cấu hình lại như sau:
- Comment setting `$CFG->wwwroot`
- Bổ sung setting `$CFG->wwwroot   = 'http://moodle.devopsviet.com';`
```
[root@tlumoodle01 ~]# cat /var/www/html/moodle/config.php
<?php  // Moodle configuration file

unset($CFG);
global $CFG;
$CFG = new stdClass();

$CFG->dbtype    = 'mariadb';
$CFG->dblibrary = 'native';
$CFG->dbhost    = '192.168.20.44';
$CFG->dbname    = 'moodle';
$CFG->dbuser    = 'moodleuser';
$CFG->dbpass    = 'Cloud3652019a@';
$CFG->prefix    = 'mdl_';
$CFG->dboptions = array (
  'dbpersist' => 0,
  'dbport' => '',
  'dbsocket' => '',
  'dbcollation' => 'utf8mb4_unicode_ci',
);

//$CFG->wwwroot   = 'http://192.168.20.30';
$CFG->wwwroot   = 'http://moodle.devopsviet.com';
$CFG->dataroot  = '/var/moodledata';
$CFG->admin     = 'admin';

$CFG->directorypermissions = 02777;

require_once(__DIR__ . '/lib/setup.php');

// There is no php closing tag in this file,
// it is intentional because it prevents trailing whitespace problems!
```

Tương tự trên Moodle02 (`tlumoodle02`)
```
[root@tlumoodle02 ~]# cat /var/www/html/moodle/config.php
<?php  // Moodle configuration file

unset($CFG);
global $CFG;
$CFG = new stdClass();

$CFG->dbtype    = 'mariadb';
$CFG->dblibrary = 'native';
$CFG->dbhost    = '192.168.20.44';
$CFG->dbname    = 'moodle';
$CFG->dbuser    = 'moodleuser';
$CFG->dbpass    = 'Cloud3652019a@';
$CFG->prefix    = 'mdl_';
$CFG->dboptions = array (
  'dbpersist' => 0,
  'dbport' => '',
  'dbsocket' => '',
  'dbcollation' => 'utf8mb4_unicode_ci',
);

//$CFG->wwwroot   = 'http://192.168.20.30';
$CFG->wwwroot   = 'http://moodle.devopsviet.com';
$CFG->dataroot  = '/var/moodledata';
$CFG->admin     = 'admin';

$CFG->directorypermissions = 02777;

require_once(__DIR__ . '/lib/setup.php');

// There is no php closing tag in this file,
// it is intentional because it prevents trailing whitespace problems!
```

Và Moodle03 (`tlumoodle03`)
```
[root@tlumoodle03 ~]# cat /var/www/html/moodle/config.php
<?php  // Moodle configuration file

unset($CFG);
global $CFG;
$CFG = new stdClass();

$CFG->dbtype    = 'mariadb';
$CFG->dblibrary = 'native';
$CFG->dbhost    = '192.168.20.44';
$CFG->dbname    = 'moodle';
$CFG->dbuser    = 'moodleuser';
$CFG->dbpass    = 'Cloud3652019a@';
$CFG->prefix    = 'mdl_';
$CFG->dboptions = array (
  'dbpersist' => 0,
  'dbport' => '',
  'dbsocket' => '',
  'dbcollation' => 'utf8mb4_unicode_ci',
);

//$CFG->wwwroot   = 'http://192.168.20.30';
$CFG->wwwroot   = 'http://moodle.devopsviet.com';
$CFG->dataroot  = '/var/moodledata';
$CFG->admin     = 'admin';

$CFG->directorypermissions = 02777;

require_once(__DIR__ . '/lib/setup.php');

// There is no php closing tag in this file,
// it is intentional because it prevents trailing whitespace problems!
```

Lưu ý:
- Cấu hình `/var/www/html/moodle/config.php` trên 3 node Moodle phải giống nhau

### Kiểm tra

Truy cập đường dẫn `http://moodle.devopsviet.com/`, kết quả

![](/images/06-nginx-moodle/moodle_http.png)

## Cấu hình HTTPS cho Moodle Cluster sử dụng Let's encrypt

### Thực hiện trên Nginx
> Hostname `tlunginx01`

Như giải thích phía trên, Nginx sẽ là node điều phối Request cho cụm, vì vậy để cấu hình HTTPS cho Moodle, mình sẽ chỉ cấu hình Cert SSL trên Nginx.

![](/images/06-nginx-moodle/https_reverse_proxy.png)

Cài đặt certbot
```
yum install certbot-nginx -y
```

Sinh SSL bằng letencrypts
```
certbot --nginx -d moodle.devopsviet.com
```

Kết quả
```
[root@tlunginx01 ~]# certbot --nginx -d moodle.devopsviet.com
Saving debug log to /var/log/letsencrypt/letsencrypt.log
Plugins selected: Authenticator nginx, Installer nginx
Starting new HTTPS connection (1): acme-v02.api.letsencrypt.org
Obtaining a new certificate
Performing the following challenges:
http-01 challenge for moodle.devopsviet.com
Waiting for verification...
Cleaning up challenges
Deploying Certificate to VirtualHost /etc/nginx/conf.d/moodle.devopsviet.com.conf

Please choose whether or not to redirect HTTP traffic to HTTPS, removing HTTP access.
- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
1: No redirect - Make no further changes to the webserver configuration.
2: Redirect - Make all requests redirect to secure HTTPS access. Choose this for
new sites, or if you're confident your site works on HTTPS. You can undo this
change by editing your web server's configuration.
- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
Select the appropriate number [1-2] then [enter] (press 'c' to cancel): 2
Redirecting all traffic on port 80 to ssl in /etc/nginx/conf.d/moodle.devopsviet.com.conf

- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
Congratulations! You have successfully enabled https://moodle.devopsviet.com

You should test your configuration at:
https://www.ssllabs.com/ssltest/analyze.html?d=moodle.devopsviet.com
- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -

IMPORTANT NOTES:
 - Congratulations! Your certificate and chain have been saved at:
   /etc/letsencrypt/live/moodle.devopsviet.com/fullchain.pem
   Your key file has been saved at:
   /etc/letsencrypt/live/moodle.devopsviet.com/privkey.pem
   Your cert will expire on 2019-12-18. To obtain a new or tweaked
   version of this certificate in the future, simply run certbot again
   with the "certonly" option. To non-interactively renew *all* of
   your certificates, run "certbot renew"
 - If you like Certbot, please consider supporting our work by:

   Donating to ISRG / Let's Encrypt:   https://letsencrypt.org/donate
   Donating to EFF:                    https://eff.org/donate-le
```

Kiểm tra
```
[root@tlunginx01 ~]# cat /etc/nginx/conf.d/moodle.devopsviet.com.conf 

server {
    server_name moodle.devopsviet.com;
    client_max_body_size 1024M;

        location / {
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header Host $http_host;
            proxy_set_header X-NginX-Proxy true;
            proxy_pass http://192.168.20.30:80;
        }


    listen 443 ssl; # managed by Certbot
    ssl_certificate /etc/letsencrypt/live/moodle.devopsviet.com/fullchain.pem; # managed by Certbot
    ssl_certificate_key /etc/letsencrypt/live/moodle.devopsviet.com/privkey.pem; # managed by Certbot
    include /etc/letsencrypt/options-ssl-nginx.conf; # managed by Certbot
    ssl_dhparam /etc/letsencrypt/ssl-dhparams.pem; # managed by Certbot

}
server {
    if ($host = moodle.devopsviet.com) {
        return 301 https://$host$request_uri;
    } # managed by Certbot


    server_name moodle.devopsviet.com;
    listen 80;
    return 404; # managed by Certbot


}
```

Test và Reload lại trang thái Nginx
```
[root@tlunginx01 conf.d]# nginx -t
nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
nginx: configuration file /etc/nginx/nginx.conf test is successful
[root@tlunginx01 conf.d]# nginx -s reload
```

Cấu hình tự động renew cert letencrypt
```
yum install wget -y
wget https://dl.eff.org/certbot-auto && chmod a+x certbot-auto
sudo mv certbot-auto /etc/letsencrypt/

echo "0 2 * * 6 cd /etc/letsencrypt/ && ./certbot-auto renew && systemctl restart nginx" >> /etc/crontab
systemctl restart crond
```

### Thực hiện trên các node Moodle
> Thực hiện trên cả 3 node Moodle01 (`tlumoodle01`), Moodle02 (`tlumoodle02`), Moodle03 (`tlumoodle03`)

Bố sung thêm settings sau vào file config `/var/www/html/moodle/config.php`:
- Thêm `$CFG->sslproxy = true;`,
- Sửa `$CFG->wwwroot   = 'http://moodle.devopsviet.com';` thành `$CFG->wwwroot   = 'https://moodle.devopsviet.com';`

Trên node Moodle01 (`tlumoodle01`) 
```
[root@tlumoodle01 ~]# cat /var/www/html/moodle/config.php
<?php  // Moodle configuration file

unset($CFG);
global $CFG;
$CFG = new stdClass();

$CFG->dbtype    = 'mariadb';
$CFG->dblibrary = 'native';
$CFG->dbhost    = '192.168.20.44';
$CFG->dbname    = 'moodle';
$CFG->dbuser    = 'moodleuser';
$CFG->dbpass    = 'Cloud3652019a@';
$CFG->prefix    = 'mdl_';
$CFG->dboptions = array (
  'dbpersist' => 0,
  'dbport' => '',
  'dbsocket' => '',
  'dbcollation' => 'utf8mb4_unicode_ci',
);

$CFG->wwwroot   = 'https://moodle.devopsviet.com';
$CFG->dataroot  = '/var/moodledata';
$CFG->admin     = 'admin';
$CFG->sslproxy = true;


$CFG->directorypermissions = 02777;

require_once(__DIR__ . '/lib/setup.php');

// There is no php closing tag in this file,
// it is intentional because it prevents trailing whitespace problems!
```

Trên node Moodle02 (`tlumoodle02`) 
```
[root@tlumoodle02 ~]# cat /var/www/html/moodle/config.php
<?php  // Moodle configuration file

unset($CFG);
global $CFG;
$CFG = new stdClass();

$CFG->dbtype    = 'mariadb';
$CFG->dblibrary = 'native';
$CFG->dbhost    = '192.168.20.44';
$CFG->dbname    = 'moodle';
$CFG->dbuser    = 'moodleuser';
$CFG->dbpass    = 'Cloud3652019a@';
$CFG->prefix    = 'mdl_';
$CFG->dboptions = array (
  'dbpersist' => 0,
  'dbport' => '',
  'dbsocket' => '',
  'dbcollation' => 'utf8mb4_unicode_ci',
);

$CFG->wwwroot   = 'https://moodle.devopsviet.com';
$CFG->dataroot  = '/var/moodledata';
$CFG->admin     = 'admin';
$CFG->sslproxy = true;


$CFG->directorypermissions = 02777;

require_once(__DIR__ . '/lib/setup.php');

// There is no php closing tag in this file,
// it is intentional because it prevents trailing whitespace problems!
```

Trên node Moodle03 (`tlumoodle03`) 
```
[root@tlumoodle03 ~]# cat /var/www/html/moodle/config.php
<?php  // Moodle configuration file

unset($CFG);
global $CFG;
$CFG = new stdClass();

$CFG->dbtype    = 'mariadb';
$CFG->dblibrary = 'native';
$CFG->dbhost    = '192.168.20.44';
$CFG->dbname    = 'moodle';
$CFG->dbuser    = 'moodleuser';
$CFG->dbpass    = 'Cloud3652019a@';
$CFG->prefix    = 'mdl_';
$CFG->dboptions = array (
  'dbpersist' => 0,
  'dbport' => '',
  'dbsocket' => '',
  'dbcollation' => 'utf8mb4_unicode_ci',
);

$CFG->wwwroot   = 'https://moodle.devopsviet.com';
$CFG->dataroot  = '/var/moodledata';
$CFG->admin     = 'admin';
$CFG->sslproxy = true;


$CFG->directorypermissions = 02777;

require_once(__DIR__ . '/lib/setup.php');

// There is no php closing tag in this file,
// it is intentional because it prevents trailing whitespace problems!
```

Lưu ý:
- Với bản moodle thấp hơn (từ 3.6 trở xuống), cần thêm settings `$CFG->reverseproxy = true;`

### Kiểm tra

Truy cập `https://moodle.devopsviet.com/`

![](/images/06-nginx-moodle/moodle_https.png)

Thành công.
