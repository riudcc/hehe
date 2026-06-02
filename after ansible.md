# Infrastructure Setup Guide — Part 2
> Docker, LAMP, Port Forwarding, Nginx Reverse Proxy

---

## Содержание

- [BS — Docker](#br-srv--docker)
- [HS — Веб-приложение (LAMP)](#hq-srv--веб-приложение-lamp)
- [Статическая трансляция портов (NAT)](#статическая-трансляция-портов-nat)
- [ISP — Nginx как обратный прокси](#isp--nginx-как-обратный-прокси)
- [HC — Финальная настройка](#hq-cli--финальная-настройка)

---

## BS — Docker

### 1. Установка

```bash
apt-get install -y docker-engine docker-compose
systemctl enable --now docker
```

### 2. Монтирование ISO с образами

```bash
blkid                        # найти additional.iso и определить устройство
mkdir /mnt/iso
mount -o loop /dev/sr0 /mnt/iso
ls /mnt/iso/docker/          # проверить наличие образов
```

### 3. Импорт Docker-образов

```bash
docker load < /mnt/iso/docker/site_latest.tar && \
docker load < /mnt/iso/docker/mariadb_latest.tar 
docker image ls              # убедиться, что образы загружены
```

### 4. Запуск контейнеров

Создайте `docker-compose.yaml`, заполнив данные согласно `readme.txt` из ISO:
```bash
services:
  database:
    container_name: db
    image: mariadb:latest
    restart: always
    ports:
      - "3306:3306"
    environment:
      DB_USER: maria
      DB_PASS: Passw0rd
      DB_NAME: mariadb
      MARIADB_ROOT_PASSWORD: P@ssw0rd

  app:
    container_name: testapp
    image: site:latest
    restart: always
    ports:
      - "8080:8000"
    environment:
      DB_HOST: "192.168.0.2"
      DB_PORT: "3306"
      DB_NAME: mariadb
      DB_USER: maria
      DB_PASS: Passw0rd
      DB_TYPE: maria
    depends_on:
      - database

docker compose up -d
```

> **Проверка:** с HQ-CLI открыть браузер и перейти по IP или доменному имени BR-SRV на нужный порт.

---

## HS — Веб-приложение (LAMP)

### 1. Установка стека

```bash
apt-get install lamp-server
systemctl enable --now mariadb
systemctl enable --now httpd2
```

> **Проверка:** с HQ-CLI убедиться, что веб-сервер отвечает по IP HS.

### 2. Монтирование ISO и копирование файлов сайта

```bash
mkdir /mnt/iso
mount -o loop /dev/sr0 /mnt/iso
ls /mnt/iso/web/

cp /mnt/iso/web/*.p* /var/www/html/
ls /var/www/html/            # проверка
```

### 3. Конфигурация `/var/www/html/index.php`

Отредактировать согласно заданию — указать параметры подключения к БД:

```php
<?php
$servername = "localhost";
$username   = "webc";
$password   = "P@ssw0rd";
$dbname     = "webdb";
```

### 4. Подготовка базы данных

```bash
mariadb -u root
```

```sql
CREATE DATABASE webdb;
CREATE USER 'webc'@'localhost' IDENTIFIED BY 'P@ssw0rd';
GRANT ALL PRIVILEGES ON webdb.* TO 'webc'@'localhost' WITH GRANT OPTION;
EXIT;
```

### 5. Импорт дампа БД

```bash
# Ввести пароль пользователя webc при запросе
mariadb -u webc -p webdb < /mnt/iso/web/dump.sql
```

> **Проверка:** с HC открыть браузер и перейти по IP HS.

---

## Статическая трансляция портов (NAT)

| Маршрутизатор | Внешний адрес:порт | Назначение | Описание |
|---|---|---|---|
| BR-RTR | `172.16.2.2:8080` | `192.168.3.2:8080` | Приложение testapp |
| BR-RTR | `172.16.2.2:2026` | `192.168.3.2:2026` | SSH → BR-SRV |
| HQ-RTR | `172.16.1.2:8080` | `192.168.100.2:80` | Веб-приложение HQ-SRV |
| HQ-RTR | `172.16.1.2:2026` | `192.168.100.2:2026` | SSH → HQ-SRV |

### BR

```bash
iptables -t nat -A PREROUTING -d 172.16.2.2 -p tcp --dport 8080 -j DNAT --to-destination 192.168.3.2:8080
iptables -t nat -A PREROUTING -d 172.16.2.2 -p tcp --dport 2026 -j DNAT --to-destination 192.168.3.2:2026
iptables-save > /etc/sysconfig/iptables
```

### HR

```bash
iptables -t nat -A PREROUTING -d 172.16.1.2 -p tcp --dport 8080 -j DNAT --to-destination 192.168.100.2:80
iptables -t nat -A PREROUTING -d 172.16.1.2 -p tcp --dport 2026 -j DNAT --to-destination 192.168.100.2:2026
iptables-save > /etc/sysconfig/iptables
```

### Проверка правил NAT

```bash
iptables -t nat -vnL
```

### Проверка с ISP

```bash
ssh sshuser@172.16.1.2 -p 2026
curl http://172.16.1.2:8080 | less
```

---

## ISP — Nginx как обратный прокси

### 1. Установка

```bash
apt-get install nginx apache2
```

### 2. Создание файла паролей (Basic Auth)

```bash
htpasswd -c /etc/nginx/.htpasswd WEB
# ввести пароль: P@ssw0rd
```

### 3. Резервная копия конфига

```bash
cp /etc/nginx/sites-available.d/default.conf{,.orig}
```

### 4. Конфигурация `/etc/nginx/sites-available.d/default.conf`

```nginx
server {
    listen 80;
    server_name web.au-team.irpo;

    location / {
        proxy_pass         http://172.16.1.10:8080;
        proxy_set_header   Host              $host;
        proxy_set_header   X-Real-IP         $remote_addr;
        proxy_set_header   X-Forwarded-For   $proxy_add_x_forwarded_for;
        proxy_set_header   X-Forwarded-Proto $scheme;

        auth_basic           "Restricted area";
        auth_basic_user_file /etc/nginx/.htpasswd;
    }
}

server {
    listen 80;
    server_name docker.au-team.irpo;

    location / {
        proxy_pass         http://172.16.2.10:8080;
        proxy_set_header   Host              $host;
        proxy_set_header   X-Real-IP         $remote_addr;
        proxy_set_header   X-Forwarded-For   $proxy_add_x_forwarded_for;
        proxy_set_header   X-Forwarded-Proto $scheme;
    }
}
```

### 5. Активация и запуск

```bash
ln -s /etc/nginx/sites-available.d/default.conf /etc/nginx/sites-enabled.d/
nginx -t                     # проверка конфига
systemctl enable --now nginx
```

---

## HC — Финальная настройка

### Добавить записи в `/etc/hosts`

```
172.16.1.1  web.au-team.irpo
172.16.2.1  docker.au-team.irpo
```

### Установить браузер

```bash
apt-get install -y yandex-browser-stable
```

> После этого оба доменных имени открываются в браузере корректно.

---
