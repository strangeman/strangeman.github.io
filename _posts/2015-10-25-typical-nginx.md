---
layout: post
title: "Типовой конфиг сайта на связке NGINX + PHP-FPM"
categories: nginx php
description: "Типовой конфиг сайта на связке NGINX + PHP-FPM"
keywords: "nginx, php, php-fpm"
---

Пути файлов в зависимости от ОС, версии ОС и используемого софта (web-панели типа WHM или Parallels Plesk) могут очень сильно варьироваться, поэтому я их тут не указываю. Опции, в зависимости от конкретного сайта и сервера (особенно это касается PHP-FPM: используемого process manager и его настроек) также могут варьироваться в широких пределах. Здесь приведен более-менее дефолтный конфиг, с которым все должно “просто работать”, а производительность сервера можно тюнить уже потом.

Установки я не касаюсь ровно по той же причине. Если с nginx чаще всего все довольно просто, то с PHP-FPM каждый случай индивидуален в зависимости от используемого софта.

## NGINX Vhost
Возможные пути:

```
/etc/nginx/sites-enabled/example.conf
/etc/nginx/conf.d/example.conf
/etc/nginx/conf/example.conf
/etc/nginx/plesk.conf.d/vhosts/example.conf
```

можно попробовать посмотреть командой `nginx -V`, флаг `--conf-path`, затем уже смотреть инклуды в основном конфиге.

```
server {
    listen 80;

    root /srv/example.com;
    server_name example.com;

    error_log /var/log/nginx/example.com.error_log;
    access_log /var/log/nginx/example.com.access_log;

    index index.php;

    location = /favicon.ico {
   log_not_found off;
            access_log off;
    }

    location = /robots.txt {
            allow all;
            log_not_found off;
            access_log off;
    }

    location / {
            # This is cool because no php is touched for static content.
            # include the "?$args" part so non-default permalinks doesn't break when using query string
            try_files $uri $uri/ /index.php?$args;

    }

    location ~ \.php$ {
         # Filter out arbitrary code execution
        location       ~ \..*/.*\.php$ {return 404;}

        fastcgi_index index.php;
        include        fastcgi_params;
                fastcgi_pass unix:/var/run/php5-fpm.example.sock;
                fastcgi_split_path_info ^(.+\.php)(/.+)$;
    }

    location ~* ^.+\.(jpg|jpeg|gif|png|svg|js|css|mp3|ogg|mpe?g|avi|zip|gz|bz2?|rar|swf)$ {
            expires max;
            log_not_found off;
    }
}
```

## PHP-FPM pool definition
Возможные пути:

```
/etc/php5/fpm/pool.d/example.conf
/etc/php-fpm/pool.d/example.conf
/etc/php5-fpm/pool.d/example.conf
/usr/local/etc/php-fpm.conf
/usr/local/etc/php5/fpm/pool.d/example.conf
```

```
; Start a new pool named 'example'.
; the variable $pool can we used in any directive and will be replaced by the
; pool name ('example' here)
[example]

; Unix user/group of processes
; Note: The user is mandatory. If the group is not set, the default user's group
;   will be used.
user = www-data
group = www-data

listen = /var/run/php5-fpm.example.sock

listen.owner = www-data
listen.group = www-data

pm = ondemand
pm.max_children = 20

; Chdir to this directory at the start.
; Note: relative path can be used.
; Default Value: current directory or / when chroot
chdir = /
```