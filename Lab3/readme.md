Развертывание системы мониторинга ELK Stack (ElasticSearch)
================
### Шабанов Р.В.

Цель работы
==============

1.  Освоить базовые подходы централизованного сбора и накопления информации

2.  Освоить современные инструменты развертывания контейнирозованных приложений

3.  Закрепить знания о современных сетевых протоколах прикладного уровня

Ход выполнения практической работы
==============

## Шаг 1 - Увеличение размера виртуальной памяти системы

Для того, чтобы развернуть Elasticsearch потребуется увеличить виртуальную память
``` bash
sysctl -w vm.max_map_count=262144
```
## Шаг 2 - Создание .env, содержащую в себе пароли, версию, порты и т.п.
```
ELASTIC_PASSWORD=kibana
KIBANA_PASSWORD=kibana
STACK_VERSION=8.6.2
CLUSTER_NAME=docker-cluster
LICENSE=basic
ES_PORT=9200
KIBANA_PORT=5601
MEM_LIMIT=1073741824
```

## Шаг 3 - Создание docker-compose.yml
1. Для начала создадим сервис setup:
``` 
version: "3"

services:
  setup:
    image: elastic/elasticsearch:${STACK_VERSION}
    volumes:
      - certs:/usr/share/elasticsearch/config/certs
    user: "0"
    command: >
      bash -c '
        if [ x${ELASTIC_PASSWORD} == x ]; then
          echo "Set the ELASTIC_PASSWORD environment variable in the .env file";
          exit 1;
        elif [ x${KIBANA_PASSWORD} == x ]; then
          echo "Set the KIBANA_PASSWORD environment variable in the .env file";
          exit 1;
        fi;
        if [ ! -f config/certs/ca.zip ]; then
          echo "Creating CA";
          bin/elasticsearch-certutil ca --silent --pem -out config/certs/ca.zip;
          unzip config/certs/ca.zip -d config/certs;
        fi;
        if [ ! -f config/certs/certs.zip ]; then
          echo "Creating certs";
          echo -ne \
          "instances:\n"\
          "  - name: es01\n"\
          "    dns:\n"\
          "      - es01\n"\
          "      - localhost\n"\
          "    ip:\n"\
          "      - 127.0.0.1\n"\
          "  - name: filebeat\n"\
          "    dns:\n"\
          "      - es01\n"\
          "      - localhost\n"\
          "    ip:\n"\
          "      - 127.0.0.1\n"\
          "  - name: packetbeat\n"\
          "    dns:\n"\
          "      - es01\n"\
          "      - localhost\n"\
          "    ip:\n"\
          "      - 127.0.0.1\n"\
          > config/certs/instances.yml;
          bin/elasticsearch-certutil cert --silent --pem -out config/certs/certs.zip --in config/certs/instances.yml --ca-cert config/certs/ca/ca.crt --ca-key config/certs/ca/ca.key;
          unzip config/certs/certs.zip -d config/certs;
        fi;
        echo "Setting file permissions"
        chown -R root:root config/certs;
        find . -type d -exec chmod 750 \{\} \;;
        find . -type f -exec chmod 640 \{\} \;;
        echo "Waiting for Elasticsearch availability";
        until curl -s --cacert config/certs/ca/ca.crt https://es01:9200 | grep -q "missing authentication credentials"; do sleep 30; done;
        echo "Setting kibana_system password";
        until curl -s -X POST --cacert config/certs/ca/ca.crt -u "elastic:${ELASTIC_PASSWORD}" -H "Content-Type: application/json" https://es01:9200/_security/user/kibana_system/_password -d "{\"password\":\"${KIBANA_PASSWORD}\"}" | grep -q "^{}"; do sleep 10; done;
        echo "Successfull!";
      '
    healthcheck:
      test: ["CMD-SHELL", "[ -f config/certs/es01/es01.crt ]"]
      interval: 1s
      timeout: 5s
      retries: 120
```
2. Затем создадим сервис главной ноды ElasticSearch (es01):
```
es01:
    depends_on:
      setup:
        condition: service_healthy
    image: elastic/elasticsearch:${STACK_VERSION}
    volumes:
      - certs:/usr/share/elasticsearch/config/certs
      - esdata01:/usr/share/elasticsearch/data
    ports:
      - ${ES_PORT}:9200
    environment:
      - node.name=es01
      - cluster.name=${CLUSTER_NAME}
      - cluster.initial_master_nodes=es01
      - ELASTIC_PASSWORD=${ELASTIC_PASSWORD}
      - bootstrap.memory_lock=true
      - xpack.security.enabled=true
      - xpack.security.http.ssl.enabled=true
      - xpack.security.http.ssl.key=certs/es01/es01.key
      - xpack.security.http.ssl.certificate=certs/es01/es01.crt
      - xpack.security.http.ssl.certificate_authorities=certs/ca/ca.crt
      - xpack.security.transport.ssl.enabled=true
      - xpack.security.transport.ssl.key=certs/es01/es01.key
      - xpack.security.transport.ssl.certificate=certs/es01/es01.crt
      - xpack.security.transport.ssl.certificate_authorities=certs/ca/ca.crt
      - xpack.security.transport.ssl.verification_mode=certificate
      - xpack.license.self_generated.type=${LICENSE}
    mem_limit: ${MEM_LIMIT}
    ulimits:
      memlock:
        soft: -1
        hard: -1
    healthcheck:
      test:
        [
          "CMD-SHELL",
          "curl -s --cacert config/certs/ca/ca.crt https://localhost:9200 | grep -q 'missing authentication credentials'",
        ]
      interval: 10s
      timeout: 10s
      retries: 120
```
3. Далее добавим графическую панель управления Kibana:
```
 kibana:
    depends_on:
      es01:
        condition: service_healthy
    image: elastic/kibana:${STACK_VERSION}
    volumes:
      - certs:/usr/share/kibana/config/certs
      - kibanadata:/usr/share/kibana/data
    ports:
      - ${KIBANA_PORT}:5601
    environment:
      - SERVERNAME=kibana
      - ELASTICSEARCH_HOSTS=https://es01:9200
      - ELASTICSEARCH_USERNAME=kibana_system
      - ELASTICSEARCH_PASSWORD=${KIBANA_PASSWORD}
      - ELASTICSEARCH_SSL_CERTIFICATEAUTHORITIES=config/certs/ca/ca.crt
    mem_limit: ${MEM_LIMIT}
    healthcheck:
      test:
        [
          "CMD-SHELL",
          "curl -s -I http://localhost:5601 | grep -q 'HTTP/1.1 302 Found'",
        ]
      interval: 10s
      timeout: 10s
      retries: 120
```
4. Добавим используемые volume:
```
volumes:
  certs:
    driver: local
  esdata01:
    driver: local
  kibanadata:
    driver: local
```
## Шаг 4 - Добавление средства сбора информации из файлов журналов Filebeat

1. Дополним наши сервисы сервисом для запуска Filebeat
```
 filebeat:
    depends_on:
       es01:
        condition: service_healthy
    image: elastic/filebeat:${STACK_VERSION}
    container_name: filebeat
    command: filebeat -e -strict.perms=false
    volumes:
      - ./filebeat.yml:/usr/share/filebeat/filebeat.yml
      - ./logs/:/var/log/
      - certs:/usr/share/elasticsearch/config/certs
    environment:
      - ELASTICSEARCH_HOSTS=https://es01:9200
      - ELASTICSEARCH_USERNAME=elastic
      - ELASTICSEARCH_PASSWORD=${ELASTIC_PASSWORD}
      - ELASTICSEARCH_SSL_CERTIFICATEAUTHORITIES=config/certs/ca/ca.crt
```
filebeat.yml - файл конфигурации Filebeat

logs - каталог с лог-файлами

2. Создадим файл конфигурации filebeat.yml:
```
filebeat.inputs:
- type: filestream
  id: sys-logs
  enabled: true
  paths:
    - /var/log/*

output.elasticsearch:
  hosts: '${ELASTICSEARCH_HOSTS:elasticsearch:9200}'
  username: '${ELASTICSEARCH_USERNAME:}'
  password: '${ELASTICSEARCH_PASSWORD:}'
  ssl:
    certificate_authorities: "/usr/share/elasticsearch/config/certs/ca/ca.crt"
    certificate: "/usr/share/elasticsearch/config/certs/filebeat/filebeat.crt"
    key: "/usr/share/elasticsearch/config/certs/filebeat/filebeat.key"
```
## Шаг 5 - Добавление средства для сбора сетевого трафика Packetbeat

1. Добавим сервис для запуска Packetbeat:
```
 packetbeat:
    depends_on:
      es01:
        condition: service_healthy
    image: elastic/packetbeat:${STACK_VERSION}
    container_name: packetbeat
    user: root
    cap_add: 
      - NET_RAW
      - NET_ADMIN
    command: packetbeat -e -strict.perms=false
    volumes:
      - ./packetbeat.yml:/usr/share/packetbeat/packetbeat.yml
      - certs:/usr/share/elasticsearch/config/certs
      - /run/docker.sock:/run/docker.sock
    environment:
      - ELASTICSEARCH_HOSTS=https://es01:9200
      - ELASTICSEARCH_USERNAME=elastic
      - ELASTICSEARCH_PASSWORD=${ELASTIC_PASSWORD}
      - ELASTICSEARCH_SSL_CERTIFICATEAUTHORITIES=config/certs/ca/ca.crt
  
```
2. Создадим файл конфигурации packetbeat.yml, который будет отвечать за фильтрацию пакетов:
```
packetbeat.interfaces.device: any

packetbeat.flows:
  timeout: 30s
  period: 10s

packetbeat.protocols.dns:
  ports: [53]
  include_authorities: true
  include_additionals: true

packetbeat.protocols.http:
  ports: [80, 5601, 9200, 8080, 8081, 5000, 8002]

packetbeat.protocols.memcache:
  ports: [11211]

packetbeat.protocols.mysql:
  ports: [3306]

packetbeat.protocols.pgsql:
  ports: [5432]

packetbeat.protocols.redis:
  ports: [6379]

packetbeat.protocols.thrift:
  ports: [9090]

packetbeat.protocols.mongodb:
  ports: [27017]

packetbeat.protocols.cassandra:
  ports: [9042]

processors:
- add_cloud_metadata: ~

output.elasticsearch:
  hosts: '${ELASTICSEARCH_HOSTS:elasticsearch:9200}'
  username: '${ELASTICSEARCH_USERNAME:}'
  password: '${ELASTICSEARCH_PASSWORD:}'
  ssl:
    certificate_authorities: "/usr/share/elasticsearch/config/certs/ca/ca.crt"
    certificate: "/usr/share/elasticsearch/config/certs/packetbeat/packetbeat.crt"
    key: "/usr/share/elasticsearch/config/certs/packetbeat/packetbeat.key"
```
3. Запустим тестовый сервис веб-сервера Nginx:
```
 nginx:
    container_name: nginx
    image: nginx:stable-alpine3.17
    ports: ['80:80']
    expose:
      - '80'
    command: nginx -g 'daemon off;'
    volumes:
      - ./logs/nginx/:/var/log/nginx/
```
## Шаг 6 - Запуск сервисов
Для запуска всех созданных нами сервисов, пропишем следующую команду:
```
docker-compose up
```
## Шаг 7 - Работа с ElasticSearch

1. Перейдем на сам сайт ```localhost:5601```

После чего авторизуемся:
<image src = "images\a.jpg">

2. Проверим работоспособность Filebeat и Packetbeat

Это можно сделать с помощью запроса ```GET _cat/indices```:
<image src = "images\b.jpg">

3. Создадим Data view для лог-файлов и получим данные:

<image src = "images\c.jpg">

4. Создадим Data view для трафика и получим статистику использования сетевого трафика:

<image src = "images\d.jpg">

Оценка результата
=================

Была развёрнута поисковая система ElasticSearch и произведена настройка системы сбора трафика и лог-файлов.

Вывод
=================
В результате проделанной практической работы были освоены приложения для работы с контейнеризации приложений  - Docker, Docker-compose и распределенная поисковая система ElasticSearch