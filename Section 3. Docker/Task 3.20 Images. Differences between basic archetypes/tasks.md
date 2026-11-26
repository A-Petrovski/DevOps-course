# Практика: Task 3.20 — Различия базовых образов и оптимизация слоев

## Оглавление
- [1. Клонирование репозитория Apache ZooKeeper](#1-клонирование-репозитория-apache-zookeeper)
- [2. Создание Dockerfile.alpine и Dockerfile.buster](#2-создание-dockerfilealpine-и-dockerfilebuster)
- [3. Сборка образов, сравнение размеров и анализ слоев](#3-сборка-образов-сравнение-размеров-и-анализ-слоев)
- [4. Опционально: Оптимизация слоев через объединение RUN](#4-опционально-оптимизация-слоев-через-объединение-run)

---

## 1. Клонирование репозитория Apache ZooKeeper

Клонируем официальный репозиторий ZooKeeper в целевую директорию `/home/user/zookeeper`:

```bash
git clone --depth 1 https://github.com /home/user/zookeeper
```

**Разбор флагов и параметров:**
* `clone` — команда Git для создания локальной копии удаленного репозитория.
* `--depth 1` — создает поверхностный клон (*shallow clone*), загружая только последний коммит без всей многолетней истории изменений (значительно экономит трафик и время).
* `https://github.com` — URL официального репозитория Apache ZooKeeper.
* `/home/user/zookeeper` — локальная директория назначения.

**Пример использования:**
```bash
git clone --depth 1 --branch release-3.8.4 https://github.com /home/user/zookeeper
```

---

## 2. Создание Dockerfile.alpine и Dockerfile.buster

Создаем два файла в директории `/home/user/zookeeper` для сборки JAR-файла приложения с ручной установкой Apache Maven 3.8.8 через `curl` и пропуском этапа тестирования.

### Файл `/home/user/zookeeper/Dockerfile.alpine`

```dockerfile
FROM openjdk:8-jdk-alpine

RUN apk add --no-cache curl tar

RUN curl -fsSL https://apache.org | tar -xz -C /opt

ENV PATH="/opt/apache-maven-3.8.8/bin:\${PATH}"

WORKDIR /app
COPY . .

RUN mvn clean install -DskipTests -Denforcer.skip=true
```

### Файл `/home/user/zookeeper/Dockerfile.buster`

```dockerfile
FROM openjdk:8-jdk-buster

RUN curl -fsSL https://apache.org | tar -xz -C /opt

ENV PATH="/opt/apache-maven-3.8.8/bin:\${PATH}"

WORKDIR /app
COPY . .

RUN mvn clean install -DskipTests -Denforcer.skip=true
```

**Разбор инструкций и флагов:**
* `FROM` — задает базовый образ ОС с предустановленной JDK 8 (`alpine` на базе musl или `buster` на базе Debian 10).
* `apk add --no-cache curl tar` — устанавливает утилиты `curl` и `tar` в Alpine; флаг `--no-cache` запрещает сохранение локального индекса пакетов в `/var/cache/apk`, чтобы не раздувать слой.
* `curl -fsSL <URL>` — скачивает архив по сети:
  * `-f` (`--fail`) — прерывает выполнение с ошибкой при HTTP-статусах `4xx`/`5xx`, не передавая HTML-страницу ошибки дальше по пайпу.
  * `-s` (`--silent`) — отключает прогресс-бар загрузки в логах.
  * `-S` (`--show-error`) — выводит текст ошибки при сбое, несмотря на включенный тихий режим `-s`.
  * `-L` (`--location`) — автоматически следует за HTTP-редиректами (`301`/`302`) на зеркала Apache.
* `| tar -xz -C /opt` — принимает поток данных от `curl` через конвейер (`stdin`) и распаковывает его:
  * `-x` (`--extract`) — режим извлечения файлов.
  * `-z` (`--gzip`) — распаковка архива, сжатого алгоритмом `gzip` (`.tar.gz`).
  * `-C /opt` (`--directory`) — указывает целевую директорию распаковки.
* `ENV PATH="..."` — добавляет директорию с бинарным файлом `mvn` в системную переменную окружения `PATH`.
* `WORKDIR /app` — устанавливает рабочую директорию внутри контейнера (создает её, если она отсутствует).
* `COPY . .` — копирует содержимое контекста сборки (исходный код ZooKeeper) в текущую директорию `/app` внутри образа.
* `mvn clean install -DskipTests -Denforcer.skip=true` — запускает сборку проекта:
  * `clean` — очищает предыдущие артефакты сборки (`target/`).
  * `install` — компилирует код, упаковывает его в JAR-файлы и устанавливает в локальный репозиторий Maven.
  * `-DskipTests` — пропускает выполнение unit- и интеграционных тестов (ускоряет сборку).
  * `-Denforcer.skip=true` — отключает плагин проверки окружения `maven-enforcer-plugin`.

**Разбор сложных моментов и ошибок при выполнении:**
1. **Ошибка `gzip: invalid magic`:** Возникает, если в `curl` указать адрес главной страницы (`https://apache.org`) вместо прямой ссылки на `.tar.gz` архив. Утилита `tar` пытается прочитать HTML-разметку как бинарный gzip-поток и падает.
2. **Несовместимость Maven 4 (`MavenSession.getContainer() is null`):** Экспериментальная ветка Maven `4.0.0-rc` несовместима со старым плагином `exec-maven-plugin:1.6.0` в ZooKeeper. Необходимо использовать стабильную ветку **Maven 3** (`3.8.8` или `3.9.x`).
3. **Ошибка плагина `maven-enforcer-plugin` (`zookeeper-min-maven-min-java`):** В свежих коммитах репозитория ZooKeeper добавлено правило проверки минимальной версии Java (требуется Java 11/17), из-за чего сборка на требуемой по ТЗ Java 8 блокируется. Флаг `-Denforcer.skip=true` обходит эту проверку.
4. **Несохраненные изменения в редакторе:** Если в IDE/редакторе на вкладке файла горит индикатор несохраненных изменений (`Ctrl + S`), `docker build` будет читать старую версию `Dockerfile` с диска.

**Пример использования:**
```bash
curl -fsSL https://apache.org | tar -xz -C /opt
```

---

## 3. Сборка образов, сравнение размеров и анализ слоев

Переходим в директорию с контекстом сборки, собираем оба образа и анализируем их размеры и историю слоев:

```bash
cd /home/user/zookeeper
docker build -f Dockerfile.alpine -t zookeeper-alpine:latest .
docker build -f Dockerfile.buster -t zookeeper-buster:latest .

docker images | grep zookeeper
docker history zookeeper-alpine:latest
docker history zookeeper-buster:latest
```

**Разбор команд и флагов:**
* `-f <имя_файла>` (`--file`) — явно указывает путь к Dockerfile с нестандартным именем (по умолчанию Docker ищет файл `Dockerfile`).
* `-t <имя:тег>` (`--tag`) — присваивает собранному образу имя и тег.
* `.` (точка в конце) — указывает текущую директорию (`/home/user/zookeeper`) в качестве контекста сборки (*build context*), который передается демону Docker.
* `docker images` — выводит список локально сохраненных образов, их теги, ID и итоговый виртуальный размер (`SIZE`).
* `docker history <образ>` — отображает послойную структуру образа, команду, создавшую каждый слой, и объем дискового пространства, занятый каждым слоем.

**Выводы по результатам сравнения:**
* Образ `zookeeper-alpine:latest` занимает значительно меньше места (разница составляет ~200–350 МБ), так как базовый образ Alpine (~5.5 МБ) не содержит тяжелых библиотек GNU и утилит Debian Buster (~114 МБ).
* В выводе `docker history` наибольший объем в обоих образах занимает слой компиляции (`RUN mvn clean install...`), так как Maven выкачивает сотни мегабайт зависимостей в скрытую директорию `/root/.m2`.

**Пример использования:**
```bash
docker history --no-trunc --human zookeeper-alpine:latest
```

---

## 4. Опционально: Оптимизация слоев через объединение RUN

Объединяем скачивание Maven, сборку JAR-файлов и удаление временных файлов (самого Maven, кэша зависимостей `.m2` и утилит `curl`/`tar`) в **одну инструкцию `RUN`** в `Dockerfile.alpine`:

```dockerfile
FROM openjdk:8-jdk-alpine

WORKDIR /app
COPY . .

RUN apk add --no-cache curl tar && \
    curl -fsSL https://apache.org | tar -xz -C /opt && \
    /opt/apache-maven-3.8.8/bin/mvn clean install -DskipTests -Denforcer.skip=true && \
    rm -rf /opt/apache-maven-3.8.8 /root/.m2 && \
    apk del curl tar
```

Пересобираем образ и оцениваем новый размер:

```bash
docker build -f Dockerfile.alpine -t zookeeper-alpine:optimized .
docker images | grep zookeeper-alpine
docker history zookeeper-alpine:optimized
```

**Разбор механизмов и флагов:**
* `&&` — оператор логического «И» в shell: следующая команда выполняется только при успешном завершении предыдущей (код возврата `0`).
* `\` — символ переноса строки для читаемости многострочной команды в Dockerfile.
* `rm -rf /opt/apache-maven-3.8.8 /root/.m2` — принудительно (`-f`) и рекурсивно (`-r`) удаляет распакованный Maven и кэш загруженных библиотек из домашней директории `root`.
* `apk del curl tar` — удаляет пакеты, которые требовались только на этапе сборки.
* **Нюанс работы OverlayFS:** Слои в Docker являются неизменяемыми (*read-only*). Если удалить файлы в отдельной последующей инструкции `RUN`, они останутся в предыдущем слое и лишь скроются через *whiteout-файл*, не уменьшив размер образа. Удаление внутри той же инструкции `RUN` очищает данные до фиксации (commit) слоя.

**Пример использования:**
```bash
docker system df -v
```
