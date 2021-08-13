# Сеть в docker
По-умолчанию запущенные контейнеры не видят друг-друга
```shell
docker run -dit --name db1 alpine
docker run -dit --name db2 alpine
docker exec db2 ping db1
# ping: bad address 'db1'
docker exec db2 ping db2
# ping: bad address 'db2'
```
Для того, чтобы контейнеры смогли взаимодействовать есть docker network
```shell
docker network create db-net
docker run -dit --name db3 --network db-net alpine
docker exec db3 ping db3
# PING db3 (172.22.0.2): 56 data bytes
```
Можно добавить сеть к уже запущенным контейнерам
```shell
docker network connect db-net db1
docker exec db3 ping db1
#PING db1 (172.22.0.3): 56 data bytes
```

# Docker compose
Управляет сетью между контейнерами (и не только) за вас и очень удобен, когда нужно поднять несколько связанных между собой приложений

## Формат файла docker-compose
Общая структура
```yaml
version: #версия_спеки

volumes: #описание томов
  volume_1:  
networks: #описание сетей
  network-1:
services: #описание того, что запускать
  db:
  kafka:
```
1. docker compose умеет собирать образы
```yaml
version: "3.9"
services:
  webapp:
    build: ./dir
```
2. использовать готовый имейдж
```yaml

services:
  db:
    image: 'postgres:12.5'
    ports:
      - '5432:5432'
    environment:
      - POSTGRES_PASSWORD=password
      - POSTGRES_DB=finmon
```

# Запускаем Kafka в compose

Начинаем с минимального файла:
```yaml
version: "3.9"
services:
  kafka:
    image: docker-proxy.tcsbank.ru/bitnami/kafka:latest
```

При запуске такой конфигурации выдается ошибка
```shell
kafka_1  | kafka 07:20:56.97 ERROR ==> The KAFKA_CFG_LISTENERS environment variable does not configure a secure listener. Set the environment variable ALLOW_PLAINTEXT_LISTENER=yes to allow the container to be started with a plaintext listener. This is only recommended for development.
kafka_1  | kafka 07:20:56.97 ERROR ==> The KAFKA_ZOOKEEPER_PROTOCOL environment variable does not configure a secure protocol. Set the environment variable ALLOW_PLAINTEXT_LISTENER=yes to allow the container to be started with a plaintext listener. This is only recommended for development.
```
Для наших тестовых целей добавляем эту настройку

```yaml
version: "3.9"
services:
  kafka:
    image: docker-proxy.tcsbank.ru/bitnami/kafka:latest
    environment:
      ALLOW_PLAINTEXT_LISTENER: "yes"
```

После запуска получаем ошибку подключения к zookeeper, что логично
```shell
kafka_1  | [2021-08-13 07:22:23,597] INFO Opening socket connection to server localhost/127.0.0.1:2181. Will not attempt to authenticate using SASL (unknown error) (org.apache.zookeeper.ClientCnxn)
kafka_1  | [2021-08-13 07:22:23,598] INFO Socket error occurred: localhost/127.0.0.1:2181: Connection refused (org.apache.zookeeper.ClientCnxn)
```

Поэтому определяем еще один сервис - zookeeper
```yaml
version: "3.9"
services:
  kafka:
    image: docker-proxy.tcsbank.ru/bitnami/kafka:latest
    environment:
      ALLOW_PLAINTEXT_LISTENER: "yes"
  zookeeper:
    image: docker-proxy.tcsbank.ru/bitnami/zookeeper:latest
```

Ошибка сохраняется 
```shell
kafka_1      | [2021-08-13 07:23:43,148] INFO Opening socket connection to server localhost/127.0.0.1:2181. Will not attempt to authenticate using SASL (unknown error) (org.apache.zookeeper.ClientCnxn)
kafka_1      | [2021-08-13 07:23:43,149] INFO Socket error occurred: localhost/127.0.0.1:2181: Connection refused (org.apache.zookeeper.ClientCnxn)
```

Ошибка происходит из-за того, что каждый из процессов запущен в своем контейнере и не видят друг-друга. Необходимо
сконфигурить kafka указав где искать zookeeper

```yaml
version: "3.9"
services:
  kafka:
    image: docker-proxy.tcsbank.ru/bitnami/kafka:latest
    environment:
      ALLOW_PLAINTEXT_LISTENER: "yes"
      KAFKA_CFG_ZOOKEEPER_CONNECT: "zookeeper:2181"
#
  zookeeper:
    image: docker-proxy.tcsbank.ru/bitnami/zookeeper:latest
```

```yaml
version: "3.9"
services:
  kafka:
    image: docker-proxy.tcsbank.ru/bitnami/kafka:latest
    environment:
      ALLOW_PLAINTEXT_LISTENER: "yes"
      KAFKA_CFG_ZOOKEEPER_CONNECT: "zookeeper:2181"
  #
  zookeeper:
    image: docker-proxy.tcsbank.ru/bitnami/zookeeper:latest
    environment:
      ALLOW_ANONYMOUS_LOGIN: "yes"
```

При попытке подключения к kafka произойдет ошибка. Проблема в том, что мы не опубликовали нужные порты
```yaml

version: "3.9"
services:
  kafka:
    image: docker-proxy.tcsbank.ru/bitnami/kafka:latest
    ports:
    - "9092:9092"
    environment:
      ALLOW_PLAINTEXT_LISTENER: "yes"
      KAFKA_CFG_ZOOKEEPER_CONNECT: "zookeeper:2181"

  #
  zookeeper:
    image: docker-proxy.tcsbank.ru/bitnami/zookeeper:latest
    ports:
    - "2181:2181"
    environment:
      ALLOW_ANONYMOUS_LOGIN: "yes"

```

После открытия портов подключение проходит, но вот работать с топиками не удастся. Дело в том, что
kafka публикует в zookeeper адрес и порт брокера - т.е. хост и порт на котором
она запущена. В нашем примере это kafka:9091. 
Клиент получает адреса брокера из zookeeper и пытается подключиться к kafka:9091 и не может зарезолвить такой хост.
Решить можно через network: host или специальной настройкой kafka

```yaml
version: "3.9"
services:
  kafka:
    image: docker-proxy.tcsbank.ru/bitnami/kafka:latest
    ports:
    - "9092:9092"
    environment:
      ALLOW_PLAINTEXT_LISTENER: "yes"
      KAFKA_CFG_ZOOKEEPER_CONNECT: "zookeeper:2181"
      KAFKA_CFG_ADVERTISED_LISTENERS: "PLAINTEXT://localhost:9092"
      KAFKA_CFG_LISTENERS: "PLAINTEXT://:9092"

  #
  zookeeper:
    image: docker-proxy.tcsbank.ru/bitnami/zookeeper:latest
    ports:
    - "2181:2181"
    environment:
      ALLOW_ANONYMOUS_LOGIN: "yes"
```


Ну и окончательный вариант со сборкой контейнера
```yaml
version: "3.9"
services:
  kafka:
    image: docker-proxy.tcsbank.ru/bitnami/kafka:latest
    ports:
    - "9092:9092"
    environment:
      ALLOW_PLAINTEXT_LISTENER: "yes"
      KAFKA_CFG_ZOOKEEPER_CONNECT: "zookeeper:2181"
      KAFKA_CFG_ADVERTISED_LISTENERS: "PLAINTEXT://localhost:9092"
      KAFKA_CFG_LISTENERS: "PLAINTEXT://:9092"

  #
  zookeeper:
    image: docker-proxy.tcsbank.ru/bitnami/zookeeper:latest
    ports:
    - "2181:2181"
    environment:
      ALLOW_ANONYMOUS_LOGIN: "yes"

  my-app:
    build:
      context: ./
```