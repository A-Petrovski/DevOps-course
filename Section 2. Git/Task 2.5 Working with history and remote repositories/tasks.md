# Практическое выполнение Task 2.5

## Оглавление
- [1. Перенос репозитория, объединение коммитов (Squash) и Force Push](#1-перенос-репозитория-объединение-коммитов-squash-и-force-push)
- [2. Перенос репозитория git-rebase и создание 2 Merge Requests](#2-перенос-репозитория-git-rebase-и-создание-2-merge-requests)
- [3. Перенос репозитория git-merge и просмотр конфликтов в GitLab](#3-перенос-репозитория-git-merge-и-просмотр-конфликтов-в-gitlab)
- [4. Fork репозитория git-checkout, коммит в ветку feature и создание Merge Request](#4-fork-репозитория-git-checkout-коммит-в-ветку-feature-и-создание-merge-request)

---

## 1. Перенос репозитория, объединение коммитов (Squash) и Force Push

Предварительно создаем пустой репозиторий `devops-task-force` на GitLab/GitHub (без файла `README.md`, чтобы не создавать лишний начальный коммит на сервере).

```bash
git clone <URL_репозитория_git-squash>
cd git-squash
git checkout develop
git remote remove origin
git remote add origin <URL_вашего_репозитория_devops-task-force>
git push -u origin --all
git push origin --tags

git rebase -i --root
git push origin develop
git push --force origin develop
```

**Разбор использованных команд и флагов:**
* `git clone <URL>` — клонирует удаленный репозиторий в новую локальную директорию.
* `git checkout develop` — переключается на ветку `develop` (если ветки нет локально, автоматически создает отслеживаемую ветку из `origin/develop`).
* `git remote remove origin` — удаляет привязку к исходному удаленному репозиторию.
* `git remote add origin <URL>` — добавляет ваш новый пустой репозиторий в качестве основного удаленного сервера `origin`.
* `git push -u origin --all` — отправляет все локальные ветки в новый репозиторий и связывает их для отслеживания (`-u` / `--set-upstream`).
* `git push origin --tags` — отправляет все локальные теги на сервер.
* `git rebase -i --root` — запускает интерактивное (`-i`) перебазирование начиная с самого первого корневого коммита (`--root`) в истории ветки. В открывшемся редакторе у первого коммита оставляется `pick`, а у всех последующих `pick` заменяется на `squash` (или `s`).
* `git push --force origin develop` — принудительно перезаписывает историю ветки `develop` на удаленном сервере локальной версией.

**Разбор ошибок и нюансов:**
* **Ошибка при обычном `git push origin develop` после `squash`:**
  ```text
  ! [rejected]        develop -> develop (non-fast-forward)
  error: failed to push some refs to '<URL>'
  hint: Updates were rejected because the tip of your current branch is behind its remote counterpart.
  ```
  *Причина:* После `squash` хэши коммитов изменились, и история локальной ветки перестала быть прямым продолжением истории на сервере (`non-fast-forward`). Обычный `push` блокируется для защиты от потери коммитов.

**Пример использования:** `git push --force-with-lease origin develop` (более безопасный аналог `--force`, который отклонит отправку, если другой разработчик успел залить новые коммиты в эту ветку на сервер).

---

## 2. Перенос репозитория git-rebase и создание 2 Merge Requests

Предварительно создаем пустой репозиторий `git-rebase` в своем аккаунте GitLab без инициализации `README`.

```bash
git clone <URL_исходного_репозитория_git-rebase>
cd git-rebase
git remote set-url origin <URL_вашего_репозитория_на_GitLab>

git checkout master
git push -u origin master

git checkout develop
git push -o merge_request.create -o merge_request.target=master origin develop

git checkout feature
git push -o merge_request.create -o merge_request.target=develop origin feature
```

**Разбор использованных команд и флагов:**
* `git remote set-url origin <URL>` — заменяет адрес существующего удаленного репозитория `origin` без необходимости удалять и заново добавлять его.
* `-o merge_request.create` — опция сервера GitLab (`push option`), дающая команду автоматически создать Merge Request для отправляемой ветки.
* `-o merge_request.target=<ветка>` — указывает целевую ветку, в которую будет направлен создаваемый Merge Request.

**Разбор ошибок и нюансов:**
* **Ошибка `Target branch ... does not exist`:** Возникает, если отправлять ветку `feature` с флагом создания MR в `develop` раньше, чем сама ветка `develop` была загружена на сервер GitLab. Всегда пушьте ветки в порядке иерархии: сначала `master`, затем `develop`, затем `feature`.
* **Ошибка `You must select different branches`:** Возникает при попытке отправить ветку `develop` с опцией `-o merge_request.target=develop` (попытка создать MR из ветки в саму себя).
* **Проблема `Everything up-to-date` при повторном запуске `git push -o ...`:** Опции `-o` обрабатываются сервером GitLab **только при фактической отправке новых коммитов**. Если ветка уже была загружена обычным `git push`, повторный запуск с `-o` ничего не отправит. Для обхода нужно либо создать пустой коммит (`git commit --allow-empty -m "Create MR"`), либо создать MR через веб-интерфейс.

**Пример использования:** `git push -o merge_request.create -o merge_request.target=develop -o merge_request.title="Feature implementation" origin feature`

---

## 3. Перенос репозитория git-merge и просмотр конфликтов в GitLab

Предварительно создаем пустой проект `git-merge` на GitLab без файла `README`.

```bash
git clone <URL_исходного_репозитория_git-merge>
cd git-merge
git branch -a
git remote set-url origin <URL_вашего_репозитория_на_GitLab>

git checkout master
git push -u origin master

git checkout development
git push -u origin development

git checkout feature
git push -o merge_request.create -o merge_request.target=development origin feature
```

**Разбор использованных команд и флагов:**
* `git branch -a` — выводит список всех локальных и удаленных (`remotes/origin/...`) веток репозитория.
* `git checkout development` — создает локальную отслеживаемую ветку на основе `remotes/origin/development` и переключается на нее.

**Разбор ошибок и нюансов:**
* **Несовпадение имени ветки (`develop` vs `development`):** В репозитории `git-merge` интеграционная ветка называется `development`, а не `develop`. Попытка выполнить `git checkout develop` приводит к ошибке `error: pathspec 'develop' did not match any file(s) known to git`.
* **Ошибка `error: src refspec feature does not match any`:** Возникает при попытке выполнить `git push origin feature` до того, как вы переключились на эту ветку локально (`git checkout feature`). Без локального переключения Git не создает локальную ссылку на удаленную ветку `origin/feature`.
* **Просмотр конфликтов в интерфейсе GitLab:** После перехода по ссылке созданного MR в центральной панели статуса отображается предупреждение `Merge blocked: merge conflicts must be resolved` и кнопка `Resolve conflicts`. При нажатии на нее открывается встроенный редактор с подсветкой конфликтующих блоков `<<<<<<< feature` и `>>>>>>> development`, а также кнопками выбора версии (`Use ours` / `Use theirs`).

**Пример использования:** `git diff development..feature` (позволяет заранее посмотреть разницу между ветками в терминале перед созданием Merge Request).

---

## 4. Fork репозитория git-checkout, коммит в ветку feature и создание Merge Request

В случае, когда исходный репозиторий закрыт и доступен только для чтения по учебному токену (кнопка Fork в браузере недоступна), выполняется локальная эмуляция форка в свой аккаунт GitLab (`git-checkout`).

```bash
git clone https://oauth2:<TOKEN>@://gitlab.com
cd git-checkout
git remote add my-fork git@gitlab.com:<ВАШ_ЛОГИН>/git-checkout.git

git checkout master
git push my-fork master

git checkout feature
echo "Первый компьютерный баг был настоящим мотыльком, найденным в реле компьютера Harvard Mark II в 1947 году." > interesting-info.txt
git add interesting-info.txt
git commit -m "Add interesting info file"

git push -o merge_request.create -o merge_request.target=master my-fork feature
```

**Разбор использованных команд и флагов:**
* `git remote add my-fork <URL>` — добавляет второй удаленный репозиторий под псевдонимом `my-fork`, сохраняя привязку к оригиналу под именем `origin`.
* `echo "..." > interesting-info.txt` — создает новый текстовый файл и записывает в него указанную строку.
* `git add interesting-info.txt` — добавляет новый неотслеживаемый файл в область подготовленных изменений (индекс / staging area).
* `git commit -m "..."` — фиксирует подготовленные изменения в локальной истории ветки `feature` отдельным коммитом с заданным сообщением.

**Разбор ошибок и нюансов:**
* **Ошибка `Access denied: This operation requires a fine-grained personal access token with the following project permissions: [Code: Push]`:** Возникает при попытке выполнить `git push origin feature` в исходный учебный репозиторий, так как выданный в задании токен имеет права только на чтение (`read_repository`).
* **Пропуск индексации (`git add`):** Если создать файл через `echo` и сразу сделать `git push`, новый файл не попадет на сервер, так как он находится в статусе `Untracked files` и не включен ни в один коммит.
* **Отсутствие целевой ветки в форке:** Перед отправкой `feature` с флагом создания MR в `my-fork` необходимо сначала отправить туда ветку `master` (`git push my-fork master`), иначе GitLab отклонит создание MR из-за отсутствия целевой ветки.

**Пример использования:** `git push my-fork --all` (отправляет сразу все локальные ветки в ваш репозиторий-форк за одну команду).
