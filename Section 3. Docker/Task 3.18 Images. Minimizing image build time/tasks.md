# Практика: Минимизация времени и объема сборки Docker-образа (counter)

## Оглавление
- [1. Клонирование репозитория counter](#1-клонирование-репозитория-counter)
- [2. Написание Dockerfile по универсальной структуре](#2-написание-dockerfile-по-универсальной-структуре)
- [3. Сборка образа normal с замером времени](#3-сборка-образа-normal-с-замером-времени)
- [4. Изменение исходного кода и пересборка normal с кэшем](#4-изменение-исходного-кода-и-пересборка-normal-с-кэшем)
- [5. Создание оптимизированного Dockerfile.mini](#5-создание-оптимизированного-dockerfilemini)
- [6. Включение experimental features и сборка образа squash](#6-включение-experimental-features-и-сборка-образа-squash)
- [7. Изменение кода и пересборка образа squash](#7-изменение-кода-и-пересборка-образа-squash)
- [8. Сравнение размеров получившихся образов](#8-сравнение-размеров-получившихся-образов)

---

## 1. Клонирование репозитория counter

Клонируем учебный репозиторий и переходим в рабочую директорию проекта (внутри находятся файлы `Pipfile`, `Pipfile.lock`, `server.py`, `settings.py`, `storage.py`).

```bash
git clone <url_репозитория_counter>
cd counter
```

**Разбор параметров:**
* `git clone <url>` — создает локальную копию удаленного Git-репозитория.
* `cd counter` — переход в директорию проекта, которая станет контекстом сборки (build context) для Docker.

**Пример использования:**
```bash
git clone https://github.com/example/counter.git && cd counter
```

---

## 2. Написание Dockerfile по универсальной структуре

Создаем `Dockerfile` на базе `python:3.7-buster`, соблюдая правильный порядок слоев: сначала зависимости, затем исходный код.

```dockerfile
FROM python:3.7-buster
WORKDIR /app

COPY Pipfile* ./
RUN pip install pipenv && pipenv install --system --deploy

COPY *.py ./
CMD ["python", "server.py"]
```

**Разбор инструкций и флагов:**
* `FROM python:3.7-buster` — задает базовый образ с предустановленным Python 3.7 на базе Debian Buster.
* `WORKDIR /app` — создает и назначает рабочую директорию внутри контейнера для всех последующих инструкций.
* `COPY Pipfile* ./` — копирует только файлы описания зависимостей (`Pipfile` и `Pipfile.lock`) на раннем этапе для сохранения кэша.
* `RUN pip install pipenv && pipenv install --system --deploy` — устанавливает `pipenv` и разворачивает зависимости напрямую в системное окружение контейнера (`--system`), строго сверяясь с `Pipfile.lock` (`--deploy`).
* `COPY *.py ./` — копирует исходный код приложения (`server.py`, `settings.py`, `storage.py`) в самом конце, чтобы правки в коде не сбрасывали кэш зависимостей.
* `CMD ["python", "server.py"]` — определяет команду запуска главного исполняемого файла приложения при старте контейнера.

**Разбор нюансов:**
* В проекте главным исполняемым файлом является `server.py` (а не `app.py`).
* Использование маски `COPY *.py ./` вместо `COPY . .` предотвращает копирование в образ служебной директории `.git`, самого `Dockerfile` и временных файлов.

**Пример использования:**
```bash
cat << 'EOF' > Dockerfile
FROM python:3.7-buster
WORKDIR /app
COPY Pipfile* ./
RUN pip install pipenv && pipenv install --system --deploy
COPY *.py ./
CMD ["python", "server.py"]
EOF
```

---

## 3. Сборка образа normal с замером времени

Выполняем первичную («холодную») сборку образа `counter:normal` с помощью системной утилиты `time`.

```bash
time docker build -t counter:normal .
```

**Разбор флагов и вывода:**
* `time` — системная утилита Linux (`/usr/bin/time` или встроенная в shell), измеряющая длительность выполнения команды.
* `docker build` — запускает сборку образа из `Dockerfile`.
* `-t counter:normal` — присваивает собираемому образу имя `counter` и тег `normal`.
* `.` — указывает текущую директорию в качестве контекста сборки.
* **Нюанс вывода `/usr/bin/time`:** В выводе вида `0.53user 4.04system 1:02.84elapsed 7%CPU` главное значение — `1:02.84elapsed` (реальное время сборки по настенным часам: 1 минута 2.84 секунды). Низкий процент CPU (`7%CPU`) означает, что клиент `docker` почти не нагружал процессор, а основное время ушло на скачивание базового образа и установку пакетов демоном `dockerd` по сети.

**Пример использования:**
```bash
/usr/bin/time -p docker build -t counter:normal .
```

---

## 4. Изменение исходного кода и пересборка normal с кэшем

Вносим некритичное изменение (комментарий) в любой `.py` файл (например, `storage.py` или `server.py`) и запускаем повторную сборку.

```bash
echo "# test build cache" >> storage.py
time docker build -t counter:normal .
```

**Разбор параметров:**
* `echo "# test build cache" >> storage.py` — дописывает комментарий в конец файла `storage.py`, меняя его контрольную сумму (хеш), но не нарушая логику работы Python.
* Повторный запуск `docker build` использует закэшированные слои (`CACHED`) для скачивания образа и установки библиотек через `pipenv`, пересобирая только слой `COPY *.py ./`.

**Пример использования:**
```bash
echo "# cache test $(date)" >> server.py && time docker build -t counter:normal .
```

---

## 5. Создание оптимизированного Dockerfile.mini

Создаем копию `Dockerfile.mini`, в которой установка зависимостей и очистка образа от `pipenv`, `pip`, кэша и `Pipfile*` происходят внутри **одной** инструкции `RUN`.

```dockerfile
FROM python:3.7-buster
WORKDIR /app

COPY Pipfile* ./

RUN pip install --no-cache-dir pipenv && \
    pipenv install --system --deploy && \
    pip uninstall -y pipenv pip && \
    rm -rf Pipfile* /root/.cache

COPY *.py ./
CMD ["python", "server.py"]
```

**Разбор инструкций и флагов:**
* `--no-cache-dir` — запрещает `pip` сохранять скачанные `.whl` архивы пакетов в локальный кэш.
* `&&` — связывает команды в единую цепочку внутри одного `RUN`, чтобы все созданные и удаленные файлы обработались в пределах одного слоя файловой системы.
* `pip uninstall -y pipenv pip` — удаляет пакетные менеджеры `pipenv` и `pip` без интерактивного подтверждения (`-y`) после того, как все нужные библиотеки уже установлены в систему.
* `rm -rf Pipfile* /root/.cache` — рекурсивно и принудительно (`-rf`) удаляет файлы манифестов зависимостей и остатки кэша из домашней директории `root`.

**Разбор нюансов:**
* Если вынести `rm -rf Pipfile*` в отдельную инструкцию `RUN` на следующей строке, объем образа **не уменьшится**: в файловой системе OverlayFS удаление файла в новом слое создает лишь метку скрытия (whiteout), но физически данные остаются в предыдущем слое.

**Пример использования:**
```bash
cp Dockerfile Dockerfile.mini && nano Dockerfile.mini
```

---

## 6. Включение experimental features и сборка образа squash

Активируем экспериментальные функции в конфигурации демона Docker для поддержки флага `--squash`, перезапускаем службу и собираем образ `counter:squash`.

```bash
echo '{"experimental": true}' | sudo tee /etc/docker/daemon.json
sudo systemctl restart docker
DOCKER_BUILDKIT=0 time docker build -f Dockerfile.mini --squash -t counter:squash .
```

**Разбор флагов и нюансов:**
* `{"experimental": true}` в `/etc/docker/daemon.json` — включает экспериментальный функционал в демоне `dockerd`.
* `sudo systemctl restart docker` — перезапускает службу Docker для применения новой конфигурации.
* `-f Dockerfile.mini` — явно указывает путь к файлу сборки (по умолчанию ищется `Dockerfile`).
* `--squash` — объединяет все вновь созданные слои образа в один единый слой поверх базового образа, удаляя историю промежуточных слоев.
* `DOCKER_BUILDKIT=0` — **важный нюанс:** в современных версиях Docker по умолчанию активен движок сборки **BuildKit** (в логах видны строки `exporting attestation manifest`), который не поддерживает классический флаг `--squash` и выдает ошибку. Переменная `DOCKER_BUILDKIT=0` временно переключает сборку на классический движок Docker Engine.

**Пример использования:**
```bash
DOCKER_BUILDKIT=0 time docker build -f Dockerfile.mini --squash -t counter:squash .
```

---

## 7. Изменение кода и пересборка образа squash

Снова вносим комментарий в исходный код и замеряем время пересборки образа с флагом `--squash`.

```bash
echo "# test squash cache" >> storage.py
DOCKER_BUILDKIT=0 time docker build -f Dockerfile.mini --squash -t counter:squash .
```

**Разбор параметров:**
* При пересборке Docker использует кэш для тяжелого слоя `RUN` с установкой зависимостей, но из-за флага `--squash` тратит дополнительное время на финальный экспорт, склеивание и перепаковку всех слоев в один монолитный слой.

**Пример использования:**
```bash
echo "# squash rebuild" >> server.py && DOCKER_BUILDKIT=0 time docker build -f Dockerfile.mini --squash -t counter:squash .
```

---

## 8. Сравнение размеров получившихся образов

Выводим список локальных образов репозитория `counter` и сравниваем колонку `SIZE`.

```bash
docker images counter
```

**Разбор параметров:**
* `docker images <имя>` — выводит таблицу локальных образов, отфильтрованную по имени репозитория `counter`, с отображением тегов (`normal` и `squash`), ID образов, времени создания и итогового размера.

**Пример использования:**
```bash
docker images --format "table {{.Repository}}\t{{.Tag}}\t{{.Size}}" counter
```
