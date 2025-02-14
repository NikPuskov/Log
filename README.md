# Log

# Настраиваем центральный сервер для сбора логов

Стенд поднят на Vagrant с boxами Ubuntu 24.04

# 1. Создаём виртуальные машины

Поднимаем 2 виртуальные машины: `vagrant up`

Заходим на web-сервер: `vagrant ssh web`

Переходим в root: `sudo -i`

Для правильной работы c логами, нужно, чтобы на всех хостах было настроено одинаковое время: `timedatectl`

Проверим, что время и дата указаны правильно: `date`

Настроить NTP нужно на обоих серверах

# 2. Установка nginx на виртуальной машине web

Установим nginx: `apt update && apt install -y nginx`

Проверим, что nginx работает корректно: `systemctl status nginx`

`ss -tln | grep 80`

![Image alt](https://github.com/NikPuskov/Log/blob/main/log1.jpg)

# 3. Настройка центрального сервера сбора логов

Откроем еще одно окно терминала и подключаемся по ssh к ВМ log: `vagrant ssh log`

Перейдем в пользователя root: `sudo -i`

rsyslog должен быть установлен по умолчанию в нашей ОС, проверим это: `apt list rsyslog`

Все настройки Rsyslog хранятся в файле /etc/rsyslog.conf `nano /etc/rsyslog.conf`

Для того, чтобы наш сервер мог принимать логи, нам необходимо внести следующие изменения в файл: 

Открываем порт 514 (TCP и UDP)

В конец файла /etc/rsyslog.conf добавляем правила приёма сообщений от хостов

![Image alt](https://github.com/NikPuskov/Log/blob/main/log2.jpg)

Данные параметры будут отправлять в папку /var/log/rsyslog логи, которые будут приходить от других серверов. 

Например, Access-логи nginx от сервера web, будут идти в файл /var/log/rsyslog/web/nginx_access.log

Сохраняем файл и перезапускаем службу rsyslog: `systemctl restart rsyslog`

Если ошибок не допущено, то у нас будут видны открытые порты TCP, UDP 514: `ss -tuln`

![Image alt](https://github.com/NikPuskov/Log/blob/main/log3.jpg)

Далее настроим отправку логов с web-сервера

Заходим на web сервер: `vagrant ssh web`

Переходим в root пользователя: `sudo -i` 

Проверим версию nginx: `nginx -v`

Версия nginx должна быть 1.7 или выше. В нашем примере используется версия nginx 1.24.

Находим в файле /etc/nginx/nginx.conf раздел с логами и приводим их к следующему виду:

![Image alt](https://github.com/NikPuskov/Log/blob/main/log6.jpg)

Для Access-логов указываем удаленный сервер и уровень логов, которые нужно отправлять. 

Для error_log добавляем удаленный сервер. Если требуется чтобы логи хранились локально и отправлялись на удаленный сервер, требуется указать 2 строки. 	

Tag нужен для того, чтобы логи записывались в разные файлы.

По умолчанию, error-логи отправляют логи, которые имеют severity: error, crit, alert и emerg. Если требуется хранить или пересылать логи с другим severity, то это также можно указать в настройках nginx. 

Проверяем, что конфигурация nginx указана правильно: `nginx -t`

Перезапускаем nginx: `systemctl restart nginx`

Попробуем несколько раз зайти по адресу (в моём случае) http://192.168.1.58

Далее заходим на log-сервер и смотрим информацию об nginx:

`cat /var/log/rsyslog/web/nginx_access.log`

`cat /var/log/rsyslog/web/nginx_error.log` 

Поскольку наше приложение работает без ошибок, файл nginx_error.log не будет создан. 

Чтобы сгенерировать ошибку, можно переместить файл веб-страницы, который открывает nginx - 

`mv /var/www/html/index.nginx-debian.html /var/www/` 

После этого мы получим 403 ошибку.

Видим, что логи отправляются корректно.

![Image alt](https://github.com/NikPuskov/Log/blob/main/log7.jpg)

# 4. Настройка аудита, контролирующего изменения конфигурации nginx

Установим утилиту auditd `apt install -y auditd` на `web`

Добавим правило, которое будет отслеживать изменения в конфигруации nginx. Для этого в конец файла `/etc/audit/rules.d/audit.rules` добавим следующие строки:

`-w /etc/nginx/nginx.conf -p wa -k nginx_conf`

`-w /etc/nginx/default.d/ -p wa -k nginx_conf`

Данные правила позволяют контролировать запись (w) и измения атрибутов (a) в:

`/etc/nginx/nginx.conf`

Всех файлов каталога `/etc/nginx/default.d/`

Для более удобного поиска к событиям добавляется метка `nginx_conf`

Перезапускаем службу auditd: `service auditd restart`

После данных изменений у нас начнут локально записываться логи аудита.

Чтобы проверить, что логи аудита начали записываться локально, нужно внести изменения в файл `/etc/nginx/nginx.conf` или поменять его атрибут, потом посмотреть информацию об изменениях

`ausearch -k nginx_conf -f /etc/nginx/nginx.conf`

![Image alt](https://github.com/NikPuskov/Log/blob/main/log8.jpg)

Настроим пересылку логов на удаленный сервер. 

Auditd по умолчанию не умеет пересылать логи, для пересылки на web-сервере потребуется установить пакет audispd-plugins

`apt -y install audispd-plugins`

Найдем и поменяем следующие строки в файле `/etc/audit/auditd.conf`:

`log_format = RAW`

`name_format = HOSTNAME`

В `name_format` указываем `HOSTNAME`, чтобы в логах на удаленном сервере отображалось имя хоста

В файле `/etc/audit/plugins.d/au-remote.conf` поменяем параметр `active` на `yes`

В файле /etc/audit/audisp-remote.conf требуется указать адрес сервера и порт, на который будут отправляться логи

`remote_server = 192.168.1.57` (ip address в моём случае)

Перезапускаем службу auditd: `service auditd restart`

На этом настройка web-сервера завершена. Далее настроим Log-сервер.

Установим утилиту auditd `apt install -y auditd` на `log`

Откроем порт TCP 60, для этого раскомментируем строку в файле /etc/audit/auditd.conf

`tcp_listen_port = 60`

Перезапускаем службу auditd: `service auditd restart`

На этом настройка пересылки логов аудита закончена. Можем попробовать поменять атрибут у файла `/etc/nginx/nginx.conf` и проверить на log-сервере, что пришла информация об изменении атрибута

![Image alt](https://github.com/NikPuskov/Log/blob/main/log9.jpg)

Лог об изменении атрибута файла на web:

На сервере `log` `grep web /var/log/audit/audit.log`
