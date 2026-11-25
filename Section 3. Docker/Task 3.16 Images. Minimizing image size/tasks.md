# Практика: Оптимизация слоев и сборка best-practice образов

## Содержание
- [Чекпоинт 1: Сборка Python-образа (Django)](#чекпоинт-1-сборка-python-образа-django)
- [Чекпоинт 2: Сборка Frontend-образов (Vue.js и React)](#чекпоинт-2-сборка-frontend-образов-vuejs-и-react)
- [Чекпоинт 3: Сборка Backend-образа (Java)](#чекпоинт-3-сборка-backend-образа-java)
- [Чекпоинт 4: Проверка истории слоев и размеров](#чекпоинт-4-проверка-истории-слоев-и-размеров)

---

## Чекпоинт 1: Сборка Python-образа (Django)
* **Задача:** Склонировать репозиторий `docker/awesome-compose` и собрать образ из директории `django/app` с тегом `django:latest`.
* **Команды:**
  ```bash
  git clone https://github.com && cd awesome-compose
  docker build -t django:latest django/app
  ```
* **Разбор флагов и пример использования:**
  * `-t <имя>:<тег>` — присваивает собираемому образу имя и тег.
  * `django/app` — путь к контексту сборки (директории с `Dockerfile` и исходниками).
  * *Пример запуска контейнера из образа:* `docker run -d -p 8000:8000 django:latest`

## Чекпоинт 2: Сборка Frontend-образов (Vue.js и React)
* **Задача:** Собрать образы `vuejs:latest` (из `vuejs/vuejs`) и `react:frontend` (из `react-java-mysql/frontend`).
* **Команды:**
  ```bash
  docker build -t vuejs:latest vuejs/vuejs
  docker build -t react:frontend react-java-mysql/frontend
  ```
* **Разбор флагов и нюансы:**
  * В старых примерах `vuejs/vuejs/Dockerfile` при конфликте версий Node.js в базовом образе (`node:14.4.0`) требуется обновить тег на `node:18-alpine` и добавить флаг `--ignore-engines` к команде `yarn install --immutable`.
  * *Пример переименования ошибочного тега без пересборки:* `docker tag java:frontends react:frontend && docker rmi java:frontends`

## Чекпоинт 3: Сборка Backend-образа (Java)
* **Задача:** Собрать multi-stage образ Java-бэкенда из `react-java-mysql/backend` с тегом `java:backend`.
* **Команда:**
  ```bash
  docker build -t java:backend react-java-mysql/backend
  ```
* **Механика выполнения:**
  * На этапе `builder` (`maven:3.8.5-eclipse-temurin-17`) скачиваются зависимости `pom.xml` и компилируется `.jar`.
  * На этапе `prepare-production` архив `.jar` распаковывается на слои (`BOOT-INF/lib`, `META-INF`, `BOOT-INF/classes`).
  * В финальный образ `eclipse-temurin:17-jre-focal` копируются только распакованные директории в `/app`, а кэш Maven и исходники отбрасываются.

## Чекпоинт 4: Проверка истории слоев и размеров
* **Задача:** Проверить список собранных образов и проанализировать размер каждого слоя.
* **Команды:**
  ```bash
  docker images
  docker history java:backend
  ```
* **Разбор команд:**
  * `docker images` (или `docker image ls`) — выводит локальные образы, их теги, `IMAGE ID` и общий виртуальный размер.
  * `docker history <образ>` — показывает послойную историю сборки образа, инструкцию (`CREATED BY`) и вес каждого отдельного слоя (`SIZE`).
