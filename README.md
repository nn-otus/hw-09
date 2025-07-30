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

for ((i=1; i<=20; i++)); do
    random_string=$(head /dev/urandom | tr -dc A-Za-z0-9 | head -c 20)
  echo "$random_string ALERT" >> /var/log/watchlog.log
done
echo "Fin"
root@u24srv09:~#
root@u24srv09:~# chmod +x fill_watchlog.sh
root@u24srv09:~# ./fill_watchlog.sh
root@u24srv09:~# tail -n1 /var/log/watchlog.log || wc
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
logger "$DATE: I found word ALERT, Master!"
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
