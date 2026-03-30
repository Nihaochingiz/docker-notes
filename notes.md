# Docker: Контейнеры, Образы и Сети

## Что такое контейнер

Контейнер - это процесс в песочнице, запущенный на хост-машине, который изолирован от всех других процессов, запущенных на этой хост-машине. Для такой изоляции используются пространства имен ядра и cgroups - функции, которые уже давно существуют в Linux. Docker делает эти возможности доступными и простыми в использовании.

**Итог:** Контейнер - это использующийся экземпляр образа (image).

### Возможности работы с контейнерами через Docker

- Создавать
- Запускать
- Останавливать
- Переносить
- Удалять

### Характеристики контейнеров

- Могут запускаться в облаке, на локальной машине и на хост-машине
- Обладают изолированностью от других контейнеров
- Имеют собственное программное обеспечение, двоичные файлы и конфигурации

---

## Docker Image

Образ (Image) включает все, что вам нужно для запуска вашего приложения:
- Скомпилированный бинарник приложения
- Runtime
- Библиотеки
- Другие ресурсы, необходимые вашему приложению

---

## Dockerfile

### Пример Dockerfile для Node.js приложения

```dockerfile
# syntax=docker/dockerfile:1

FROM node:18-alpine
WORKDIR /app
COPY . .
RUN yarn install --production
CMD ["node", "src/index.js"]
EXPOSE 3000
```

### Пример Dockerfile для Go приложения с многоэтапной сборкой

```dockerfile
# syntax=docker/dockerfile:1

##
## Build the application from source
##

FROM golang:1.19 AS build-stage

WORKDIR /app

COPY go.mod go.sum ./
RUN go mod download

COPY *.go ./

RUN CGO_ENABLED=0 GOOS=linux go build -o /docker-gs-ping

##
## Run the tests in the container
##

FROM build-stage AS run-test-stage
RUN go test -v ./ .

##
## Deploy the application binary into a lean image
##

FROM gcr.io/distroless/base-debian11 AS build-release-stage

WORKDIR /

COPY --from=build-stage /docker-gs-ping /docker-gs-ping

EXPOSE 8080

USER nonroot:nonroot

ENTRYPOINT ["/docker-gs-ping"]
```

### Инструкции Dockerfile

| Инструкция | Описание |
|------------|----------|
| `FROM golang:1.19` | Загружаем в image golang с версией 1.19 |
| `WORKDIR /app` | Создаем директорию внутри image и указываем ее как пункт назначения по умолчанию для всех последующих команд |
| `COPY go.mod go.sum ./` | Копируем файлы из текущей директории в `/app` внутри image |
| `RUN go mod download` | Загружаем модули в image |
| `COPY *.go ./` | Копируем все файлы с расширением `.go` из текущей директории |
| `RUN CGO_ENABLED=0 GOOS=linux go build -o /docker-gs-ping` | Компилируем приложение. Результатом будет бинарный файл `docker-gs-ping` в корне файловой системы image |
| `CMD ["/docker-gs-ping"]` | Указываем докеру, какую команду выполнять при запуске контейнера |
| `EXPOSE 8080` | Указывает порт, который контейнер будет прослушивать во время выполнения |

---

## Работа с образами (Images)

### Сборка образа

```bash
docker build -t getting-started .
docker build --tag docker-gs-ping .
```

Точка в конце говорит о том, что нужно искать Dockerfile в текущем каталоге. Флаг `--tag` (или `-t`) задает имя вашему образу.

### Управление тегами

```bash
# Добавить новый тег к существующему образу
docker image tag docker-gs-ping:latest docker-gs-ping:v1.0

# Показать все Docker образы
docker image ls

# Удалить тег образа (не сам образ)
docker image rm docker-gs-ping:v1.0
```

---

## Работа с контейнерами

### Запуск контейнера

```bash
# Базовый запуск
docker run docker-gs-ping

# Запуск с пробросом портов
docker run -t -i --publish 8080:8080 docker-gs-ping
docker run -dp 127.0.0.1:3000:3000 getting-started

# Запуск в фоновом режиме
docker run -t -i -d -p 8080:8080 docker-gs-ping

# Запуск с именем контейнера
docker run -d -p 8080:8080 --name rest-server docker-gs-ping
```

### Управление контейнерами

```bash
# Показать работающие контейнеры
docker ps

# Показать все контейнеры (включая остановленные)
docker ps -all
docker ps -a

# Остановить контейнер
docker stop <container_name>

# Перезапустить контейнер
docker restart <container_name>

# Удалить контейнер
docker rm <container_name>

# Удалить несколько контейнеров
docker rm container1 container2 container3

# Удалить все остановленные контейнеры
docker container prune
```

---

## Docker Network

Сетевое взаимодействие контейнеров - это возможность контейнеров подключаться и взаимодействовать друг с другом.

### Особенности сетевой работы по умолчанию

- По умолчанию у контейнеров включена сетевая связь, и они могут устанавливать исходящие соединения
- Контейнер не имеет информации о том, к какой сети он привязан
- Контейнер видит только сетевой интерфейс с IP-адресом, шлюз, таблицу маршрутизации, службы DNS и другие сетевые данные (если контейнер не использует сетевой драйвер none)

### Пользовательские сети (User-defined networks)

Вы можете создавать пользовательские сети и соединять несколько контейнеров к одной сети. После подключения к пользовательской сети контейнеры могут взаимодействовать друг с другом, используя IP-адреса контейнеров или их имена.

```bash
# Создание сети с использованием bridge network driver
docker network create -d bridge my-net

# Запуск контейнера в созданной сети
docker run --network=my-net -itd --name=container3 busybox

# Показать все сети
docker network list
```

### Container networks

В дополнение вы можете привязать один контейнер к сети другого контейнера с помощью команды:

```bash
--network container:<name|id>
```

**Пример:** Запускается контейнер Redis с привязкой к localhost, затем выполняется команда redis-cli и устанавливается соединение с сервером Redis через интерфейс localhost.

```bash
docker run -d --name redis example/redis --bind 127.0.0.1
docker run --rm -it --network container:redis example/redis-cli -h 127.0.0.1
```

---

## Проброс портов (Published Ports)

По умолчанию, когда вы создаете или запускаете контейнер с помощью `docker create` или `docker run`, контейнер не раскрывает свои порты внешнему миру. Чтобы сделать порт доступным для служб за пределами Docker, используйте флаг `--publish` или `-p`. При этом на хосте создается правило брандмауэра, сопоставляющее порт контейнера с портом на хосте Docker для внешнего мира.

| Значение флага | Описание |
|----------------|----------|
| `-p 8080:80` | Map port 8080 on the Docker host to TCP port 80 in the container |
| `-p 192.168.1.100:8080:80` | Map port 8080 on the Docker host IP 192.168.1.100 to TCP port 80 in the container |
| `-p 8080:80/tcp -p 8080:80/udp` | Map TCP port 8080 on the Docker host to TCP port 80 in the container, and map UDP port 8080 on the Docker host to UDP port 80 in the container |

---

## Работа с базами данных

### Тома (Volumes)

Тома используются для сохранения данных вне контейнера.

```bash
# Создать том
docker volume create roach

# Показать все тома
docker volume list
```

### Запуск CockroachDB с томом и сетью

```bash
# Создать сеть
docker network create -d bridge mynet

# Запуск CockroachDB
docker run -d \
  --name roach \
  --hostname db \
  --network mynet \
  -p 26257:26257 \
  -p 8080:8080 \
  -v roach:/cockroach/cockroach-data \
  cockroachdb/cockroach:latest-v20.1 start-single-node \
  --insecure

# Запуск SQL оболочки
docker exec -it roach ./cockroach sql --insecure
```

---

## Разработка с Live Reload

### Dockerfile для разработки

```dockerfile
FROM golang:1.16 as base

FROM base as dev
RUN curl -sSfL https://raw.githubusercontent.com/cosmtrek/air/master/install.sh | sh -s -- -b $(go env GOPATH)/bin

WORKDIR /opt/app/api
CMD ["air"]
```

### docker-compose.yml

```yaml
version: '3.9'
services:
  app:
    build:
      dockerfile: Dockerfile
      context: .
      target: dev
    volumes:
      - .:/opt/app/api
```

---

## Docker Init

Начиная с версии Docker 4.26+ доступна команда:

```bash
docker init
```

Создает шаблон (template) для Docker конфигурации.
