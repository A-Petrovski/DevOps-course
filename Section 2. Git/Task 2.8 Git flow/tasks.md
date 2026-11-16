# Практика: Реализация рабочего процесса Git Flow и Merge Requests

## Оглавление
- [1. Создание локального репозитория и работа с ветками master и develop](#1-создание-локального-репозитория-и-работа-с-ветками-master-и-develop)
- [2. Создание ветки feature, параллельные коммиты и загрузка на GitLab](#2-создание-ветки-feature-параллельные-коммиты-и-загрузка-на-gitlab)
- [3. Создание и слияние Merge Request'ов через терминал](#3-создание-и-слияние-merge-requestов-через-терминал)
- [4. Создание тега release_1.1 и отправка в репозиторий](#4-создание-тега-release_11-и-отправка-в-репозиторий)
- [Разбор типовых ошибок и нюансов](#разбор-типовых-ошибок-и-нюансов)

---

## 1. Создание локального репозитория и работа с ветками master и develop

Создаем каталог проекта, инициализируем репозиторий, формируем начальную ветку `master` с двумя коммитами и ответвляем от нее `develop`.

```bash
# Создание директории и инициализация Git
mkdir git-flow-repo && cd git-flow-repo
git init
git branch -M master

# Первый коммит в master
echo "First line" > README.md
git add README.md
git commit -m "First commit in master"

# Второй коммит в master
echo "Second line" >> README.md
git add README.md
git commit -m "Second commit in master"

# Создание ветки develop (перенос изменений из master) и добавление коммита
git checkout -b develop
echo "Third line" >> README.md
git add README.md
git commit -m "Third commit in develop"
```

**Разбор флагов и параметров:**
- `mkdir ... && cd ...` — создает директорию и при успешном выполнении сразу переходит в нее.
- `git init` — создает скрытую директорию `.git` и инициализирует пустой локальный репозиторий.
- `git branch -M master` — принудительно переименовывает текущую ветку (по умолчанию часто `main`) в `master`. Флаг `-M` является сокращением от `--move --force`.
- `git add <файл>` — добавляет изменения указанного файла в индекс (staging area) для подготовки к коммиту.
- `git commit -m "<сообщение>"` — фиксирует проиндексированные изменения в истории репозитория. Флаг `-m` (`--message`) позволяет передать текст коммита прямо в командной строке.
- `git checkout -b develop` — создает новую ветку `develop` от текущего коммита и сразу переключает на нее указатель `HEAD`. Флаг `-b` (`--branch`) отвечает за создание новой ветки перед переключением.

**Пример использования:** `git switch -c develop` (современный аналог `git checkout -b develop`, где флаг `-c` означает `--create`).

---

## 2. Создание ветки feature, параллельные коммиты и загрузка на GitLab

Ответвляем `feature` от `develop`, делаем по одному коммиту в каждой из веток и связываем локальный репозиторий с удаленным сервером GitLab.

```bash
# Создание ветки feature из develop и добавление коммита
git checkout -b feature
echo "Fourth line" >> README.md
git add README.md
git commit -m "Fourth commit in feature"

# Возврат в ветку develop (БЕЗ флага -b) и добавление отдельного коммита
git checkout develop
echo "Commit specifically for develop" > develop.txt
git add develop.txt
git commit -m "Commit in develop"

# Привязка удаленного репозитория и отправка всех веток на GitLab
git remote add origin git@gitlab.com:devops7071547/git-flow-repo.git
git push -u origin master
git push -u origin develop
git push -u origin feature
```

**Разбор флагов и параметров:**
- `git checkout develop` — переключает рабочую директорию на уже существующую ветку `develop`.
- `git remote add origin <URL>` — регистрирует удаленный репозиторий по указанному SSH-адресу под псевдонимом `origin`.
- `git push -u origin <ветка>` — отправляет локальные коммиты в удаленный репозиторий `origin`. Флаг `-u` (`--set-upstream`) связывает локальную ветку с удаленной, чтобы в будущем можно было выполнять `git push` и `git pull` без аргументов.

**Пример использования:** `git push -u origin --all` (отправляет на удаленный сервер сразу все локальные ветки одной командой).

---

## 3. Создание и слияние Merge Request'ов через терминал

В GitLab аналогом Pull Request является **Merge Request (MR)**. Создать и автоматически смержить их из терминала можно с помощью встроенных параметров отправки Git (`Push Options`) или через утилиту `glab`.

```bash
# Вариант 1: Использование встроенных Git Push Options (не требует сторонних утилит)
# 1. Создание MR из feature в develop и автоматическое слияние
git push -o merge_request.create -o merge_request.target=develop -o merge_request.merge_when_pipeline_succeeds origin feature

# (Пауза 3-5 секунд для обработки слияния на стороне GitLab)

# 2. Создание MR из develop в master и автоматическое слияние
git push -o merge_request.create -o merge_request.target=master -o merge_request.merge_when_pipeline_succeeds origin develop
```

```bash
# Вариант 2: Использование официального клиента GitLab CLI (glab)
glab mr create --source-branch feature --target-branch develop --title "Merge feature into develop" --yes
glab mr merge feature --yes

glab mr create --source-branch develop --target-branch master --title "Merge develop into master" --yes
glab mr merge develop --yes
```

**Разбор флагов и параметров:**
- `-o <опция>` (`--push-option`) — передает серверу GitLab инструкции во время выполнения `git push`.
- `merge_request.create` — указывает GitLab автоматически создать новый Merge Request для отправляемой ветки.
- `merge_request.target=<ветка>` — задает целевую ветку, в которую будут влиты изменения.
- `merge_request.merge_when_pipeline_succeeds` — ставит MR в очередь на автоматическое слияние сразу после успешного выполнения проверок (или немедленно, если CI/CD не настроен).
- `--source-branch` / `--target-branch` — флаги `glab`, определяющие ветку-источник и целевую ветку для MR.
- `--title` — задает заголовок Merge Request.
- `--yes` (`-y`) — автоматически подтверждает действие без интерактивного запроса в терминале.

**Пример использования:** `git push -o merge_request.create -o merge_request.target=develop -o merge_request.remove_source_branch origin feature` (создаст MR и автоматически удалит ветку `feature` после слияния).

---

## 4. Создание тега release_1.1 и отправка в репозиторий

После слияния `develop` в `master` на сервере необходимо обновить локальную ветку `master`, создать аннотированный тег и отправить его на GitLab.

```bash
# Переключение на master и получение актуального состояния с сервера
git checkout master
git pull origin master

# Создание аннотированного тега и его отправка в удаленный репозиторий
git tag -a release_1.1 -m "Release 1.1"
git push origin release_1.1
```

**Разбор флагов и параметров:**
- `git pull origin master` — скачивает изменения из удаленной ветки `master` (включая merge-коммит, созданный на GitLab) и объединяет их с локальной веткой.
- `git tag -a <имя_тега> -m "<описание>"` — создает аннотированный тег на текущем коммите. Флаг `-a` (`--annotate`) записывает в тег метаданные (автора, дату и сообщение), а `-m` задает текст аннотации.
- `git push origin release_1.1` — отправляет конкретный тег на удаленный сервер (по умолчанию `git push` теги не отправляет).

**Пример использования:** `git push origin --tags` (отправляет в удаленный репозиторий сразу все локальные теги).

---

## Разбор типовых ошибок и нюансов

1. **Ошибка `fatal: not a valid object name: 'master'` при создании ветки сразу после `git init`:**
   В пустом репозитории до первого коммита ветки физически не существуют, так как указателю `HEAD` не на что ссылаться. Сначала необходимо сделать хотя бы один коммит (`git commit`), и только после этого создавать новые ветки.
2. **Ошибка `fatal: a branch named 'develop' already exists` при переключении веток:**
   Флаг `-b` в команде `git checkout -b develop` используется только один раз — в момент создания новой ветки. Для возврата на уже существующую ветку команду нужно вызывать без флага: `git checkout develop`.
3. **Ошибка `fatal: The current branch feature has no upstream branch` при первом `git push`:**
   Локальная ветка изначально не связана с веткой на сервере. При первой отправке новой ветки необходимо явно указать удаленный репозиторий, имя ветки и флаг привязки: `git push -u origin feature`.
4. **Предотвращение конфликтов слияния при параллельной работе:**
   Если в ветках `feature` и `develop` изменить одни и те же строки одного файла (`README.md`), при автоматическом слиянии возникнет конфликт (`Merge conflict`). Изменение разных файлов (например, `README.md` в `feature` и `develop.txt` в `develop`) позволяет провести слияние автоматически.
