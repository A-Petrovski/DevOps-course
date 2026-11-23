# Task 3.11 Images. Слои

## Оглавление
- [Введение и понятие слоя](#введение-и-понятие-слоя)
- [Образование слоёв в Dockerfile](#образование-слоёв-в-dockerfile)
- [Проверка слоёв и история образа (docker inspect и docker history)](#проверка-слоёв-и-история-образа-docker-inspect-и-docker-history)
- [Почему нельзя использовать ARG для хранения секретов](#почему-нельзя-использовать-arg-для-хранения-секретов)

---

## Введение и понятие слоя

Мы уже знаем как собирать образы, но это еще не все — мы изучили только первый этап. Нам еще необходимо понять, как работает сборка и сами образы изнутри.

**Слой** (он же Layer) в понятии Docker-образа — это изменение состояния операционной системы относительно ее прошлого состояния. 

В рамках самого первого состояния мы говорили о **UnionFS**:
* **UnionFS** — тип файловой системы, позволяющий накладывать состояние файловой системы слоями: каждый последующий слой использует предыдущий в режиме **Copy-on-write**, то есть хранит только изменения от предыдущего слоя.
* Основные поддерживаемые реализации: `aufs`, `btrfs`, `overlay2`.

Слой в понятии Docker связан со слоем UnionFS.

---

## Образование слоёв в Dockerfile

Слои в Dockerfile создает каждая **директива**. Каждая директива в ходе сборки создает новый слой поверх слоя в предыдущей директиве. При этом исходным слоем является последний слой образа, который берется за основу образа.

Представим, что у нас есть образ, в который добавлены два файла:
```text
/usr/local/bin/daemon
/etc/daemon/config.yml
```
и переменная окружения:
```text
DAEMON_VERSION=1.0.0
```

`Dockerfile` для этого образа будет выглядеть следующим образом:
```dockerfile
FROM alpine:latest
ENV DAEMON_VERSION 1.0.0
COPY daemon /usr/local/bin/daemon
COPY config.yml /etc/daemon/config.yml
```

Сборка такого образа будет выглядеть следующим образом:
```bash
$ docker build -t test:latest .
Sending build context to Docker daemon  3.072kB
Step 1/4 : FROM alpine:latest
 ---> 11cd0b38bc3c
Step 2/4 : ENV DAEMON_VERSION 1.0.0
 ---> Running in c3374710eccd
Removing intermediate container c3374710eccd
 ---> 8382abf6bc1c
Step 3/4 : COPY daemon /usr/local/bin/daemon
 ---> ac9fd5c52c26
Step 4/4 : COPY config.yml /etc/daemon/config.yml
 ---> 4ae8e33f705f
Successfully built 4ae8e33f705f
```

### Особенности процесса сборки:
* Каждая директива создает свой слой и выдает его идентификатор, который можно проверить через `docker inspect`.
* Если посмотреть последний и предпоследний слой у образов, список растет от слоя к слою. Директивы производятся в четкой последовательности и могут зависеть друг от друга (например, установка версии приложения, номер которой задан в предыдущей директиве через `ENV`).
* Сообщение `Removing intermediate container c3374710eccd` в директиве `ENV` — это стандартное поведение сборщика для экономии места: так он удаляет все промежуточные контейнеры, которые были созданы в рамках конкретного слоя. Образы можно сохранять, если потребуется, добавив к директиве `build` флаг `--rm=false`.

---

## Проверка слоёв и история образа (docker inspect и docker history)

При выполнении команды `docker pull` можно увидеть слои, которые загружает Docker — какие слои уже есть, а какие скачиваются. Пример вывода:
```bash
$ docker pull golang:1.10-alpine
1.10-alpine: Pulling from library/golang
6c40cc604d8e: Already exists
51d6d4c96fd3: Already exists
945d978b4f19: Already exists
74ebde557090: Pull complete
e6618267e614: Pull complete
Digest: sha256:549535d72baf28b3d1470ef9fcc6d4e0cb52c7fc3012655a659b65eaf942ec63
Status: Downloaded newer image for golang:1.10-alpine
docker.io/library/golang:1.10-alpine
```

Если с первым (локально собранным) образом получение информации через `docker inspect` сработает на каждом слое, то на втором (скачанном) — будет выдавать ошибку:
```bash
$ docker inspect 6c40cc604d8e
[]
Error: No such object: 6c40cc604d8e
```

Для анализа таких образов используется команда `docker history`:
* `docker history` — выводит историю образа со всеми командами, которые выполнены в рамках сборки образа, и все идентификаторы образов, созданные в ходе сборки образа.

Сравнение истории сборки нашего образа и загруженного образа:
```bash
$ docker history test:latest
IMAGE               CREATED             CREATED BY                                      SIZE                COMMENT
4ae8e33f705f        21 minutes ago      /bin/sh -c #(nop) COPY file:45b18160e6353231…   0B
ac9fd5c52c26        21 minutes ago      /bin/sh -c #(nop) COPY file:7c185a9a7817592d…   0B
8382abf6bc1c        21 minutes ago      /bin/sh -c #(nop)  ENV DAEMON_VERSION=1.0.0     0B
11cd0b38bc3c        16 months ago       /bin/sh -c #(nop)  CMD ["/bin/sh"]              0B
<missing>           16 months ago       /bin/sh -c #(nop) ADD file:25f61d70254b9807a…   4.41MB

$ docker history golang:1.10-alpine
IMAGE               CREATED             SIZE                COMMENT                                  CREATED BY
7b53e4a31d21        9 months ago        0B                                                           /bin/sh -c #(nop) WORKDIR /go
<missing>           9 months ago        0B                                                           /bin/sh -c mkdir -p "$GOPATH/src" "$GOPATH/b…
<missing>           9 months ago        0B                                                           /bin/sh -c #(nop)  ENV PATH=/go/bin:/usr/loc…
<missing>           9 months ago        0B                                                           /bin/sh -c #(nop)  ENV GOPATH=/go
<missing>           9 months ago        253MB                                                        /bin/sh -c set -eux;  apk add --no-cache --v…
<missing>           9 months ago        0B                                                           /bin/sh -c #(nop)  ENV GOLANG_VERSION=1.10.8
<missing>           9 months ago        17B                                                          /bin/sh -c [ ! -e /etc/nsswitch.conf ] && ec…
<missing>           9 months ago        551kB                                                        /bin/sh -c apk add --no-cache   ca-certifica…
<missing>           9 months ago        0B                                                           /bin/sh -c #(nop)  CMD ["/bin/sh"]
<missing>           9 months ago        5.53MB                                                       /bin/sh -c #(nop) ADD file:2a1fc9351afe35698…
```

У большинства слоев скачанного образа отсутствует `IMAGE ID` (`<missing>`). Это связано с тем, что образы, которые мы скачиваем, содержат лишь основные слои, необходимые для работы, а остальные слои, которые использовались для сборки, не нужны и поэтому не хранятся.

---

## Почему нельзя использовать ARG для хранения секретов

Представим, что у нас есть такой `Dockerfile`:
```dockerfile
FROM alpine:latest
ARG SECRET
RUN echo $SECRET > /opt/secret
```

Соберем этот образ:
```bash
$ docker build -t test:secret --build-arg SECRET=dontshowmeonwebinar .
Sending build context to Docker daemon  3.072kB
Step 1/3 : FROM alpine:latest
 ---> 965ea09ff2eb
Step 2/3 : ARG SECRET
 ---> Using cache
 ---> 9c3d613dac54
Step 3/3 : RUN echo $SECRET > /opt/secret
 ---> Running in ef25a9923e81
Removing intermediate container ef25a9923e81
 ---> c86141820a5d
Successfully built c86141820a5d
Successfully tagged test:secret
```

А теперь посмотрим его историю:
```bash
$ docker history c86141820a5d
IMAGE               CREATED              CREATED BY                                      SIZE                COMMENT
c86141820a5d        10 seconds ago       |1 SECRET=dontshowmeonwebinar /bin/sh -c ech…   20B
9c3d613dac54        About a minute ago   /bin/sh -c #(nop)  ARG SECRET                   0B
965ea09ff2eb        2 weeks ago          /bin/sh -c #(nop)  CMD ["/bin/sh"]              0B
<missing>           2 weeks ago          /bin/sh -c #(nop) ADD file:fe1f09249227e2da2…   5.55MB
```

Данные видны в выводе `docker history`, а значит, **скомпрометированы**.
