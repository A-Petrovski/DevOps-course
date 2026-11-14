# Практическое задание: Настройка балансировки нагрузки (Reverse Proxy) в Nginx

## Подготовительная часть: Инфраструктура (VirtualBox)

1. **Создание ВМ:** В VirtualBox создаются три виртуальные машины на базе Ubuntu Server:
   * `Blue-server` (бэкенд №1)
   * `Yellow-server` (бэкенд №2)
   * `balancer-server` (балансировщик нагрузки)
2. **Настройка сети:** В свойствах каждой ВМ (`Настроить` -> `Сеть` -> `Адаптер 1`) устанавливается тип подключения **Сетевой мост (Bridged Adapter)**. Это позволяет всем машинам получить индивидуальные IP-адреса в локальной подсети от роутера.
3. **Запуск и определение адресов:** После запуска всех ВМ на каждой машине определяется её локальный IP-адрес.

```bash
ip -br a
```

* **Разбор флагов и параметров:**
  * `-br` (`--brief`) — вывод сетевых интерфейсов и их IP-адресов в кратком, компактном табличном виде без лишней технической информации.
  * `a` (`address`) — объект управления сетевыми адресами утилиты `ip`.

**Пример использования:** `ip -br -4 a` (выведет только IPv4-адреса в одну строку на интерфейс, отсекая длинные IPv6-адреса).

---

## 1. Настройка веб-серверов (Backend)

### 1.1. Настройка ВМ `Blue-server`

```bash
# 1. Установка Nginx
sudo apt update && sudo apt install nginx -y

# 2. Создание статической HTML-страницы
echo '<body style="background-color: blue;"><h1>Blue Server</h1></body>' | sudo tee /var/www/html/blue.html

# 3. Запись конфигурации виртуального хоста
sudo tee /etc/nginx/sites-available/default << 'EOF'
server {
    listen 80 default_server;
    listen [::]:80 default_server;

    root /var/www/html;
    index blue.html;

    server_name _;

    location / {
        try_files /blue.html =404;
    }
}
EOF

# 4. Проверка синтаксиса и перезапуск службы
sudo nginx -t && sudo systemctl restart nginx
```

* **Разбор флагов, команд и директив:**
  * `apt update` — обновление локального индекса доступных пакетов из репозиториев.
  * `apt install -y` — автоматическое подтверждение установки пакета без интерактивного запроса (`yes`).
  * `sudo tee <путь>` — чтение данных из стандартного потока ввода (`stdin`) и одновременная запись в системный файл с правами суперпользователя `root` (в отличие от оператора `>`, который выполняется от имени текущего пользователя).
  * `<< 'EOF'` — конструкция *Here Document* для передачи многострочного текста; одинарные кавычки вокруг `'EOF'` запрещают оболочке Bash интерпретировать переменные внутри текста (например, `$host` или `$remote_addr`).
  * `listen 80 default_server;` — указывает Nginx принимать HTTP-соединения на 80-м порту и использовать этот блок `server` по умолчанию, если заголовок `Host` не совпал ни с одним другим правилом.
  * `root /var/www/html;` — задает корневой каталог на диске, откуда веб-сервер берет файлы.
  * `try_files /blue.html =404;` — при любом запросе к `/` проверяет наличие файла `blue.html` в корневой папке и отдает его, иначе возвращает HTTP-код ошибки `404`.
  * `nginx -t` — тестирование конфигурационных файлов на синтаксические ошибки без перезапуска рабочего процесса.
  * `systemctl restart nginx` — полная перезагрузка службы Nginx для применения новых настроек.

**Пример использования:** `curl -i http://localhost` (проверка локальной отдачи страницы вместе с HTTP-заголовками ответа `200 OK`).

### 1.2. Настройка ВМ `Yellow-server`

```bash
# 1. Установка Nginx
sudo apt update && sudo apt install nginx -y

# 2. Создание статической HTML-страницы
echo '<body style="background-color: yellow;"><h1>Yellow Server</h1></body>' | sudo tee /var/www/html/yellow.html

# 3. Запись конфигурации виртуального хоста
sudo tee /etc/nginx/sites-available/default << 'EOF'
server {
    listen 80 default_server;
    listen [::]:80 default_server;

    root /var/www/html;
    index yellow.html;

    server_name _;

    location / {
        try_files /yellow.html =404;
    }
}
EOF

# 4. Проверка синтаксиса и перезапуск службы
sudo nginx -t && sudo systemctl restart nginx
```

* **Разбор флагов и директив:**
  * `index yellow.html;` — определяет имя индексного файла по умолчанию, который Nginx отдает при обращении к директории.
  * `try_files /yellow.html =404;` — перенаправляет все входящие запросы в корневой `location /` непосредственно на файл `yellow.html`.

**Пример использования:** `curl -s http://<IP_YELLOW_SERVER> | grep -o "Yellow Server"` (быстрая автоматизированная проверка наличия нужного заголовка в ответе удаленного сервера).

---

## 2. Настройка Балансировщика нагрузки (`balancer-server`)

```bash
# 1. Установка Nginx и создание резервной копии стандартного конфига
sudo apt update && sudo apt install nginx -y
sudo cp /etc/nginx/sites-available/default /etc/nginx/sites-available/default.bak

# 2. Запись конфигурации балансировщика (upstream + log_format + proxy)
sudo tee /etc/nginx/sites-available/default << 'EOF'
upstream backend_servers {
    server 192.168.1.21:80;
    server 192.168.1.22:80;
}

log_format upstream_log '$remote_addr - [$time_local] "$request" '
                        'status: $status -> upstream: $upstream_addr';

server {
    listen 80 default_server;
    listen [::]:80 default_server;

    server_name _;

    access_log /var/log/nginx/access.log upstream_log;

    location / {
        proxy_pass http://backend_servers;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
}
EOF

# 3. Проверка и перезапуск Nginx
sudo nginx -t && sudo systemctl restart nginx
```

* **Разбор флагов, директив и переменных:**
  * `cp <исходный_файл> <копия>` — создание резервной копии конфигурации перед редактированием.
  * `upstream backend_servers { ... }` — объявление именованной группы бэкенд-серверов на уровне контекста `http` (по умолчанию использует алгоритм балансировки Round Robin).
  * `log_format upstream_log ...` — создание пользовательского шаблона логирования:
    * `$remote_addr` — IP-адрес клиента, отправившего запрос.
    * `$time_local` — локальное время сервера в момент обработки запроса.
    * `$request` — полная исходная строка HTTP-запроса (метод, URI и версия протокола).
    * `$status` — итоговый HTTP-код ответа, отправленный клиенту.
    * `$upstream_addr` — IP-адрес и порт бэкенд-сервера, который фактически обработал запрос.
  * `access_log <путь> <формат>;` — включение записи журнала доступа в указанный файл с применением кастомного формата `upstream_log`.
  * `proxy_pass http://backend_servers;` — директива модуля `ngx_http_proxy_module`, перенаправляющая входящие запросы в группу серверов `backend_servers`.
  * `proxy_set_header Host $host;` — передача оригинального заголовка `Host` от клиента на бэкенд-сервер.
  * `proxy_set_header X-Real-IP $remote_addr;` — передача реального IP-адреса клиента в HTTP-заголовке `X-Real-IP` (иначе бэкенды будут видеть только IP балансировщика).

**Пример использования:** `sudo nginx -T | grep -E "upstream|proxy_pass"` (вывод итоговой активной конфигурации Nginx из памяти для быстрой проверки подключенных прокси-правил).

---

## 3. Проверка работы балансировщика

```bash
# 1. Серия тестовых запросов для проверки чередования ответов
for i in {1..6}; do curl -s http://localhost; echo ""; done

# 2. Мониторинг логов маршрутизации в реальном времени
sudo tail -f /var/log/nginx/access.log
```

* **Разбор флагов и команд:**
  * `for i in {1..6}; do ...; done` — цикл в Bash для выполнения заданного числа последовательных запросов.
  * `curl -s` (`--silent`) — тихий режим работы `curl`, скрывающий индикатор прогресса и статистику загрузки.
  * `tail -f` (`--follow`) — вывод последних строк файла и удержание его открытым для отображения новых записей по мере их появления (выход осуществляется комбинацией `Ctrl + C`).

**Пример использования:** `sudo tail -n 20 /var/log/nginx/access.log | awk '{print $NF}'` (вывод последних 20 строк лога с фильтрацией только последнего столбца — IP-адресов бэкендов, принявших запросы).

---

## 4. Добавление локального Backup-сервера

```bash
# 1. Создание HTML-страницы для резервного сервера
echo '<body style="background-color: grey;"><h1>Backup Server</h1></body>' | sudo tee /var/www/html/grey.html

# 2. Обновление конфигурации: алгоритм least_conn, backup-узел и локальный виртуальный хост на порту 8081
sudo tee /etc/nginx/sites-available/default << 'EOF'
upstream backend_servers {
    least_conn;
    server 192.168.1.21:80;
    server 192.168.1.22:80;
    server 127.0.0.1:8081 backup;
}

log_format upstream_log '$remote_addr - [$time_local] "$request" '
                        'status: $status -> upstream: $upstream_addr';

server {
    listen 80 default_server;
    listen [::]:80 default_server;

    server_name _;

    access_log /var/log/nginx/access.log upstream_log;

    location / {
        proxy_pass http://backend_servers;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
}

server {
    listen 8081;

    location / {
        root /var/www/html;
        index grey.html;
        try_files /grey.html =404;
    }
}
EOF

# 3. Проверка синтаксиса и применение изменений
sudo nginx -t && sudo systemctl restart nginx
```

* **Разбор флагов, параметров и директив:**
  * `least_conn;` — алгоритм балансировки (*Least Connections*), направляющий каждый новый запрос на сервер с наименьшим количеством текущих активных соединений.
  * `127.0.0.1:8081` — адрес локального интерфейса обратной петли (`loopback`) и нестандартный порт `8081`, на котором запущен резервный виртуальный хост.
  * `backup` — параметр сервера в группе `upstream`, помечающий узел как резервный: трафик на него не отправляется, пока доступен хотя бы один основной сервер; активируется автоматически при отказе всех основных узлов.
  * `listen 8081;` — настройка отдельного блока server на прослушивание локального порта 8081 для отдачи страницы-заглушки.

**Пример использования:** `sudo ss -tulpn | grep nginx` (проверка открытых сетевых сокетов, подтверждающая, что Nginx одновременно слушает порты 80 и 8081).
