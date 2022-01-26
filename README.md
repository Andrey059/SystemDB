SystemD
Задание.

Написать сервис, который будет раз в 30 секунд мониторить лог на предмет наличия ключевого слова. Файл и слово должны задаваться в /etc/sysconfig.
Из epel установить spawn-fcgi и переписать init-скрипт на unit-файл. Имя сервиса должно также называться.
Дополнить юнит-файл apache httpd возможностью запустить несколько инстансов сервера с разными конфигами
№1.  
Для начала создаём файл с конфигурацией для сервиса в директории /etc/sysconfig - из неё сервис будет брать необходимые переменные.  
cat /etc/sysconfig/watchlog
``` 
#Configuration file for my watchdog service
#Place it to /etc/sysconfig
#File and word in that file that we will be monit
WORD="ALERT"   
LOG=/var/log/watchlog.log
```
Затем создаем ```/var/log/watchlog.log``` и пишем туда строки на своё усмотрение, плюс ключевое слово ‘ALERT’

Создадим скрипт: cat /opt/watchlog.sh   
```
#!/bin/bash 
WORD=$1 
LOG=$2 
DATE=date 
if grep $WORD $LOG &> /dev/null 
then logger "$DATE: I found word, Master!" else exit 0 fi
```
Команда logger отправляет лог в системный журнал   
Дадим скрипту все разрешения : ```chmod 777 watchlog.sh```   
Создадим юнит для сервиса:   
```
[Unit]   
Description=My watchlog service   
[Service]   
Type=oneshot   
EnvironmentFile=/etc/sysconfig/watchlog   
ExecStart=/opt/watchlog.sh $WORD $LOG  
```
Создадим юнит для таймера:   
```
[Unit]   
Description=Run watchlog script every 30 second   
[Timer]   
#Run every 30 second   
OnUnitActiveSec=30   
Unit=watchlog.service 
[Install]   
WantedBy=multi-user.target   
```
Затем достаточно только стартануть timer:``` systemctl start watchlog.timer ``` И убедиться в результате: ``` tail -f /var/log/messages```

№2
Устанавливаем spawn-fcgi и необходимые для него пакеты:
```
yum install epel-release -y && yum install spawn-fcgi php php-cli mod_fcgid httpd -y
```
Раскомментировать строки с переменными в /etc/sysconfig/spawn-fcgi, должен принять следующий вид:
```
cat /etc/sysconfig/spawn-fcgi
# You must set some working options before the "spawn-fcgi" service will work.
# If SOCKET points to a file, then this file is cleaned up by the init script.
#
# See spawn-fcgi(1) for all possible options.
#
# Example :
SOCKET=/var/run/php-fcgi.sock
OPTIONS="-u apache -g apache -s $SOCKET -S -M 0600 -C 32 -F 1 -- /usr/bin/php-cgi"
```
Юнит файл имеет следующий вид ```vi /etc/systemd/system/spawn-fcgi.service```:
```
[Unit]
Description=Spawn-fcgi startup service by Otus
After=network.target

[Service]
Type=simple
PIDFile=/var/run/spawn-fcgi.pid
EnvironmentFile=/etc/sysconfig/spawn-fcgi
ExecStart=/usr/bin/spawn-fcgi -n $OPTIONS
KillMode=process

[Install]
WantedBy=multi-user.target

```
Убеждаемся, что все успешно работает:
```
systemctl start spawn-fcgi
systemctl status spawn-fcgi
```

№3

Для запуска нескольких экземпляров сервиса будем использовать шаблон в конфигурации файла окружения:
```
    cp /usr/lib/systemd/system/httpd.service /usr/lib/systemd/system/httpd@.service
    sed -i 's/sysconfig\/httpd/sysconfig\/httpd-%I/g' /usr/lib/systemd/system/httpd@.service
```
Cозданы и отредактированы два файла со средами
```
    cp /etc/sysconfig/httpd /etc/sysconfig/httpd-first && cp /etc/sysconfig/httpd /etc/sysconfig/httpd-second
    sed -i 's/#OPTIONS=/OPTIONS=-f conf\/httpd-first.conf/g' /etc/sysconfig/httpd-first && sed -i 's/#OPTIONS=/OPTIONS=-f conf\/httpd-second.conf/g' /etc/sysconfig/httpd-second
```
Cозданы и отредактированы два файла конфигурации httpd
 ```   
    cp /etc/httpd/conf/httpd.conf /etc/httpd/conf/httpd-first.conf && cp /etc/httpd/conf/httpd.conf /etc/httpd/conf/httpd-second.conf
    sed -i 's/Listen 80/Listen 8080/g' /etc/httpd/conf/httpd-second.conf && echo "PidFile /var/run/httpd-second.pid" >> /etc/httpd/conf/httpd-second.conf
```
Запускаем
```
    systemctl daemon-reload
    systemctl enable --now httpd@first
    systemctl enable --now httpd@second
```
  Проверяем
  ```
  ss -tnulp | grep httpd
  ```
