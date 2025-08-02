## ДЗ-09 Systemd — создание unit-файла

##### Задачи
1. Написать service, который будет раз в 30 секунд мониторить лог на предмет наличия ключевого слова (файл лога и ключевое слово должны задаваться в /etc/default).
2. Установить spawn-fcgi и создать unit-файл (spawn-fcgi.sevice) с помощью переделки init-скрипта (https://gist.github.com/cea2k/1318020).
3. Доработать unit-файл Nginx (nginx.service) для запуска нескольких инстансов сервера с разными конфигурационными файлами одновременно.  

##### Инструкция по выполнению домашнего задания
Написать service, который будет раз в 30 секунд мониторить лог на предмет наличия ключевого слова  
Для начала создаём файл с конфигурацией для сервиса в директории /etc/default - из неё сервис будет брать необходимые переменные.  

```
root@u24srv09:~# cat /etc/default/watchlog
cat: /etc/default/watchlog: No such file or directory
root@u24srv09:~# nano !$
nano /etc/default/watchlog
root@u24srv09:~# cat /etc/default/watchlog
# Configuration file for my watchlog service
# Place it to /etc/default

# File and word in that file that we will monitor
WORD="ALERT"
LOG=/var/log/watchlog.log
root@u24srv09:~#
```
Затем создаем /var/log/watchlog.log и пишем туда строки на своё усмотрение,
плюс ключевое слово ‘ALERT’
```
root@u24srv09:~# nano fill_watchlog.sh
root@u24srv09:~# cat !$
cat fill_watchlog.sh
#!/bin/bash
WORD="ALERT"
for ((i=1; i<=20; i++)); do
    random_string=$(head /dev/urandom | tr -dc A-Za-z0-9 | head -c 20)
  echo "$random_string $WORD" >> /var/log/watchlog.log
done
echo "Fin"
root@u24srv09:~#
root@u24srv09:~# chmod +x fill_watchlog.sh
root@u24srv09:~# ./fill_watchlog.sh
root@u24srv09:~# tail -n1 /var/log/watchlog.log 
COt9YaDwnTED2WYxtL1U ALERT
root@u24srv09:~#
```
##### Создадим скрипт:
```
root@u24srv09:~# nano /opt/watchlog.sh
root@u24srv09:~# cat !$
cat /opt/watchlog.sh
#!/bin/bash

WORD=$1
LOG=$2
DATE=`date`

if grep $WORD $LOG &> /dev/null
then
logger "$DATE: I found word $WORD, Master!"
else
exit 0
fi
root@u24srv09:~#
root@u24srv09:~# chmod +x /opt/watchlog.sh
root@u24srv09:~#
```
##### Создадим юнит для сервиса
```
root@u24srv09:~# nano /opt/watchlog.sh
root@u24srv09:~# cat !$
cat /opt/watchlog.sh
#!/bin/bash

WORD=$1
LOG=$2
DATE=`date`

if grep $WORD $LOG &> /dev/null
then
logger "$DATE: I found word, Master!"
else
exit 0
fi
root@u24srv09:~# chmod +x /opt/watchlog.sh
root@u24srv09:~# nano /etc/systemd/system/watchlog.service
root@u24srv09:~# cat !$
cat /etc/systemd/system/watchlog.service
[Unit]
Description=My watchlog service

[Service]
Type=oneshot
EnvironmentFile=/etc/default/watchlog
ExecStart=/opt/watchlog.sh $WORD $LOG
root@u24srv09:~#
```
##### Создадим юнит для таймера
```
root@u24srv09:~# cat !$
cat /etc/systemd/system/watchlog.timer
[Unit]
Description=Run watchlog script every 3 second

[Timer]
# Run every 3 second
OnUnitActiveSec=3
Unit=watchlog.service

[Install]
WantedBy=multi-user.target
root@u24srv09:~#
```
Затем достаточно только запустить timer
```
root@u24srv09:~# systemctl start watchlog.timer
root@u24srv09:~# systemctl status watchlog.timer
● watchlog.timer - Run watchlog script every 3 second
     Loaded: loaded (/etc/systemd/system/watchlog.timer; enabled; pre
set: enabled)
     Active: active (elapsed) since Wed 2025-07-30 19:56:45 UTC; 28min ago
    Trigger: n/a
   Triggers: ● watchlog.service

Jul 30 19:56:45 u24srv09 systemd[1]: Stopped watchlog.timer - Run watchlog script every 3 second.
Jul 30 19:56:45 u24srv09 systemd[1]: Stopping watchlog.timer - Run watchlog script every 3 second...
Jul 30 19:56:45 u24srv09 systemd[1]: Started watchlog.timer - Run watchlog script every 3 second.
root@u24srv09:~#
...
root@u24srv09:~# journalctl -u watchlog.service -n 4
Jul 30 20:41:37 u24srv09 systemd[1]: Starting watchlog.service - My watchlog service...
Jul 30 20:41:37 u24srv09 root[1903]: Wed Jul 30 20:41:37 UTC 2025: I found word ALERT, Master!
Jul 30 20:41:37 u24srv09 systemd[1]: watchlog.service: Deactivated successfully.
Jul 30 20:41:37 u24srv09 systemd[1]: Finished watchlog.service - My watchlog service.
root@u24srv09:~#

```
### time-out 
##### Установить spawn-fcgi и создать unit-файл (spawn-fcgi.sevice) с помощью переделки init-скрипта

Устанавливаем spawn-fcgi и необходимые для него пакеты
```
root@u24srv09:~# apt install spawn-fcgi php php-cgi php-cli  apache2 libapache2-mod-fcgid -y
...
Created symlink /etc/systemd/system/multi-user.target.wants/apache2.service → /usr/lib/systemd/system/apache2.service.
Created symlink /etc/systemd/system/multi-user.target.wants/apache-htcacheclean.service → /usr/lib/systemd/system/apache-htcacheclean.service.
...
```
 Создаем файл с настройками для будущего сервиса в файле /etc/spawn-fcgi/fcgi.conf
```
cat > /etc/spawn-fcgi/fcgi.conf
# You must set some working options before the "spawn-fcgi" service will work.
# If SOCKET points to a file, then this file is cleaned up by the init script.
#
# See spawn-fcgi(1) for all possible options.
#
# Example :
SOCKET=/var/run/php-fcgi.sock
OPTIONS="-u www-data -g www-data -s $SOCKET -S -M 0600 -C 32 -F 1 -- /usr/bin/php-cgi"
```
Создаем unit-file
```
cat /etc/systemd/system/spawn-fcgi.service
[Unit]
Description=Spawn-fcgi startup service by Otus
After=network.target

[Service]
Type=simple
PIDFile=/var/run/spawn-fcgi.pid
EnvironmentFile=/etc/spawn-fcgi/fcgi.conf
ExecStart=/usr/bin/spawn-fcgi -n $OPTIONS
KillMode=process

[Install]
WantedBy=multi-user.target
```
Убеждаемся, что все успешно работает
```
root@u24srv09:~# systemctl start spawn-fcgi
root@u24srv09:~# systemctl status spawn-fcgi
● spawn-fcgi.service - Spawn-fcgi startup service by Otus
     Loaded: loaded (/etc/systemd/system/spawn-fcgi.service; disabled; preset:
enabled)
     Active: active (running) since Sat 2025-08-02 10:22:42 UTC; 7s ago
   Main PID: 10045 (php-cgi)
      Tasks: 33 (limit: 2211)
     Memory: 14.5M (peak: 14.7M)
        CPU: 58ms
     CGroup: /system.slice/spawn-fcgi.service
             ├─10045 /usr/bin/php-cgi
             ├─10046 /usr/bin/php-cgi
             ├─10047 /usr/bin/php-cgi
```
##### Доработка unit-файла Nginx (nginx.service) для запуска нескольких инстансов сервера с разными конфигурационными файлами одновременно

Установим Nginx из стандартного репозитория
```
root@u24srv09:~# apt install nginx -y
Reading package lists... Done
Building dependency tree... Done
Reading state information... Done
...
Created symlink /etc/systemd/system/multi-user.target.wants/nginx.service → /usr/lib/systemd/system/nginx.service.
Could not execute systemctl:  at /usr/bin/deb-systemd-invoke line 148.
Setting up nginx (1.24.0-2ubuntu7.4) ...
Not attempting to start NGINX, port 80 is already in use.
```

Для запуска нескольких экземпляров сервиса модифицируем исходный service для использования различной конфигурации, а также PID-файлов. Для этого создадим новый Unit для работы с шаблонами (/etc/systemd/system/nginx@.service)
```
root@u24srv09:~# touch /etc/systemd/system/nginx@.service
root@u24srv09:~# nano !$
nano /etc/systemd/system/nginx@.service
root@u24srv09:~# cat !$
cat /etc/systemd/system/nginx@.service
# Stop dance for nginx
# =======================
#
# ExecStop sends SIGSTOP (graceful stop) to the nginx process.
# If, after 5s (--retry QUIT/5) nginx is still running, systemd takes control
# and sends SIGTERM (fast shutdown) to the main process.
# After another 5s (TimeoutStopSec=5), and if nginx is alive, systemd sends
# SIGKILL to all the remaining processes in the process group (KillMode=mixed).
#
# nginx signals reference doc:
# http://nginx.org/en/docs/control.html
#
[Unit]
Description=A high performance web server and a reverse proxy server
Documentation=man:nginx(8)
After=network.target nss-lookup.target

[Service]
Type=forking
PIDFile=/run/nginx-%I.pid
ExecStartPre=/usr/sbin/nginx -t -c /etc/nginx/nginx-%I.conf -q -g 'daemon on; master_process on;'
ExecStart=/usr/sbin/nginx -c /etc/nginx/nginx-%I.conf -g 'daemon on; master_process on;'
ExecReload=/usr/sbin/nginx -c /etc/nginx/nginx-%I.conf -g 'daemon on; master_process on;' -s reload
ExecStop=-/sbin/start-stop-daemon --quiet --stop --retry QUIT/5 --pidfile /run/nginx-%I.pid
TimeoutStopSec=5
KillMode=mixed

[Install]
WantedBy=multi-user.target
root@u24srv09:~#
```
Далее необходимо создать два файла конфигурации (/etc/nginx/nginx-first.conf, /etc/nginx/nginx-second.conf). Их можно сформировать из стандартного конфига /etc/nginx/nginx.conf, с модификацией путей до PID-файлов и разделением по портам
```
root@u24srv09:~# cat /etc/nginx/nginx-first.conf
user www-data;
worker_processes auto;
pid /run/nginx-first.pid;
error_log /var/log/nginx/error.log;
include /etc/nginx/modules-enabled/*.conf;

events {
        worker_connections 768;
        # multi_accept on;
}

http {
        include /etc/nginx/mime.types;
        default_type application/octet-stream;

        ssl_protocols TLSv1 TLSv1.1 TLSv1.2 TLSv1.3; # Dropping SSLv3, ref: POODLE
        ssl_prefer_server_ciphers on;
        access_log /var/log/nginx/access.log;
        gzip on;

        server {
                listen 9001;
        }

        include /etc/nginx/conf.d/*.conf;
        # include /etc/nginx/sites-enabled/*; # commented by NN
}
```
### Проверка работы сервисов
Если мы видим две группы процессов Nginx, то всё в порядке. Если сервисы не стартуют, смотрим их статус, ищем ошибки, проверяем ошибки в /var/log/nginx/error.log, а также в journalctl -u nginx@first.

+ systemctl status

```
root@u24srv09:~# systemctl status nginx@first
● nginx@first.service - A high performance web server and a reverse proxy server
     Loaded: loaded (/etc/systemd/system/nginx@.service; disabled; preset: enabled
)
     Active: active (running) since Sat 2025-08-02 10:59:55 UTC; 6s ago
       Docs: man:nginx(8)
    Process: 10486 ExecStartPre=/usr/sbin/nginx -t -c /etc/nginx/nginx-first.conf -q -g daemon on; master_process on; (code=exited, status=0/SUCCESS)
    Process: 10487 ExecStart=/usr/sbin/nginx -c /etc/nginx/nginx-first.conf -g daemon on; master_process on; (code=exited, status=0/SUCCESS)
   Main PID: 10490 (nginx)
      Tasks: 3 (limit: 2211)
     Memory: 2.3M (peak: 2.7M)
        CPU: 25ms
     CGroup: /system.slice/system-nginx.slice/nginx@first.service
             ├─10490 "nginx: master process /usr/sbin/nginx -c /etc/nginx/nginx-first.conf -g daemon on; master_process on;"
             ├─10491 "nginx: worker process"
             └─10492 "nginx: worker process"
root@u24srv09:~# systemctl status nginx@second
● nginx@second.service - A high performance web server and a reverse proxy server
     Loaded: loaded (/etc/systemd/system/nginx@.service; disabled; preset: enabled
)
     Active: active (running) since Sat 2025-08-02 11:00:24 UTC; 8s ago
       Docs: man:nginx(8)
    Process: 10501 ExecStartPre=/usr/sbin/nginx -t -c /etc/nginx/nginx-second.conf -q -g daemon on; master_process on; (code=exited, status=0/SUCCESS)
    Process: 10502 ExecStart=/usr/sbin/nginx -c /etc/nginx/nginx-second.conf -g daemon on; master_process on; (code=exited, status=0/SUCCESS)
   Main PID: 10504 (nginx)
      Tasks: 3 (limit: 2211)
     Memory: 2.4M (peak: 2.5M)
        CPU: 27ms
     CGroup: /system.slice/system-nginx.slice/nginx@second.service
             ├─10504 "nginx: master process /usr/sbin/nginx -c /etc/nginx/nginx-second.conf -g daemon on; master_process on;"
             ├─10505 "nginx: worker process"
             └─10506 "nginx: worker process"
```
+ Какие порты прослушиваются
```
root@u24srv09:~# ss -4lnpt | grep nginx
LISTEN 0      511          0.0.0.0:9001      0.0.0.0:*    users:(("nginx",pid=10492,fd=5),("nginx",pid=10491,fd=5),("nginx",pid=10490,fd=5))
LISTEN 0      511          0.0.0.0:9002      0.0.0.0:*    users:(("nginx",pid=10506,fd=5),("nginx",pid=10505,fd=5),("nginx",pid=10504,fd=5))
root@u24srv09:~#
```
+ Список процессов
```
root@u24srv09:~# ps afx | grep nginx-
  10520 pts/2    S+     0:00  |       \_ grep --color=auto nginx-
  10490 ?        Ss     0:00 nginx: master process /usr/sbin/nginx -c /etc/nginx/nginx-first.conf -g daemon on; master_process on;
  10504 ?        Ss     0:00 nginx: master process /usr/sbin/nginx -c /etc/nginx/nginx-second.conf -g daemon on; master_process on;
root@u24srv09:~#
```
+ Доступность сайта
```
$ curl -I http://otus-hw-09.ru:9002
HTTP/1.1 200 OK
Server: nginx/1.24.0 (Ubuntu)
Date: Sat, 02 Aug 2025 11:30:56 GMT
Content-Type: text/html
Content-Length: 615
Last-Modified: Tue, 11 Apr 2023 01:45:34 GMT
Connection: keep-alive
ETag: "6434bbbe-267"
Accept-Ranges: bytes
```
## ДЗ-09 выполнено

