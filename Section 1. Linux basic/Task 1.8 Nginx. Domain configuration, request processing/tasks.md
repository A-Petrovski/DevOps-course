# Практика: Настройка Nginx, HTTPS (SSL), редиректов и обработки контента в Ubuntu

## 1. Подготовка инфраструктуры
1. Создайте виртуальную машину (ВМ) с **Ubuntu Server** в VirtualBox.
2. В настройках сети ВМ выберите тип подключения **«Сетевой мост» (Bridged Adapter)**, чтобы ВМ получила IP-адрес из локальной сети.
3. Запустите ВМ и подключитесь к ней по SSH с основного компьютера:

```bash
ssh username@<IP_АДРЕС_ВМ>
```

* **`username`** — имя пользователя на сервере Ubuntu.
* **`<IP_АДРЕС_ВМ>`** — локальный IP-адрес виртуальной машины.

**Пример использования:**
```bash
ssh amp@192.168.1.55
```

---

## 2. Установка Nginx и создание главной страницы

### 2.1. Установка веб-сервера и определение IP-адреса
```bash
sudo apt update && sudo apt install nginx -y
hostname -I
```

* **`sudo`** — выполнение команды с правами суперпользователя.
* **`apt update`** — обновление локального кэша доступных пакетов.
* **`apt install nginx`** — установка пакета веб-сервера Nginx.
* **`-y`** — автоматическое подтверждение установки без интерактивного запроса.
* **`hostname -I`** — вывод всех назначенных сетевых IP-адресов хоста (через пробел).

**Пример использования:**
```bash
ip -br a
```

### 2.2. Создание главной HTML-страницы и перезагрузка Nginx
```bash
echo "My private IP is $(hostname -I)" | sudo tee /var/www/html/index.html
sudo systemctl reload nginx
```

* **`echo "..."`** — вывод текстовой строки в стандартный поток вывода (`stdout`).
* **`$(hostname -I)`** — подстановка вывода команды получения IP-адреса внутрь строки.
* **`|`** — перенаправление вывода предыдущей команды на вход следующей (конвейер/pipe).
* **`sudo tee /var/www/html/index.html`** — запись полученных данных в системный файл с правами `root` и одновременный вывод текста в терминал.
* **`systemctl reload nginx`** — плавная перезагрузка конфигурации Nginx без разрыва активных соединений.

**Пример использования:**
```bash
echo "<h1>Server $(hostname) is active</h1>" | sudo tee /var/www/html/index.html
```

---

## 3. Настройка локального DNS (файл `hosts`)
Выполняется на **основном компьютере** (хосте) для привязки локального домена `my-secure-server.local` к IP-адресу виртуальной машины.

* **Linux / macOS:**
```bash
sudo vim /etc/hosts
```
* **Windows (PowerShell от имени Администратора):**
```powershell
notepad C:\Windows\System32\drivers\etc\hosts
```

* **`sudo vim /etc/hosts`** — открытие системного файла сопоставления доменов и IP-адресов в редакторе Vim с правами администратора.
* **`notepad C:\Windows\System32\drivers\etc\hosts`** — запуск Блокнота с открытием файла `hosts` в ОС Windows.

**Содержимое добавляемой строки (замените IP на адрес вашей ВМ):**
```text
192.168.1.55 my-secure-server.local
```

**Пример использования:**
```bash
echo "192.168.1.55 my-secure-server.local" | sudo tee -a /etc/hosts
```

---

## 4. Создание самоподписанного SSL-сертификата

### 4.1. Подготовка директории для закрытого ключа
```bash
sudo mkdir -p /etc/ssl/private
sudo chmod 700 /etc/ssl/private
```

* **`mkdir -p`** — создание директории (включая родительские каталоги, если они отсутствуют) без ошибки, если папка уже существует.
* **`chmod 700`** — установка прав доступа `rwx------` (чтение, запись и вход в каталог разрешены только владельцу `root`).

**Пример использования:**
```bash
sudo install -d -m 700 /etc/ssl/private
```

### 4.2. Генерация пары ключ + сертификат через OpenSSL
```bash
sudo openssl req -x509 -nodes -days 365 -newkey rsa:2048   -keyout /etc/ssl/private/nginx-selfsigned.key   -out /etc/ssl/certs/nginx-selfsigned.crt
```

* **`req`** — подкоманда OpenSSL для создания и обработки запросов на сертификат (CSR) и самоподписанных сертификатов.
* **`-x509`** — выпуск самоподписанного сертификата стандарта X.509 вместо запроса на подпись.
* **`-nodes`** — (*no DES*) отказ от шифрования закрытого ключа парольной фразой, чтобы Nginx мог запускаться автоматически без ручного ввода пароля.
* **`-days 365`** — срок действия сертификата в днях (1 год).
* **`-newkey rsa:2048`** — одновременная генерация нового закрытого ключа по алгоритму RSA длиной 2048 бит.
* **`-keyout <путь>`** — путь для сохранения файла закрытого ключа.
* **`-out <путь>`** — путь для сохранения файла публичного сертификата.

> **Важно:** При интерактивном заполнении полей обязательно укажите доменное имя **`my-secure-server.local`** в поле **`Common Name (e.g. server FQDN or YOUR name)`**.

**Пример использования (неинтерактивная генерация одной строкой с параметром `-subj`):**
```bash
sudo openssl req -x509 -nodes -days 365 -newkey rsa:2048   -keyout /etc/ssl/private/nginx-selfsigned.key   -out /etc/ssl/certs/nginx-selfsigned.crt   -subj "/CN=my-secure-server.local"
```

---

## 5. Настройка Nginx для работы с HTTPS и редиректом с HTTP

### 5.1. Создание конфигурационного файла виртуального хоста
```bash
sudo tee /etc/nginx/sites-available/my-secure-server > /dev/null <<'EOF'
# Блок для HTTP (порт 80) — принудительный редирект на HTTPS
server {
    listen 80;
    server_name my-secure-server.local;

    return 301 https://$host$request_uri;
}

# Основной блок для HTTPS (порт 443)
server {
    listen 443 ssl;
    server_name my-secure-server.local;

    # Пути к самоподписанному сертификату и закрытому ключу
    ssl_certificate /etc/ssl/certs/nginx-selfsigned.crt;
    ssl_certificate_key /etc/ssl/private/nginx-selfsigned.key;

    root /var/www/html;
    index index.html;

    location / {
        try_files $uri $uri/ =404;
    }
}
EOF
```

* **`listen 80;`** — прослушивание входящего незашифрованного HTTP-трафика на 80 порту.
* **`listen 443 ssl;`** — прослушивание входящего HTTPS-трафика на 443 порту с включением TLS/SSL.
* **`server_name`** — доменное имя виртуального хоста, для которого применяется данный блок `server`.
* **`return 301 https://$host$request_uri;`** — постоянный HTTP-редирект (код `301 Moved Permanently`) с сохранением имени хоста (`$host`) и полного пути запроса (`$request_uri`).
* **`ssl_certificate`** — путь к файлу публичного SSL-сертификата.
* **`ssl_certificate_key`** — путь к файлу закрытого ключа.
* **`root /var/www/html;`** — корневая директория с файлами сайта.
* **`index index.html;`** — файл, отдаваемый по умолчанию при обращении к директории.
* **`try_files $uri $uri/ =404;`** — последовательная проверка существования запрошенного файла (`$uri`), директории (`$uri/`) или возврат ошибки `404 Not Found`.

**Пример использования:**
```bash
sudo nano /etc/nginx/sites-available/my-secure-server
```

### 5.2. Активация конфигурации, проверка синтаксиса и перезапуск
```bash
sudo ln -s /etc/nginx/sites-available/my-secure-server /etc/nginx/sites-enabled/
sudo nginx -t
sudo systemctl restart nginx
```

* **`ln -s <источник> <ссылка>`** — создание символической ссылки (*symlink*) в каталоге активных сайтов `sites-enabled`.
* **`nginx -t`** — тестирование синтаксиса всех конфигурационных файлов Nginx без перезапуска службы.
* **`systemctl restart nginx`** — полный перезапуск службы Nginx (необходим при добавлении новых портов или SSL-сертификатов).

**Пример использования:**
```bash
sudo nginx -t && sudo systemctl reload nginx
```

---

## 6. Проверка работы HTTPS в браузере
1. Откройте на основном компьютере адрес **`https://my-secure-server.local`** (или `http://my-secure-server.local` для проверки редиректа).
2. При появлении предупреждения безопасности о самоподписанном сертификате нажмите **«Дополнительно» -> «Перейти на сайт my-secure-server.local (небезопасно)»**.
3. Убедитесь, что отображается страница с IP-адресом сервера, а в адресной строке активен протокол `https://`.

**Пример проверки через терминал (с игнорированием проверки самоподписанного сертификата):**
```bash
curl -Ik https://my-secure-server.local
```

---

## 7. Настройка редиректов и обработка контента

### 7.1. Редирект на страницу с картинкой (`/picture`)
```bash
echo '<img src="/images/my_cat.jpg">' | sudo tee /var/www/html/picture.html
sudo mkdir -p /var/www/html/images
sudo apt install curl -y
sudo curl -L -o /var/www/html/images/my_cat.jpg https://http.cat/images/200.jpg
echo "My private IP is $(hostname -I) <br> <a href="/picture">Show Picture</a>" | sudo tee /var/www/html/index.html
```

* **`curl -L`** — автоматический переход по HTTP-редиректам (`301`/`302`), если сервер перенаправляет запрос.
* **`curl -o <файл>`** — сохранение скачанного содержимого в указанный локальный файл вместо вывода в консоль.
* **`<br>`** — HTML-тег переноса строки.
* **`<a href="/picture">...</a>`** — HTML-ссылка на относительный путь `/picture`.

**Блок `location` для добавления внутрь `server { listen 443 ssl; ... }`:**
```nginx
location = /picture {
    return 302 /picture.html;
}
```

* **`location = /picture`** — модификатор `=` задает строгое точное совпадение URI (сработает только для `/picture`, но не для `/picture/1`).
* **`return 302 /picture.html;`** — временный редирект (код `302 Found`) на страницу `/picture.html`.

**Пример использования:**
```bash
file /var/www/html/images/my_cat.jpg
```

---

### 7.2. Настройка скачивания MP3-файла через директиву `alias`
```bash
sudo curl -L -o /var/www/html/music.mp3 https://www.soundhelix.com/examples/mp3/SoundHelix-Song-1.mp3
echo '<br> <a href="/download/music.mp3">Download Music</a>' | sudo tee -a /var/www/html/index.html
```

* **`tee -a`** — (*append*) дозапись строки в конец существующего файла без перезаписи его содержимого.

**Блок `location` для добавления внутрь `server { listen 443 ssl; ... }`:**
```nginx
location /download/ {
    alias /var/www/html/;
}
```

* **`alias /var/www/html/;`** — подмена префикса пути запроса. При запросе `/download/music.mp3` Nginx отбросит `/download/` и будет искать файл напрямую по пути `/var/www/html/music.mp3` (в отличие от `root`, который искал бы `/var/www/html/download/music.mp3`).

> **Важно:** Имя файла в ссылке (`music.mp3` или `song.mp3`) должно точно совпадать с именем файла, сохраненного в `/var/www/html/`.

**Пример использования:**
```bash
ls -lh /var/www/html/music.mp3
```

---

### 7.3. Регулярные выражения для статики и поворот JPG на 180°

**Установка дополнительного модуля обработки изображений:**
```bash
sudo apt install libnginx-mod-http-image-filter -y
```

* **`libnginx-mod-http-image-filter`** — динамический модуль Nginx на базе библиотеки `libgd` для обрезки, изменения размера и поворота изображений на лету.

**Блоки `location` для добавления внутрь `server { listen 443 ssl; ... }`:**
```nginx
# Кеширование статических файлов (без учета регистра)
location ~* \.(png|gif|ico|css|js)$ {
    root /var/www/html;
    expires 1d;
}

# Обработка JPG с поворотом на 180 градусов
location ~* \.jpg$ {
    root /var/www/html;
    image_filter rotate 180;
    image_filter_buffer 10M;
}
```

* **`location ~*`** — совпадение по регулярному выражению без учета регистра символов (*case-insensitive*, сработает и для `.jpg`, и для `.JPG`).
* **`\.(png|gif|ico|css|js)$`** — регулярное выражение, проверяющее окончание (`$`) URL на одно из перечисленных расширений.
* **`expires 1d;`** — добавление HTTP-заголовков `Expires` и `Cache-Control` для кеширования файлов браузером на 1 день (`1d`).
* **`image_filter rotate 180;`** — поворот изображения на 180 градусов перед отправкой клиенту (допустимые углы: `90`, `180`, `270`).
* **`image_filter_buffer 10M;`** — увеличение максимального размера буфера для обрабатываемого изображения с `1M` (по умолчанию) до `10M`, чтобы избежать ошибки `415 Unsupported Media Type` на крупных фото.

**Применение всех изменений:**
```bash
sudo nginx -t && sudo systemctl restart nginx
```

**Пример использования:**
```bash
curl -Ik https://my-secure-server.local/images/my_cat.jpg
```
