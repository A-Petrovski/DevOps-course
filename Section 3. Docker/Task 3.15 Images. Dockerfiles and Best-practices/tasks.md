# Практика: Task 3.15 — Несколько Dockerfile и Best-practices

## Оглавление
- [1. Создание репозитория на GitLab и перенос кода из alonetone](#1-создание-репозитория-на-gitlab-и-перенос-кода-из-alonetone)
- [2. Написание Dockerfile.dev и Dockerfile.prod](#2-написание-dockerfiledev-и-dockerfileprod)
- [3. Сборка образов с тегами alonetone:dev и alonetone:prod](#3-сборка-образов-с-тегами-alonedev-и-alonetoneprod)
- [4. Вывод списка образов и фиксация изменений в Git](#4-вывод-списка-образов-и-фиксация-изменений-в-git)
- [5. Разбор типичных ошибок и нюансов из практики](#5-разбор-типичных-ошибок-и-нюансов-из-практики)

---

## 1. Создание репозитория на GitLab и перенос кода из alonetone

После создания пустого репозитория `dkr-17-alonetone` в веб-интерфейсе GitLab и его клонирования на локальную машину, переносим кодовую базу из официального репозитория проекта `alonetone`:

```bash
cd dkr-17-alonetone
git remote add upstream https://github.com/sudara/alonetone.git
git fetch upstream
git pull upstream main --allow-unrelated-histories
git push -u origin main
git remote remove upstream
```

**Разбор команд и флагов:**
- `git remote add upstream <url>` — добавляет внешний репозиторий `alonetone` как второй удалённый источник с именем `upstream` (ваш GitLab остаётся под именем `origin`).
- `git fetch upstream` — загружает ветки и коммиты из `upstream` в локальный кэш `.git` без изменения рабочих файлов.
- `git pull upstream main --allow-unrelated-histories` — скачивает и вливает ветку `main` из `upstream` в текущую ветку. Флаг `--allow-unrelated-histories` разрешает слияние двух репозиториев с разной начальной историей (например, если GitLab создал свой `README.md`).
- `git push -u origin main` — отправляет перенесённый код в ваш репозиторий на GitLab. Флаг `-u` (`--set-upstream`) связывает локальную ветку `main` с удалённой `origin/main`.
- `git remote remove upstream` — удаляет привязку к исходному репозиторию, чтобы исключить случайную отправку изменений в чужой проект.

**Пример использования:**
```bash
git remote -v
```

---

## 2. Написание Dockerfile.dev и Dockerfile.prod

### Анализ `.github/workflows/specs.yml`
Из CI-конфигурации проекта `alonetone` извлекаем ключевые требования для сборки:
- **Версия Ruby:** `4.0.10` (не ниже 2.6 по условию задания).
- **Системные пакеты:** `libvips-dev`, `rustc`, а также клиентская библиотека MySQL (`default-libmysqlclient-dev`), `nodejs` и `yarn` (в GitHub Actions они предустановлены в образе раннера, но в минимальном образе `ruby` их нужно поставить явно).
- **Исключения:** Запуск MySQL-сервера (`systemctl start mysql.service`) и прогон тестов (`rspec`, `jest`, `playwright`) в образ не включаем — БД по Best-practices выносится в отдельный контейнер.

### Создание файлов в корне проекта
```bash
cd ~/task15/dkr-17-alonetone
touch Dockerfile.dev Dockerfile.prod
```

### Конфигурация `Dockerfile.dev` (с development-зависимостями)
```dockerfile
FROM ruby:4.0.10

ENV RAILS_ENV=development

RUN apt-get update -qq && apt-get install -y --no-install-recommends \
    build-essential \
    default-libmysqlclient-dev \
    libvips-dev \
    rustc \
    nodejs \
    npm \
    && npm install -g yarn \
    && rm -rf /var/lib/apt/lists/*

WORKDIR /app

COPY Gemfile Gemfile.lock package.json yarn.lock ./

RUN bundle install --jobs 4 --retry 3

RUN yarn install --network-timeout 600000

COPY . .
```

### Конфигурация `Dockerfile.prod` (без development/test-зависимостей)
```dockerfile
FROM ruby:4.0.10

ENV RAILS_ENV=production

RUN apt-get update -qq && apt-get install -y --no-install-recommends \
    build-essential \
    default-libmysqlclient-dev \
    libvips-dev \
    rustc \
    nodejs \
    npm \
    && npm install -g yarn \
    && rm -rf /var/lib/apt/lists/*

WORKDIR /app

COPY Gemfile Gemfile.lock package.json yarn.lock ./

RUN bundle config set --local without 'development test' \
    && bundle install --jobs 4 --retry 3

RUN yarn install --production --network-timeout 600000

COPY . .
```

**Разбор инструкций и флагов:**
- `FROM ruby:4.0.10` — задаёт базовый официальный образ Ruby нужной версии.
- `ENV RAILS_ENV=...` — устанавливает переменную окружения внутри образа, определяющую режим работы Rails (`development` или `production`).
- `apt-get update -qq` — обновляет индекс пакетов в тихом режиме (`-qq` подавляет лишний вывод).
- `apt-get install -y --no-install-recommends` — устанавливает пакеты без интерактивного подтверждения (`-y`) и без необязательных рекомендаций (`--no-install-recommends`), уменьшая вес слоя.
- `rm -rf /var/lib/apt/lists/*` — очищает кэш списков пакетов `apt` в том же слое `RUN`, чтобы не раздувать размер образа.
- `WORKDIR /app` — создаёт директорию `/app` и делает её текущей рабочей папкой для всех последующих команд (`COPY`, `RUN`).
- `COPY Gemfile Gemfile.lock package.json yarn.lock ./` — копирует только файлы описания зависимостей до основного кода, чтобы Docker кэшировал тяжёлый слой установки библиотек при изменении исходников.
- `bundle config set --local without 'development test'` — настраивает Bundler в локальной конфигурации проекта так, чтобы исключить установку гемов из групп `development` и `test`.
- `--jobs 4 --retry 3` — запускает установку гемов в 4 параллельных потока и делает до 3 повторных попыток при сетевых сбоях.
- `yarn install --production` — устанавливает только runtime-пакеты из `dependencies`, игнорируя `devDependencies` из `package.json`.
- `--network-timeout 600000` — увеличивает таймаут сетевых запросов Yarn до 600 000 мс (10 минут), предотвращая обрывы при медленном соединении.
- `COPY . .` — копирует все остальные файлы из текущей директории хоста (`.`) в текущую директорию контейнера (`/app`).

**Пример использования:**
```bash
bundle config list
```

---

## 3. Сборка образов с тегами alonetone:dev и alonetone:prod

Запускаем сборку двух образов из корня репозитория, явно указывая путь к нужному файлу инструкций:

```bash
docker build -f Dockerfile.dev -t alonetone:dev .
docker build -f Dockerfile.prod -t alonetone:prod .
```

**Разбор команд и флагов:**
- `docker build` — команда сборки Docker-образа из инструкций Dockerfile.
- `-f` (`--file`) — указывает имя или путь к конкретному Dockerfile (по умолчанию Docker ищет файл строго с именем `Dockerfile`).
- `-t` (`--tag`) — присваивает собранному образу имя и тег в формате `имя:тег` (`alonetone:dev` и `alonetone:prod`).
- `.` (точка в конце) — задаёт **контекст сборки** (текущую директорию), содержимое которой Docker CLI передаёт демону Docker для выполнения инструкций `COPY`.

**Пример использования:**
```bash
docker build --no-cache -f Dockerfile.prod -t alonetone:prod .
```

---

## 4. Вывод списка образов и фиксация изменений в Git

Проверяем собранные образы, их теги и разницу в размере, после чего отправляем созданные Dockerfile в репозиторий на GitLab:

```bash
docker images
git add Dockerfile.dev Dockerfile.prod
git commit -m "Add Dockerfile.dev and Dockerfile.prod"
git push origin main
```

**Разбор команд и флагов:**
- `docker images` — выводит таблицу всех локальных образов (репозиторий, тег, ID образа, время создания и размер `SIZE`). Образ `alonetone:prod` будет занимать меньше места за счёт отсутствия dev/test-библиотек.
- `git add` — добавляет новые файлы в индекс Git.
- `git commit -m "..."` — фиксирует изменения в локальной истории репозитория с указанным сообщением (`-m`).
- `git push origin main` — отправляет коммит на удалённый сервер GitLab.

**Пример использования:**
```bash
docker images alonetone
```

---

## 5. Разбор типичных ошибок и нюансов из практики

1. **Создание файлов в корне системы вместо папки проекта:**
   - *Ошибка:* Выполнение `touch /Dockerfile.dev` создаёт файл в корне файловой системы Linux (`/`), требуя прав `sudo`.
   - *Решение:* Использовать относительный путь без начального слеша (`touch Dockerfile.dev`) и предварительно удалить лишние файлы из `/` (`sudo rm /Dockerfile.dev /Dockerfile.prod`).

2. **Нехватка места на корневом разделе (`no space left on device`):**
   - *Ошибка:* При экспорте слоёв образа заканчивается место на разделе `/` (например, при объёме диска 15 ГБ), хотя рядом примонтирован свободный диск `/mnt/data`.
   - *Решение:* Перенести рабочие директории Docker и containerd на свободный диск с созданием символических ссылок:
     ```bash
     sudo systemctl stop docker docker.socket containerd
     sudo mv /var/lib/docker /mnt/data/docker
     sudo mv /var/lib/containerd /mnt/data/containerd
     sudo ln -s /mnt/data/docker /var/lib/docker
     sudo ln -s /mnt/data/containerd /var/lib/containerd
     sudo systemctl start containerd docker
     docker builder prune -af
     ```

3. **Сетевой таймаут при установке JS-пакетов (`ESOCKETTIMEDOUT` / `ENOENT` в Yarn):**
   - *Ошибка:* При нестабильном соединении с `registry.yarnpkg.com` загрузка прерывается, и из-за объединения команд в один `RUN` приходится заново ждать выполнения `bundle install`.
   - *Решение:* Разделить `bundle install` и `yarn install` на две независимые инструкции `RUN` (чтобы закэшировать слой гемов) и добавить флаг `--network-timeout 600000` для Yarn.
