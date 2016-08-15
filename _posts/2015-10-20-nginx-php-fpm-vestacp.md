---
layout: post
categories: nginx php
title: "Автоматическая настройка Nginx+PHP-FPM в VestaCP"
description: "Автоматическая настройка Nginx+PHP-FPM в VestaCP"
keywords: "nginx, php, php-fpm, vestacp"
---

Неактуально по причине выхода новой Весты, которая поддерживает PHP-FPM из коробки.

## Пререквизиты:

* Ubuntu 14.04
* VestaCP 0.9.8

Хочется, чтобы при добавлении домена через веб-морду VestaCP автоматически создавался пул PHP-FPM.

TODO: Реализовать удаление пула из конфига при удалении домена. :)

## For nginx

Основано вот на этом: https://github.com/autronix/Vesta-0.9.8-with-nginx-only

Но я внес кое-какие правки под себя:


```
# cat /usr/local/vesta/data/templates/web/nginx/default.tpl

server {
    listen      %proxy_port%;
    server_name %domain_idn% %alias_idn%;
    error_log  /var/log/apache2/domains/%domain%.error.log error;

    root           %docroot%;
    index index.php index.html index.htm;

    location / {
        try_files $uri $uri/ /index.php?$args;
    }

    include %home%/%user%/conf/web/nginx.%domain%.conf*;

    location ~ \.php$ {
        try_files $uri =404;
        fastcgi_split_path_info ^(.+\.php)(/.+)$;
        fastcgi_pass  unix:/var/run/php5-fpm.%domain_idn%.sock;
        fastcgi_index index.php;
        include fastcgi_params;
    }

    location ~ /\.ht    {return 404;}
    location ~ /\.svn/  {return 404;}
    location ~ /\.git/  {return 404;}
    location ~ /\.hg/   {return 404;}
    location ~ /\.bzr/  {return 404;}

}

# cat /usr/local/vesta/data/templates/web/nginx/default.stpl

server {
    listen      %proxy_ssl_port%;
    server_name %domain_idn% %alias_idn%;
    ssl         on;
    ssl_certificate      %ssl_pem%;
    ssl_certificate_key  %ssl_key%;
    error_log  /var/log/apache2/domains/%domain%.error.log error;

    root           %docroot%;
    index index.php index.html index.htm;

    location / {
        try_files $uri $uri/ /index.php?$args;
    }

    include %home%/%user%/conf/web/nginx.%domain%.conf*;

    location ~ \.php$ {
        try_files $uri =404;
        fastcgi_split_path_info ^(.+\.php)(/.+)$;
        fastcgi_pass  unix:/var/run/php5-fpm.%domain_idn%.sock;
        fastcgi_index index.php;
        include fastcgi_params;
    }

    location ~ /\.ht    {return 404;}
    location ~ /\.svn/  {return 404;}
    location ~ /\.git/  {return 404;}
    location ~ /\.hg/   {return 404;}
    location ~ /\.bzr/  {return 404;}

}
```

## For PHP-FPM

Устанавливаем php5-fpm

```
apt install php5-fpm
```

Прописываем, где искать инклуды конфигов:

```
# tail -n 1 /etc/php5/fpm/php-fpm.conf
include=/home/*/conf/web/php_pool.conf
```

Прописываем шаблон для конфига:

```
# cat /usr/local/vesta/data/templates/web/nginx/phpfpm.tpl
 
[%domain_idn%]
 
user = %user%
group = %group%
 
listen = /var/run/php5-fpm.$pool.sock
 
listen.owner = %user%
listen.group = %group%
;listen.mode = 0660
 
; Choose how the process manager will control the number of child processes.
; Possible Values:
;   static  - a fixed number (pm.max_children) of child processes;
;   dynamic - the number of child processes are set dynamically based on the
;             following directives. With this process management, there will be
;             always at least 1 children.
;             pm.max_children      - the maximum number of children that can
;                                    be alive at the same time.
;             pm.start_servers     - the number of children created on startup.
;             pm.min_spare_servers - the minimum number of children in 'idle'
;                                    state (waiting to process). If the number
;                                    of 'idle' processes is less than this
;                                    number then some children will be created.
;             pm.max_spare_servers - the maximum number of children in 'idle'
;                                    state (waiting to process). If the number
;                                    of 'idle' processes is greater than this
;                                    number then some children will be killed.
;  ondemand - no children are created at startup. Children will be forked when
;             new requests will connect. The following parameter are used:
;             pm.max_children           - the maximum number of children that
;                                         can be alive at the same time.
;             pm.process_idle_timeout   - The number of seconds after which
;                                         an idle process will be killed.
; Note: This value is mandatory.
pm = ondemand
 
pm.max_children = 20
pm.start_servers = 2
pm.min_spare_servers = 1
pm.max_spare_servers = 3
pm.process_idle_timeout = 10s
pm.max_requests = 500
```

Вносим правку в скрипт Vesta CP для добавления нового веб-прокси

```
# diff -p /usr/local/vesta/bin/v-add-web-domain-proxy /usr/local/vesta_fixed/bin/v-add-web-domain-proxy -c5
*** /usr/local/vesta/bin/v-add-web-domain-proxy    2015-06-05 02:00:50.000000000 +1000
--- /usr/local/vesta_fixed/bin/v-add-web-domain-proxy    2015-10-15 10:45:52.162316790 +1000
*************** if [ "$SSL" = 'yes' ]; then
*** 84,93 ****
--- 84,100 ----
      if [ -z "$(grep "$conf" $proxy_conf)" ]; then
          echo "include $conf;"&gt;&gt; $proxy_conf
      fi
  fi
 
+ # Add PHP-fpm pool
+ tpl_file="$WEBTPL/$PROXY_SYSTEM/phpfpm.tpl"
+ conf="$HOMEDIR/$user/conf/web/php_pool.conf"
+ add_web_config
+ service php5-fpm restart
+
+
  # Running template trigger
  if [ -x $WEBTPL/$PROXY_SYSTEM/$template.sh ]; then
      $WEBTPL/$PROXY_SYSTEM/$template.sh $user $domain $ip $HOMEDIR $docroot
  fi
```

**Важный нюанс: sock-файл для общения между Nginx и PHP-FPM у нас создается с правами 0660 от имени пользователя. Соответственно Nginx должен запускаться от пользователя, состоящего в той же группе, что и пользователь (ну или наоборот, пользователь должен быть быть в группе nginx).**

Собственно все. Рестартим Vesta, Nginx, PHP-FPM, добавляем новый домен через веб-морду, проверяем.

```
service vesta restart
service php5-fpm restart
service nginx restart
```