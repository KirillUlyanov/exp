#### Задание 1. Восстановить работу сайта my-site.ru . Сервер был удален и восстановлен, файлы размещены, но возникает ошибка «404 Not Found».

Решение проблемы  
**\*** После восстановления сервера был изменен IP сервера, по этому потребовалось обновить www-домен с указанием правильных, существующих IP адресов.  
**\*** Ошибка «404 Not Found» сменилась на ошибку «403 Forbidden». В журналах ошибка:  
```
[Thu Jan 21 10:55:23.315690 2021] [core:crit] [pid 1592] (13)Permission denied: [client 85.113.39.54:37760] AH00529: /var/www/www-root/data/www/.htaccess pcfg_openfile: unable to check htaccess file, ensure it is readable and that '/var/www/www-root/data/www/' is executable
```
Нужно исправить права доступа на `www` и `www/my-site.ru`

**\*** Сайт не может подключиться к базе данных с ошибкой «Ошибка установки соединения с базой данных». Данные, указанные в конфиге сайта, неправильные:
```
root@194-67-91-136:~# mysql -umysite -p'G1a1D3r5!'
mysql: [Warning] Using a password on the command line interface can be insecure.
ERROR 1045 (28000): Access denied for user 'mysite'@'localhost' (using password: YES)
```

При открытии раздела «Базы данных» возникает ошибка `Не удалось подключиться к базе данных '' Access denied for user 'root'@'localhost' (using password: NO)` – проблема вызвана отсутствием пароля в разделе «Серверы баз данных». На сервере так же нет файла с паролем root-а:
```
root@194-67-91-136:~# cat ~/.my.cnf
cat: /root/.my.cnf: No such file or directory
root@194-67-91-136:~# 
```

Решения проблемы два:  
1\. Сбросить пароль пользователя root для mysql в безопасном режиме;  
2\. Подключиться к mysql под системным пользователем ( данные из конфига /etc/mysql/debian.cnf ) и сбросить пароль пользователя root
После этого нужно поправить данные подключения к СУБД в разделе «Серверы баз данных» и продолжить восстановление работы сайта. 

---
##### Сброс пароля mysql в безопасном режиме

https://deivydas.com/how-to-reset-the-mysql-5-7-root-password-in-ubuntu-16-04-lts/

```
service mysql stop
mkdir /var/run/mysqld
chown mysql: /var/run/mysqld
mysqld_safe --skip-grant-tables --skip-networking &
mysql -uroot mysql
UPDATE mysql.user SET authentication_string=PASSWORD('Pa$$w0rd') WHERE User='root';
FLUSH PRIVILEGES;
EXIT;

-- завершить работу mysql любым удобным способом
-- запустить службу mysql в обычном режиме 
```
---
##### Сброс пароля mysql под системным пользователем

```
mysql -udebian-sys-maint -p4M5rybHlZc0dHxm4 mysql
UPDATE mysql.user SET authentication_string=PASSWORD('Pa$$w0rd') WHERE User='root';
FLUSH PRIVILEGES;
EXIT;
```
---

После правки `wp-config.php`, сайт начинает редиректить на dev.local . Меняем адрес сайта в базе:

```
mysql> SELECT * FROM wp_options WHERE option_name IN ('siteurl', 'home');
+-----------+-------------+------------------+----------+
| option_id | option_name | option_value     | autoload |
+-----------+-------------+------------------+----------+
|         2 | home        | http://dev.local | yes      |
|         1 | siteurl     | http://dev.local | yes      |
+-----------+-------------+------------------+----------+
2 rows in set (0.01 sec)

mysql> UPDATE wp_options SET option_value = 'http://my-site.ru' WHERE option_name IN ('siteurl', 'home');
Query OK, 2 rows affected (0.01 sec)
Rows matched: 2  Changed: 2  Warnings: 0

mysql> SELECT * FROM wp_options WHERE option_name IN ('siteurl', 'home');
+-----------+-------------+-------------------+----------+
| option_id | option_name | option_value      | autoload |
+-----------+-------------+-------------------+----------+
|         2 | home        | http://my-site.ru | yes      |
|         1 | siteurl     | http://my-site.ru | yes      |
+-----------+-------------+-------------------+----------+
2 rows in set (0.00 sec)
```

Сайт продолжает редиректить на dev.local . Проверяю файл `wp-config.php` и вижу:
```
define('DOMAIN_CURRENT_SITE', 'dev.local');
```
Меняю домен на нужный:
```
define('DOMAIN_CURRENT_SITE', 'my-site.ru');
```

Вновь возникает ошибка покдлючения к базе на сайте, без редиректа. Нужен стрейс, в котором можно найти:
```
11:15:14 sendto(5, "z\0\0\0\3SELECT  wp_blogs.blog_id FROM wp_blogs  WHERE domain = 'my-site.ru' AND path = '/'  ORDER BY wp_blogs.blog_id ASC LIMIT 1", 126, MSG_DONTWAIT, NULL, 0) = 126
```

В базе домен вообще не указан:
```
mysql> SELECT * FROM wp_blogs;
+---------+---------+--------+------+---------------------+---------------------+--------+----------+--------+------+---------+---------+
| blog_id | site_id | domain | path | registered          | last_updated        | public | archived | mature | spam | deleted | lang_id |
+---------+---------+--------+------+---------------------+---------------------+--------+----------+--------+------+---------+---------+
|       1 |       1 |        | /    | 2021-01-20 21:02:13 | 0000-00-00 00:00:00 |      1 |        0 |      0 |    0 |       0 |       0 |
+---------+---------+--------+------+---------------------+---------------------+--------+----------+--------+------+---------+---------+
1 row in set (0.00 sec)
```

Исправляем:
```
UPDATE wp_blogs SET domain = 'my-site.ru' WHERE blog_id = 1;
```

#### Сайт начал работать

---

---

#### Задание 2. Обновить MySQL 5.7 на MariaDB 10.4 с сохранением всех баз данных.

При обновлении, нужно обратить внимание на документацию MariaDB: https://mariadb.com/kb/en/upgrading-from-mysql-to-mariadb/  
В частности, обновление нужно выполнять в несколько этапов: MySQL 5.7 -> MariaDB 10.2 -> MariaDB 10.3 -> MariaDB 10.4  
Дополнительно, документация ISPsystem: https://doc.ispsystem.ru/index.php/Смена_основной_версии_MySQL

##### Работающий вариант обновления

1\. Установить репозиторий MariaDB 10.2: https://downloads.mariadb.org/mariadb/repositories/

```
apt-get install software-properties-common
apt-key adv --fetch-keys 'https://mariadb.org/mariadb_release_signing_key.asc'
add-apt-repository 'deb [arch=amd64,arm64,ppc64el] https://mirror.docker.ru/mariadb/repo/10.2/ubuntu bionic main'
```

2\. Установить MariaDB. При этом, удалять MySQL не требуется – он будет удален автоматически:
```
apt update
apt install mariadb-server
mysql_upgrade
```

3\. Обновление MariaDB 10.2 -> 10.3:  
в `/etc/apt/sources.list` поправить путь к репозиторию, выполнить команды
```
apt update
apt install mariadb-server
mysql_upgrade
```
4\. Обновление MariaDB 10.3 -> 10.4 – аналогично пункту 2.1.

5. Убедиться, что панель корректно работает с базами, сайт так же работает. 

#### Обновление выполнено


---

---

#### Задание 3. Настроить шаблон конфига nginx, *генерируемого панелью*, так, чтобы файлы с расширениями `ttf|otf|woff|woff2|webp` отдавались nginx точно так же, как другая статика. 

Задача заключается в том, что при редактировании существующих или добавлении новых www-доменов, в конфиге nginx для этих сайтов должно указываться:
```
		location ~* ^.+\.(jpg|jpeg|gif|png|svg|js|css|mp3|ogg|mpe?g|avi|zip|gz|bz2?|rar|swf|ttf|otf|woff|woff2|webp)$ {
			try_files $uri $uri/ @fallback;
		}
```
а не:
```
		location ~* ^.+\.(jpg|jpeg|gif|png|svg|js|css|mp3|ogg|mpe?g|avi|zip|gz|bz2?|rar|swf)$ {
			try_files $uri $uri/ @fallback;
		}
```

##### Решение задачи

Запрос "ispmanager шаблон nginx" в Google приводит на статью [Шаблонизатор конфигурационных файлов](https://docs.ispsystem.ru/ispmanager-lite/nastrojki-veb-serverov/shablonizator-konfiguratsionnyh-fajlov)

Нужно скопировать файлы 
```
/usr/local/mgr5/etc/templates/default/nginx-vhosts.template 
/usr/local/mgr5/etc/templates/default/nginx-vhosts-ssl.template 
```
в
```
/usr/local/mgr5/etc/templates/nginx-vhosts.template 
/usr/local/mgr5/etc/templates/nginx-vhosts-ssl.template 
```

В скопированных файлах соответствующим образом поправить `location`. В файле `/usr/local/mgr5/etc/templates/nginx-vhosts.template` дополнительно заменить строку:
```
{% import etc/templates/default/nginx-vhosts-ssl.template %}
```
на:
```
{% import etc/templates/nginx-vhosts-ssl.template %}
```

Перезапустить панель:
```
killall -9 core
или
/usr/local/mgr5/sbin/mgrctl -m ispmgr exit
```

Далее необходимо убедиться, что:  
\* существующие www-домены редактируются, не возникает ошибок;  
\* создаются новые www-домены без ошибок;  

В конфиге для my-site.ru `location` задублируется (ошибок при этом не будет), будет плюсом его удаление. Альтернативно, можно воспользоваться командами для перегенерации всех конфигов: https://docs.ispsystem.ru/ispmanager-lite/nastrojka-ispmanager/konfiguratsiya-web-servera#id-Конфигурацияwebсервера-Переконфигурированиеweb-сервера

```
/usr/local/mgr5/sbin/mgrctl -m ispmgr webreconfigure.initialize shutdown=on
/usr/local/mgr5/sbin/mgrctl -m ispmgr webreconfigure.restore
```

#### Настройка шаблона выполнена

---

---

#### Задание 4. Пересобрать nginx с модулем pagespeed. Настроить nginx так, чтобы все сайты (в том числе новые) работали с ниже указанным конфигом. 

У клиента должна быть возможность легкого изменения конфига pagespeed сразу для всех сайтов. Конфиг:

```
pagespeed on;

# Needs to exist and be writable by nginx.  Use tmpfs for best performance.
pagespeed FileCachePath /var/ngx_pagespeed_cache;

# Ensure requests for pagespeed optimized resources go to the pagespeed handler
# and no extraneous headers get set.
location ~ "\.pagespeed\.([a-z]\.)?[a-z]{2}\.[^.]{10}\.[^.]+" {
  add_header "" "";
}
location ~ "^/pagespeed_static/" { }
location ~ "^/ngx_pagespeed_beacon$" { }

pagespeed LowercaseHtmlNames on;
pagespeed EnableFilters combine_css,combine_javascript;
pagespeed XHeaderValue "Powered By ngx_pagespeed";
```

---

##### Решение задачи

https://www.modpagespeed.com/doc/build_ngx_pagespeed_from_source  
Собирать можно в автоматическом режиме или, если хочется больше контроля, в ручном. 

---

##### Пересборка nginx с pagespeed

---

##### Автоматический режим

---

Официальная документация: https://www.modpagespeed.com/doc/build_ngx_pagespeed_from_source

```
bash <(curl -f -L -sS https://ngxpagespeed.com/install) --nginx-version latest
```

При установке просит указать параметры сборки nginx. Чтобы после пересборки nginx не возникло проблем (например, не отвалились SSL из-за отсутствия соответствующего модуля), нужно указать те же параметры, которые уже используются. Посмотреть можно командой `nginx -V`, раздел `configure arguments`:

```
root@80-78-255-49:~# nginx -V
...
configure arguments: --prefix=/etc/nginx --sbin-path=/usr/sbin/nginx --modules-path=/usr/lib/nginx/modules --conf-path=/etc/nginx/nginx.conf --error-log-path=/var/log/nginx/error.log --http-log-path=/var/log/nginx/access.log --pid-path=/var/run/nginx.pid --lock-path=/var/run/nginx.lock --http-client-body-temp-path=/var/cache/nginx/client_temp --http-proxy-temp-path=/var/cache/nginx/proxy_temp --http-fastcgi-temp-path=/var/cache/nginx/fastcgi_temp --http-uwsgi-temp-path=/var/cache/nginx/uwsgi_temp --http-scgi-temp-path=/var/cache/nginx/scgi_temp --user=nginx --group=nginx --with-compat --with-file-aio --with-threads --with-http_addition_module --with-http_auth_request_module --with-http_dav_module --with-http_flv_module --with-http_gunzip_module --with-http_gzip_static_module --with-http_mp4_module --with-http_random_index_module --with-http_realip_module --with-http_secure_link_module --with-http_slice_module --with-http_ssl_module --with-http_stub_status_module --with-http_sub_module --with-http_v2_module --with-mail --with-mail_ssl_module --with-stream --with-stream_realip_module --with-stream_ssl_module --with-stream_ssl_preread_module --with-cc-opt='-g -O2 -fdebug-prefix-map=/home/jenkins/workspace/Nginx_packages/label/ubuntu1804amd64/nginx-1.16.1=. -fstack-protector-strong -Wformat -Werror=format-security -Wp,-D_FORTIFY_SOURCE=2 -fPIC' --with-ld-opt='-Wl,-Bsymbolic-functions -Wl,-z,relro -Wl,-z,now -Wl,--as-needed -pie'
```

При сборке может возникнуть ошибка:
```
./configure: error: SSL modules require the OpenSSL library.
You can either do not enable the modules, or install the OpenSSL library
into the system, or build the OpenSSL library statically from the source
with nginx by using --with-openssl=<path> option.
```

Решение гуглится: https://github.com/apache/incubator-pagespeed-ngx/issues/1419
```
You need to install the libssl-dev package if you want to have ssl support in nginx.
```

После этого сборка происходит без ошибок, нужно утвердительно отвечать на все вопросы системы. На последний вопрос об установке Nginx так же нужно ответить утвердительно. В nginx -V нужно найти подключение модуля pagespeed:

```
configure arguments: --add-module=/root/incubator-pagespeed-ngx-latest-stable …
```

---

##### Ручной режим

---

В официальной документации информации мало, по этому в довесок: https://www.digitalocean.com/community/questions/how-to-add-pagespeed-on-existent-nginx-ubuntu-server

**Установить зависимости**
```
apt-get install build-essential zlib1g-dev libpcre3 libpcre3-dev unzip
```

**Скачать pagespeed**
```
NPS_VERSION=1.13.35.2-stable
cd
wget -O- https://github.com/apache/incubator-pagespeed-ngx/archive/v${NPS_VERSION}.tar.gz | tar -xz
nps_dir=$(find . -name "*pagespeed-ngx-${NPS_VERSION}" -type d)
cd "$nps_dir"
NPS_RELEASE_NUMBER=${NPS_VERSION/beta/}
NPS_RELEASE_NUMBER=${NPS_VERSION/stable/}
psol_url=https://dl.google.com/dl/page-speed/psol/${NPS_RELEASE_NUMBER}.tar.gz
[ -e scripts/format_binary_url.sh ] && psol_url=$(scripts/format_binary_url.sh PSOL_BINARY_URL)
wget -O- ${psol_url} | tar -xz  # extracts to psol/
```

**Скачать исходники nginx**
```
NGINX_VERSION=1.18.0
cd
wget -O- http://nginx.org/download/nginx-${NGINX_VERSION}.tar.gz | tar -xz
cd nginx-${NGINX_VERSION}/
```

**Сконфигурировать nginx (делаю по инструкции с сайта pagespeed)**
```
./configure --add-module=$HOME/$nps_dir ${PS_NGX_EXTRA_FLAGS}
```

---

**На этом этапе – несколько моментов:**

##### возникает ошибка
```
./configure: error: module ngx_pagespeed requires the pagespeed optimization library.
```
решение гуглится: https://github.com/apache/incubator-pagespeed-ngx/issues/1533  
нужно доустановить пакет `uuid-dev`

\* configure показывает, что nginx будет установлен в `/usr/local/nginx`. Нужно просто обратить на это внимание и засомневаться – nginx после сборки должен быть установлен туда, куда он уже установлен, по этому команду надо привести к виду с аргументами, подсмотренными командой `nginx -V`:
```
./configure --prefix=/etc/nginx --sbin-path=/usr/sbin/nginx --modules-path=/usr/lib/nginx/modules --conf-path=/etc/nginx/nginx.conf --error-log-path=/var/log/nginx/error.log --http-log-path=/var/log/nginx/access.log --pid-path=/var/run/nginx.pid --lock-path=/var/run/nginx.lock --http-client-body-temp-path=/var/cache/nginx/client_temp --http-proxy-temp-path=/var/cache/nginx/proxy_temp --http-fastcgi-temp-path=/var/cache/nginx/fastcgi_temp --http-uwsgi-temp-path=/var/cache/nginx/uwsgi_temp --http-scgi-temp-path=/var/cache/nginx/scgi_temp --user=nginx --group=nginx --with-compat --with-file-aio --with-threads --with-http_addition_module --with-http_auth_request_module --with-http_dav_module --with-http_flv_module --with-http_gunzip_module --with-http_gzip_static_module --with-http_mp4_module --with-http_random_index_module --with-http_realip_module --with-http_secure_link_module --with-http_slice_module --with-http_ssl_module --with-http_stub_status_module --with-http_sub_module --with-http_v2_module --with-mail --with-mail_ssl_module --with-stream --with-stream_realip_module --with-stream_ssl_module --with-stream_ssl_preread_module --with-cc-opt='-g -O2 -fdebug-prefix-map=/home/jenkins/workspace/Nginx_packages/label/ubuntu1804amd64/nginx-1.16.1=. -fstack-protector-strong -Wformat -Werror=format-security -Wp,-D_FORTIFY_SOURCE=2 -fPIC' --with-ld-opt='-Wl,-Bsymbolic-functions -Wl,-z,relro -Wl,-z,now -Wl,--as-needed -pie' --add-module=$HOME/$nps_dir ${PS_NGX_EXTRA_FLAGS} 
```

##### Возникает ошибка:
```
./configure: error: SSL modules require the OpenSSL library.
You can either do not enable the modules, or install the OpenSSL library
into the system, or build the OpenSSL library statically from the source
with nginx by using --with-openssl=<path> option.
```

Решение гуглится: https://github.com/apache/incubator-pagespeed-ngx/issues/1419
```
You need to install the libssl-dev package if you want to have ssl support in nginx.
```

---

Необходимый результат:
```

Configuration summary
  + using threads
  + using system PCRE library
  + using system OpenSSL library
  + using system zlib library

  nginx path prefix: "/etc/nginx"
  nginx binary file: "/usr/sbin/nginx"
  nginx modules path: "/usr/lib/nginx/modules"
  nginx configuration prefix: "/etc/nginx"
  nginx configuration file: "/etc/nginx/nginx.conf"
  nginx pid file: "/var/run/nginx.pid"
  nginx error log file: "/var/log/nginx/error.log"
  nginx http access log file: "/var/log/nginx/access.log"
  nginx http client request body temporary files: "/var/cache/nginx/client_temp"
  nginx http proxy temporary files: "/var/cache/nginx/proxy_temp"
  nginx http fastcgi temporary files: "/var/cache/nginx/fastcgi_temp"
  nginx http uwsgi temporary files: "/var/cache/nginx/uwsgi_temp"
  nginx http scgi temporary files: "/var/cache/nginx/scgi_temp"
```
---

**Завершить установку nginx с модулем**
```
make install
```

**В nginx -V нужно найти подключение модуля pagespeed:**
```
configure arguments:  …  --add-module=/root/./incubator-pagespeed-ngx-1.13.35.2-stable
```

---

##### Настройка конфигов nginx

Так как конфиг должен быть подключен ко всем сайтам, в том числе новым www-доменам, и не перезатираться при редактировании www-доменов, самым логичным является добавление конфига как файла в директории `/etc/nginx/vhosts-includes/`. После создания конфига и рестарта nginx, в заголовках сайтов должен появится заголовок "Powered By ngx_pagespeed"

```
% curl -I my-site.ru
HTTP/1.1 200 OK
...
X-Page-Speed: Powered By ngx_pagespeed
```

---

Нужно вспомнить о том, что обновление nginx повлечет установку версии без модуля, по этому пакет нужно заблокировать от обновлений. В Ubuntu делается командой:
```
apt-mark hold nginx
```

```
root@80-78-255-49:~/nginx-1.18.0# dpkg --get-selections | grep nginx
ispmanager-pkg-nginx				install
nginx						install
root@80-78-255-49:~/nginx-1.18.0# apt-mark hold nginx
nginx set on hold.
root@80-78-255-49:~/nginx-1.18.0# dpkg --get-selections | grep nginx
ispmanager-pkg-nginx				install
nginx						hold
```
