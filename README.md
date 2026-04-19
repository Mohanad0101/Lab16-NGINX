# **Лабораторная работа: Развёртывание и защита веб\-сервера Nginx на Ubuntu 24.04**

---

## **Цель работы**

Освоить процесс развёртывания веб\-сервера Nginx, настройки межсетевого экрана и обеспечения безопасного HTTPS-соединения.

## **Необходимые компоненты**

| Компонент | Требование |
| :---- | :---- |
| ОС | Ubuntu 24.04 LTS |
| Сеть | VirtualBox Host-Only адаптер |
| Права | sudo |
| Хост | Windows с PowerShell |

---

## **Теоретическое введение**

### **Что такое Nginx?**

Nginx :это высокопроизводительный веб\-сервер, использующий асинхронную событийно-ориентированную архитектуру. В отличие от Apache, который создаёт отдельный поток на каждое соединение, Nginx использует один рабочий процесс для обслуживания тысяч одновременных соединений. Это делает его более эффективным по использованию памяти и производительности.

Где применяется: Netflix, Airbnb, Яндекс, ВКонтакте, большинство современных VPS/VDS серверов.

### **Основные концепции лабораторной работы**

| Концепция | Описание |
| :---- | :---- |
| systemd | Менеджер служб Linux. Управляет запуском, остановкой и автозапуском сервисов |
| UFW | Межсетевой экран. Реализует принцип минимальных привилегий   открывает только необходимые порты |
| Виртуальный хост | Блок `server` в конфигурации Nginx. Позволяет обслуживать несколько сайтов на одном сервере |
| TLS/HTTPS | Шифрование трафика. Самоподписанный сертификат используется в лабораторной среде |
| Graceful reload | Применение новой конфигурации без остановки обслуживания пользователей |

---

## **  1: Удаление Apache2**

Цель: Освободить порты 80 и 443, удалить конфликтующее ПО.

### **Шаг 1.1: Остановка службы**

`bash`

`sudo systemctl stop apache2`

`sudo systemctl disable apache2`

### **Шаг 1.2: Полное удаление**

`bash`

`sudo apt purge --auto-remove apache2* -y`

### **Шаг 1.3: Проверка**

`bash`

`sudo ss -tulpn | grep :80`

Ожидаемый результат: Пустой вывод. Порт 80 свободен.

---

## **Настройка сети виртуальной машины**

В данной лабораторной работе используются два сетевых адаптера:

    enp0s3 : NAT, используется для доступа в Интернет.

    enp0s8 : Host-Only, используется для связи между хост-машиной и виртуальной машиной.

Для enp0s3 следует оставить DHCP, чтобы виртуальная машина автоматически получала доступ в Интернет. Для enp0s8 можно использовать DHCP, но  рекомендуется статический IP, чтобы адрес не изменялся после перезагрузки.


Чтобы задать фиксированный адрес, нужно изменить конфигурацию Netplan и вручную указать IP для enp0s8. Это обеспечивает стабильный адрес Host-Only интерфейса и упрощает доступ к Nginx с хост-машины.
Команды для настройки

Проверить имена интерфейсов:

`bash`

`ip link`

Открыть конфигурацию Netplan:

`bash`

`ls /etc/netplan/
sudo nano /etc/netplan/00-installer-config.yaml`


Пример конфигурации:

text
network:
  version: 2
  ethernets:
    enp0s3:
      dhcp4: true
    enp0s8:
      dhcp4: no
      addresses:
        - 192.168.56.101/24

Применить изменения:

`bash`

`sudo netplan apply`

Проверить результат:

`bash`

`ip addr show enp0s8`

`ip route`

---

## **  2: Установка Nginx**

Цель: Установить и запустить веб\-сервер.

### **Шаг 2.1: Установка**

`bash`

`sudo apt update`

`sudo apt install nginx -y`

### **Шаг 2.2: Запуск и автозапуск**

`bash`

`sudo systemctl enable nginx`

`sudo systemctl start nginx`

### **Шаг 2.3: Проверка состояния**

`bash`

`sudo systemctl status nginx`

Ожидаемый результат: `Active: active (running)`, `enabled`

### **Шаг 2.4: Проверка HTTP-доступа**

`bash`

`curl  http://localhost`

Ожидаемый результат: `HTTP/1.1 200 OK`, `Server: nginx`
---
Windows Browser 
http://192.168.56.10?
---

## **  3: Настройка межсетевого экрана UFW**

Цель: Разрешить только необходимые порты (22, 80, 443), запретить всё остальное.

Важно: SSH должен быть разрешён ДО включения UFW, иначе вы потеряете доступ к серверу.

### **Шаг 3.1: Разрешение портов**

---`bash`

`sudo ufw allow ssh`  
`sudo ufw allow http`

`sudo ufw allow https`
---
### **Шаг 3.2: Включение UFW**

---`bash`

`sudo ufw enable`
---
При запросе введите `y`.

### **Шаг 3.3: Проверка правил**

---`bash`

`sudo ufw status verbose`

---
### **Шаг 3.4: Тестирование с хост-машины**

В PowerShell на Windows:

`in new powershell terminal`

`curl  http://<IP-адрес-вашей-ВМ>`

Ожидаемый результат: 
StatusCode   : 200   
StatusDescription : OK 
Content  : <!DOCTYPE html>  
---

## **  4: Изучение конфигурации Nginx**

Цель: Понять структуру конфигурационных файлов.

### **Шаг 4.1: Просмотр главного файла**

---`bash`

`sudo cat /etc/nginx/nginx.conf`
---
Ключевые директивы:

| Директива | Значение |
| :---- | :---- |
| `user www-data;` | Рабочие процессы работают от непривилегированного пользователя |
| `worker_processes auto;` | Один процесс на ядро CPU |
| `include /etc/nginx/sites-enabled/*;` | Подключение активных виртуальных хостов |

### **Шаг 4.2: Модульная структура**

---`bash`

`ls -la /etc/nginx/sites-available/`

`ls -la /etc/nginx/sites-enabled/`
---
Пояснение:

* `sites-available`   хранит все конфигурации (активные и неактивные)  
* `sites-enabled`   содержит символические ссылки на активные конфигурации

### **Шаг 4.3: Проверка синтаксиса**

`bash`

`sudo nginx -t`

Ожидаемый результат: `syntax is ok`, `test is successful`

Важно: Всегда выполняйте эту команду перед перезагрузкой Nginx.

---

## **  5: Генерация TLS-сертификата**

Цель: Создать самоподписанный сертификат для HTTPS.

### **Шаг 5.1: Создание каталога для сертификатов**

`bash`

`sudo mkdir -p /etc/nginx/ssl`

### **Шаг 5.2: Генерация ключа и сертификата**

`bash`

`sudo openssl req -x509 -nodes -days 365 -newkey rsa:2048 \`  
  `-keyout /etc/nginx/ssl/selfsigned.key \`  
  `-out /etc/nginx/ssl/selfsigned.crt \`

  `-subj "/C=RU/ST=Moscow/L=Moscow/O=Lab/CN=localhost"`

Параметры:

| Параметр | Значение |
| :---- | :---- |
| `-x509` | Создаёт самоподписанный сертификат |
| `-nodes` | Ключ без парольной защиты |
| `-days 365` | Срок действия 1 год |
| `-newkey rsa:2048` | 2048-битный ключ RSA |

### **Шаг 5.3: Проверка**

`bash`

`ls -la /etc/nginx/ssl/`

Ожидаемый результат: Файлы `selfsigned.crt` и `selfsigned.key` присутствуют.

---

## **  6: Настройка виртуального хоста с HTTPS**

Цель: Настроить Nginx для приёма HTTPS-соединений и перенаправления HTTP на HTTPS.

### **Шаг 6.1: Редактирование конфигурации**

`bash`

`sudo nano /etc/nginx/sites-available/default`

Замените содержимое на следующее:

`nginx`
---
server {
    listen 443 ssl;
    server_name _;

    ssl_certificate /etc/nginx/ssl/selfsigned.crt;
    ssl_certificate_key /etc/nginx/ssl/selfsigned.key;

    root /var/www/html;
    index index.html index.htm;

    location / {
        try_files $uri $uri/ =404;
    }
}

server {
    listen 80;
    server_name _;

    return 301 https://$host$request_uri;
}
---
Что делает конфигурация:

| Блок | Функция |
| :---- | :---- |
| Первый `server` | Принимает HTTPS на порту 443, использует TLS-сертификат |
| Второй `server` | Принимает HTTP на порту 80, перенаправляет на HTTPS (код 301\) |

### **Шаг 6.2: Сохранение**

В nano: `Ctrl+O`, `Enter`, `Ctrl+X`

### **Шаг 6.3: Проверка и перезагрузка**

`bash`

`sudo nginx -t`

`sudo systemctl reload nginx`
---
`bash`

`echo '<h1>I have successfully installed and configured nginx webserver</h1> </br> <h2> Your Name here</h2>' | sudo tee /var/www/html/index.html`
---
Пояснение: `reload` применяет конфигурацию без остановки сервера (graceful reload).


### **Шаг 6.4: Проверка HTTPS на локальной машине**

`from windows browser `

`https://192.168.56.10?`

Ожидаемый результат: HTML-содержимое страницы приветствия Nginx.

---

## **  7: Итоговая проверка**

### **Проверка 7.1: Состояние службы**

`bash`

`sudo systemctl status nginx --no-pager`

Ожидаемый результат: `active (running)`

### **Проверка 7.2: Прослушиваемые порты**

`bash`

`sudo ss -tulpn | grep nginx`

Ожидаемый результат: Строки с `:80` и `:443`

### **Проверка 7.3: Тестирование с Windows (HTTP → HTTPS)**

В PowerShell:

`powershell`

`http://<IP-адрес-вашей-ВМ>`

Ожидаемый результат: `HTTP/1.1 301 Moved Permanently` и заголовок `Location: https://...`

### **Проверка 7.4: Тестирование с Windows (HTTPS)**

`powershell`

`curl -k https://<IP-адрес-вашей-ВМ>`

Ожидаемый результат: HTML-содержимое

### **Проверка 7.5: Просмотр журнала доступа**

`bash`

`sudo tail -5 /var/log/nginx/access.log`

Ожидаемый результат: Записи с IP-адресом Windows и кодами 301 или 200

---

## **Полный список команд **

`bash`

*`#   1: Удаление Apache2`*  
`sudo systemctl stop apache2`  
`sudo systemctl disable apache2`  
`sudo apt purge --auto-remove apache2* -y`  
`sudo ss -tulpn | grep :80`

*`#   2: Установка Nginx`*  
`sudo apt update`  
`sudo apt install nginx -y`  
`sudo systemctl enable nginx`  
`sudo systemctl start nginx`  
`sudo systemctl status nginx`

*`#   3: Настройка UFW`*  
`sudo ufw allow ssh`  
`sudo ufw allow http`  
`sudo ufw allow https`  
`sudo ufw enable`  
`sudo ufw status verbose`

*`#   4: Проверка конфигурации`*  
`sudo nginx -t`

*`#   5: Генерация TLS-сертификата`*  
`sudo mkdir -p /etc/nginx/ssl`  
`sudo openssl req -x509 -nodes -days 365 -newkey rsa:2048 \`  
  `-keyout /etc/nginx/ssl/selfsigned.key \`  
  `-out /etc/nginx/ssl/selfsigned.crt \`  
  `-subj "/C=RU/ST=Moscow/L=Moscow/O=Lab/CN=localhost"`

*`#   6: Настройка виртуального хоста`*  
`sudo nano /etc/nginx/sites-available/default`  
*`# (Вставьте конфигурацию из шага 6.1)`*

`sudo nginx -t`  
`sudo systemctl reload nginx`  
` https://192.168.56.10?`

*`#   7: Итоговая проверка`*  
`sudo ss -tulpn | grep nginx`

`sudo tail -5 /var/log/nginx/access.log`

---

## **Контрольный список**

| Пункт | Команда проверки | Ожидаемый результат |
| :---- | :---- | :---- |
| Apache2 удалён | `sudo ss -tulpn | grep :80` | Пустой вывод |
| Nginx работает | `sudo systemctl status nginx` | `active (running)` |
| UFW настроен | `sudo ufw status verbose` | Порты 22,80,443 ALLOW IN |
| Синтаксис верен | `sudo nginx -t` | `syntax is ok` |
| Сертификат создан | `ls /etc/nginx/ssl/` | Файлы .crt и .key |
| HTTPS доступен | `curl -k https://localhost` | HTML-содержимое |
| HTTP → HTTPS | Из PowerShell `curl -I http://<IP>` | Код 301 |

---

## **Устранение неполадок**

| Проблема | Диагностика | Решение |
| :---- | :---- | :---- |
| Port 80 busy | `sudo ss -tulpn | grep :80` | Завершить процесс или завершить удаление Apache2 |
| UFW блокирует | `sudo ufw status verbose` | Выполнить `sudo ufw allow http/https` |
| Nginx не стартует | `sudo nginx -t` | Исправить синтаксические ошибки |
| Сертификат не найден | `ls /etc/nginx/ssl/` | Повторить   5 |

---


## **Заключение**

В ходе лабораторной работы вы выполнили:

|   | Выполненное действие |
| :---- | :---- |
| 1 | Удалили Apache2 и освободили порты |
| 2 | Установили и запустили Nginx |
| 3 | Настроили межсетевой экран по принципу минимальных привилегий |
| 4 | Изучили иерархическую структуру конфигурации Nginx |
| 5 | Сгенерировали самоподписанный TLS-сертификат |
| 6 | Настроили виртуальный хост с HTTPS и перенаправлением |
| 7 | Выполнили полную проверку работоспособности |

Лабораторная работа завершена. Вы успешно развернули защищённый веб\-сервер Nginx.  
