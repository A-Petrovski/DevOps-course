# Task 3.14. Images. Multistage (Практика)

## Оглавление
- [Чекпоинт 1: Подготовка исходного файла main.go](#чекпоинт-1-подготовка-исходного-файла-maingo)
- [Чекпоинт 2: Multi-stage Dockerfile для Go-приложения](#чекпоинт-2-multi-stage-dockerfile-для-go-приложения)
- [Чекпоинт 3: Сборка, анализ истории слоев и публикация в GitLab](#чекпоинт-3-сборка-анализ-истории-слоев-и-публикация-в-gitlab)
- [Чекпоинт 4–5: Безымянный этап сборки и передача секрета через ARG](#чекпоинт-45-безымянный-этап-сборки-и-передача-секрета-через-arg)
- [Чекпоинт 6–7: Клонирование Grafana и раздельная сборка (--target)](#чекпоинт-67-клонирование-grafana-и-раздельная-сборка---target)

---

## Чекпоинт 1: Подготовка исходного файла main.go

✅ **Выполнено**

### Задача
Подготовить тестовый файл `main.go` в рабочей директории проекта для проверки скачивания внешних зависимостей через `go get` в режиме `GO111MODULE=auto`.

### Рабочие команды
```bash
mkdir -p /home/amp/task14/gocalc && cd /home/amp/task14/gocalc

cat << 'EOF' > main.go
package main

import (
	"fmt"
	"://github.com"
)

func main() {
	fmt.Println(uuid.New().String())
}
EOF
```

### Разбор флагов и команд
* `mkdir -p` — создает целевую директорию вместе со всеми отсутствующими родительскими каталогами без ошибки, если директория уже существует.
* `cat << 'EOF' > main.go` — перенаправляет многострочный текст в файл `main.go` до встречи маркера `EOF`. Одинарные кавычки вокруг `'EOF'` запрещают оболочке bash интерпретировать символы `$` внутри блока.

---

## Чекпоинт 2: Multi-stage Dockerfile для Go-приложения

✅ **Выполнено**

### Задача
Написать `Dockerfile` с многоэтапной сборкой: компиляция в `golang:1.19.1-alpine` (с `GO111MODULE=auto`, скачиванием зависимостей `go get` и сборкой бинарника `app`), запуск в `alpine:3.10.3`.

### Рабочий `Dockerfile`
```dockerfile
FROM golang:1.19.1-alpine AS builder
ENV GO111MODULE=auto
WORKDIR /go/src/gocalc
COPY main.go .
RUN apk add --no-cache git && go get . && CGO_ENABLED=0 go build -o app .

FROM alpine:3.10.3
WORKDIR /app
COPY --from=builder /go/src/gocalc/app .
CMD ["./app"]
```

### Важные технические нюансы и разбор директив
* `FROM golang:1.19.1-alpine AS builder` — задает базовый образ сборщика и присваивает этапу псевдоним `builder`.
* `ENV GO111MODULE=auto` — при отсутствии файла `go.mod` переключает Go в классический режим работы через `$GOPATH` (`/go`).
* `WORKDIR /go/src/gocalc` — **критически важный путь**: в режиме `GO111MODULE=auto` без `go.mod` сборка обязана происходить внутри каталога `$GOPATH/src` (`/go/src/...`), иначе `go get` завершится ошибкой `outside GOPATH`.
* `apk add --no-cache git` — в образе `golang:1.19.1-alpine` отсутствует утилита `git`, без которой `go get` не сможет склонировать пакет с GitHub. Флаг `--no-cache` не сохраняет индекс пакетов на диск, уменьшая размер слоя.
* `go get .` — скачивает все внешние зависимости, указанные в блоке `import` файлов текущей директории.
* `CGO_ENABLED=0` — отключает динамическую линковку с системными C-библиотеками (`musl libc`), создавая полностью автономный статический бинарный файл (исключает ошибку `no such file or directory` при запуске в другой версии Alpine).
* `go build -o app .` — компилирует пакет в текущей директории (`.`), флаг `-o app` задает имя выходного исполняемого файла.
* `COPY --from=builder /go/src/gocalc/app .` — копирует скомпилированный файл не из локального контекста сборки, а из файловой системы предыдущего этапа `builder`.

---

## Чекпоинт 3: Сборка, анализ истории слоев и публикация в GitLab

✅ **Выполнено**

### Рабочие команды
```bash
sudo docker build -t gocalc:v1 .
sudo docker images | grep gocalc
sudo docker history gocalc:v1
sudo docker run --rm gocalc:v1

git add .
git commit -m "Task 3.14: add multi-stage Dockerfile for gocalc"
git push
```

### Разбор флагов и команд
* `docker build -t gocalc:v1 .` — собирает образ из `Dockerfile` в текущей директории (`.`), флаг `-t` (`--tag`) присваивает образу имя и тег.
* `docker history gocalc:v1` — выводит историю слоев итогового образа. Показывает, что слои этапа `builder` (исходники, кэш `go`, пакет `git`) отсутствуют в финальном образе.
* `docker run --rm gocalc:v1` — запускает контейнер для проверки и автоматически удаляет его (`--rm`) сразу после завершения работы.

---

## Чекпоинт 4–5: Безымянный этап сборки и передача секрета через ARG

✅ **Выполнено**

### Задача
Убрать именование этапа (`AS builder`), использовать обращение по индексу (`--from=0`), передать секрет через `ARG` в первый образ и записать его в файл внутри конечного образа.

### Рабочий `Dockerfile`
```dockerfile
FROM golang:1.19.1-alpine
ARG APP_SECRET
ENV GO111MODULE=auto
WORKDIR /go/src/gocalc
COPY main.go .
RUN apk add --no-cache git && \
    go get . && \
    CGO_ENABLED=0 go build -o app . && \
    echo "${APP_SECRET}" > /go/src/gocalc/secret.txt

FROM alpine:3.10.3
WORKDIR /app
COPY --from=0 /go/src/gocalc/app .
COPY --from=0 /go/src/gocalc/secret.txt .
CMD ["./app"]
```

### Команды сборки и проверки
```bash
sudo docker build --build-arg APP_SECRET="my_secret_123" -t gocalc:v2 .
sudo docker run --rm gocalc:v2 cat /app/secret.txt
sudo docker history gocalc:v2
```

### Разбор директив и флагов
* `ARG APP_SECRET` — объявляет переменную времени сборки. Область видимости `ARG` ограничена текущим блоком `FROM` и сбрасывается при переходе к следующему этапу.
* `COPY --from=0` — обращается к первому этапу сборки по его порядковому индексу (нумерация этапов `FROM` начинается с `0`).
* `--build-arg APP_SECRET="my_secret_123"` — передает значение переменной `ARG` в демон Docker во время запуска `docker build`.

---

## Чекпоинт 6–7: Клонирование Grafana и раздельная сборка (--target)

✅ **Выполнено**

### Задача
Склонировать ветку `v6.3.x` официального репозитория Grafana, добавить этап на базе `nginx:alpine` со скомпилированной статикой (`public`) и собрать раздельно образы `grafana:app` и `grafana:static`.

### Анализ этапов оригинального `Dockerfile` Grafana `v6.3.x`
В файле определены 3 последовательных этапа:
1. Индекс `0`: `FROM golang:1.12.9-stretch` (сборка бэкенда).
2. Индекс `1`: `FROM node:10.14.2` (сборка фронтенда, результат в `/usr/src/app/public`).
3. Индекс `2`: `FROM ubuntu:18.04` (финальный образ приложения Grafana).

### Рабочие команды модификации и сборки
```bash
git clone -b v6.3.x --depth 1 https://github.com /home/amp/task14/grafana
cd /home/amp/task14/grafana

# Присваиваем имя AS app финальному этапу приложения (ubuntu:18.04)
sed -i 's/^FROM ubuntu:18.04$/FROM ubuntu:18.04 AS app/' Dockerfile

# Добавляем в конец файла этап static, копирующий папку public из этапа с индексом 1
cat << 'EOF' >> Dockerfile

FROM nginx:alpine AS static
COPY --from=1 /usr/src/app/public /usr/share/nginx/html
EOF

# Раздельная сборка образов по целевым этапам
sudo docker build --target static -t grafana:static .
sudo docker build --target app -t grafana:app .

# Проверка списка образов
sudo docker images | grep grafana
```

### Разбор флагов и команд
* `git clone -b v6.3.x --depth 1` — флаг `-b` (`--branch`) выбирает конкретную ветку репозитория; `--depth 1` выполняет поверхностное клонирование (только последний коммит без всей истории), что значительно экономит время и трафик.
* `--target static` / `--target app` — указывает `docker build` остановить сборку на конкретном именованном этапе `Dockerfile`, игнорируя этапы, расположенные ниже, или неиспользуемые ветки.
