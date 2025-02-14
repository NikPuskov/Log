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

