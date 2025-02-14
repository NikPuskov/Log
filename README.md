# Log

# Настраиваем центральный сервер для сбора логов

Стенд поднят на Vagrant с boxами Ubuntu 24.04

1. Создаём виртуальные машины

Поднимаем 2 виртуальные машины: `vagrant up`

Заходим на web-сервер: `vagrant ssh web`

Переходим в root: `sudo -i`

Для правильной работы c логами, нужно, чтобы на всех хостах было настроено одинаковое время: `timedatectl`

Проверим, что время и дата указаны правильно: `date`

Настроить NTP нужно на обоих серверах

2. Установка nginx на виртуальной машине web

Установим nginx: `apt update && apt install -y nginx`

Проверим, что nginx работает корректно: `systemctl status nginx`

`ss -tln | grep 80`

![Image alt](https://github.com/NikPuskov/Log/blob/main/log1.jpg)

3. Настройка центрального сервера сбора логов

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

