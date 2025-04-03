# Лабораторная работа №6: Взаимодействие контейнеров

## Цель работы

Научиться управлять взаимодействием нескольких контейнеров.

## Задание

- Создать PHP-сайт в папке `mounts/site`
- Создать контейнер `backend` на основе `php:7.4-fpm`
- Создать контейнер `frontend` на основе `nginx:1.23-alpine`
- Связать контейнеры сетью `internal`
- Сделать сервер nginx доступным на `localhost`

---

## Ход работы

### 1. Клонирован пустой Git-репозиторий `containers06`

```sh
git clone git@github.com:iurii1801/containers06.git
```

![image](https://i.imgur.com/Lrfrnl8.png)

### 2. Созданы папки `nginx/` и `mounts/site/`, а также файлы `.gitignore` и `README.md`

```sh
# Создание папки для конфигурации nginx
New-Item -ItemType Directory -Path "nginx"

# Создание вложенной папки для сайта
New-Item -ItemType Directory -Path "mounts/site" -Force
```

![image](https://i.imgur.com/Y0kbG0E.png)

### 3. Создание файлов `.gitignore` и `README.md`

```sh
# Создание файла .gitignore
New-Item -Name ".gitignore" -ItemType "File"

# Создание файла README.md
New-Item -Name "README.md" -ItemType "File"
```

![image](https://i.imgur.com/UkzckTa.png)

### 4. В `.gitignore` добавлено

```sh
# Ignore files and directories
mounts/site/*
```

![image](https://i.imgur.com/gv3tnBJ.png)

### 5. В директории `containers06` добавлен файл `nginx/default.conf` со следующим содержимым

```sh
server {
    listen 80;
    server_name _;
    root /var/www/html;
    index index.php;
    location / {
        try_files $uri $uri/ /index.php?$args;
    }
    location ~ \.php$ {
        fastcgi_pass backend:9000;
        fastcgi_index index.php;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
        include fastcgi_params;
    }
}
```

![image](https://i.imgur.com/kJciwak.png)

### 6. Создан конфиг nginx: `nginx/default.conf`

```sh
New-Item -Path "nginx" -Name "default.conf" -ItemType "File"
```

![image](https://i.imgur.com/rywCSNJ.png)

### 7. Создана сеть Docker

```sh
docker network create internal
```

![image](https://i.imgur.com/3CR0o1V.png)

### 8. Создан контейнер `backend`

```sh
docker run -d \
  --name backend \
  --network internal \
  -v ${PWD}\mounts\site:/var/www/html \
  php:7.4-fpm
```

Контейнер `backend` создан на основе образа `php:7.4-fpm`. Он выполняет роль сервера PHP, обрабатывающего `.php`-файлы по запросу от веб-сервера nginx.

Контейнер подключён к внутренней Docker-сети `internal`, что позволяет nginx-контейнеру (`frontend`) передавать ему запросы через `fastcgi_pass backend:9000`.

Также в контейнер примонтирована директория `mounts/site` в путь `/var/www/html`, чтобы сервер имел доступ к коду PHP-сайта.

Контейнер запущен в фоновом режиме и работает без ошибок.

![image](https://i.imgur.com/pYoS04O.png)

### 9. Создан контейнер `frontend`

```sh
docker run -d \
  --name frontend \
  --network internal \
  -v ${PWD}\mounts\site:/var/www/html \
  -v ${PWD}\nginx\default.conf:/etc/nginx/conf.d/default.conf \
  -p 80:80 \
  nginx:1.23-alpine
```

Контейнер `frontend` создан на основе образа `nginx:1.23-alpine`. Он выполняет роль веб-сервера, доступного по адресу `http://localhost`.
Внутрь контейнера были примонтированы:

- сайт из локальной директории `mounts/site`, чтобы nginx мог обслуживать HTML и PHP-файлы;

- собственная конфигурация nginx из файла `default.conf`, в которой настроено взаимодействие с PHP-FPM (`backend`) через `fastcgi_pass`.

Контейнер подключён к внутренней сети `internal`, чтобы иметь доступ к контейнеру `backend` по его имени. Порт 80 контейнера проброшен на порт 80 хоста, что позволяет открыть сайт в браузере.

![image](https://i.imgur.com/KJVVmSU.png)

### 10. Проверка

- Сайт открывается по адресу `http://localhost`
- Отображается сайт (в случае этой лабы это галерея авто)

![image](https://i.imgur.com/cIzGEZw.png)

---

## Ответы на вопросы

### Каким образом в данном примере контейнеры могут взаимодействовать друг с другом?

Контейнеры `frontend` (nginx) и `backend` (php-fpm) были подключены к одной и той же пользовательской сети Docker с именем `internal`. Docker автоматически настраивает внутри такой сети встроенный DNS, благодаря чему контейнеры могут «видеть» друг друга по именам. Это значит, что nginx может передавать PHP-запросы на `backend:9000`, где `backend` — это имя PHP-контейнера, а `9000` — порт, на котором работает `php-fpm`. Такой механизм взаимодействия удобен, надёжен и не требует указания IP-адресов.

### Как видят контейнеры друг друга в рамках сети internal?

Контейнеры, подключённые к одной и той же сети (`internal`), автоматически получают возможность обращаться друг к другу по имени контейнера, так как в Docker-сети реализован встроенный DNS-сервер. Внутри nginx-конфигурации используется строка `fastcgi_pass backend:9000`; — здесь nginx отправляет запросы на контейнер с именем `backend`, и DNS Docker разрешает это имя в нужный IP внутри сети. Таким образом, контейнеры взаимодействуют напрямую, как если бы они находились в одной локальной сети.

### Почему необходимо было переопределять конфигурацию nginx?

Стандартная конфигурация образа `nginx:alpine` не содержит настроек для обработки PHP-файлов. По умолчанию nginx может обрабатывать только статические файлы (HTML, изображения и т. п.). Чтобы сайт на PHP работал корректно, необходимо вручную указать правила обработки `.php`-файлов через FastCGI и перенаправлять такие запросы в контейнер `backend` (где запущен `php-fpm`). Поэтому был создан файл `default.conf`, в котором настроен корневой каталог, индексация файлов и проксирование PHP-запросов. Этот конфигурационный файл был примонтирован в контейнер `frontend` и заменил собой стандартный конфиг nginx.

---

## Вывод

В результате выполнения лабораторной работы был успешно создан проект, состоящий из двух взаимосвязанных контейнеров: один для веб-сервера nginx, второй — для PHP-FPM. Контейнеры были объединены в общую сеть, что позволило им взаимодействовать друг с другом на уровне внутренних DNS-имен. Сайт на PHP был корректно размещён и отображается в браузере при переходе на `http://localhost`. Были отработаны навыки ручного создания контейнеров без использования docker-compose, что углубляет понимание работы Docker и сетевых взаимодействий между сервисами. Работа продемонстрировала уверенное владение инструментами контейнеризации и достигла поставленной цели по организации взаимодействия контейнеров в рамках одного приложения.

---

## Библиография

1. [Официальная документация Docker](https://docs.docker.com/)
2. [Документация PHP-FPM](https://www.php.net/manual/ru/install.fpm.php)
3. [Nginx: FastCGI конфигурация](https://nginx.org/ru/docs/http/ngx_http_fastcgi_module.html)
4. [Docker. Работа с сетями контейнеров](https://docs.docker.com/network/)
5. [DockerHub: php:7.4-fpm](https://hub.docker.com/_/php)
6. [DockerHub: nginx:1.23-alpine](https://hub.docker.com/_/nginx)