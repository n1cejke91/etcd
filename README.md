# Развертывание и проверка отказоустойчивости кластеров Etcd и Consul в WSL с использованием Docker Compose

Описание процесса развертывания отказоустойчивых кластеров Etcd и Consul с использованием Docker Compose в среде WSL, а также шаги по проверке их отказоустойчивости. Так как запускаем все на localhost, будем использовать разные порты для эмуляции нескольких нод.

## Требования

* Установленные Docker и Docker Compose в WSL.

## Развертывание кластера Etcd

1.  **Создание файла `docker-compose.yml`:**

    Создать файл `docker-compose.yml` со следующим содержимым:

    ```yml
    version: '3.8'
    services:
      etcd1:
        image: quay.io/coreos/etcd:v3.5.11
        container_name: etcd1
        ports:
          - "2379:2379"
          - "2380:2380"
        environment:
          - ETCD_NAME=etcd1
          - ETCD_LISTEN_CLIENT_URLS=[http://0.0.0.0:2379](http://0.0.0.0:2379)
          - ETCD_ADVERTISE_CLIENT_URLS=[http://127.0.0.1:2379](http://127.0.0.1:2379)
          - ETCD_LISTEN_PEER_URLS=[http://0.0.0.0:2380](http://0.0.0.0:2380)
          - ETCD_ADVERTISE_PEER_URLS=[http://127.0.0.1:2380](http://127.0.0.1:2380)
          - ETCD_INITIAL_CLUSTER=etcd1=[http://127.0.0.1:2380](http://127.0.0.1:2380),etcd2=[http://127.0.0.1:2381](http://127.0.0.1:2381),etcd3=[http://127.0.0.1:2382](http://127.0.0.1:2382)
          - ETCD_INITIAL_CLUSTER_STATE=new
        volumes:
          - etcd-data-1:/data
        networks:
          etcd-net:
            ipv4_address: 172.20.0.10

      etcd2:
        image: quay.io/coreos/etcd:v3.5.11
        container_name: etcd2
        ports:
          - "2389:2379"
          - "2390:2380"
        environment:
          - ETCD_NAME=etcd2
          - ETCD_LISTEN_CLIENT_URLS=[http://0.0.0.0:2379](http://0.0.0.0:2379)
          - ETCD_ADVERTISE_CLIENT_URLS=[http://127.0.0.1:2389](http://127.0.0.1:2389)
          - ETCD_LISTEN_PEER_URLS=[http://0.0.0.0:2380](http://0.0.0.0:2380)
          - ETCD_ADVERTISE_PEER_URLS=[http://127.0.0.1:2381](http://127.0.0.1:2381)
          - ETCD_INITIAL_CLUSTER=etcd1=[http://127.0.0.1:2380](http://127.0.0.1:2380),etcd2=[http://127.0.0.1:2381](http://127.0.0.1:2381),etcd3=[http://127.0.0.1:2382](http://127.0.0.1:2382)
          - ETCD_INITIAL_CLUSTER_STATE=new
        volumes:
          - etcd-data-2:/data
        networks:
          etcd-net:
            ipv4_address: 172.20.0.11
        depends_on:
          - etcd1

      etcd3:
        image: quay.io/coreos/etcd:v3.5.11
        container_name: etcd3
        ports:
          - "2409:2379"
          - "2410:2380"
        environment:
          - ETCD_NAME=etcd3
          - ETCD_LISTEN_CLIENT_URLS=[http://0.0.0.0:2379](http://0.0.0.0:2379)
          - ETCD_ADVERTISE_CLIENT_URLS=[http://127.0.0.1:2409](http://127.0.0.1:2409)
          - ETCD_LISTEN_PEER_URLS=[http://0.0.0.0:2380](http://0.0.0.0:2380)
          - ETCD_ADVERTISE_PEER_URLS=[http://127.0.0.1:2382](http://127.0.0.1:2382)
          - ETCD_INITIAL_CLUSTER=etcd1=[http://127.0.0.1:2380](http://127.0.0.1:2380),etcd2=[http://127.0.0.1:2381](http://127.0.0.1:2381),etcd3=[http://127.0.0.1:2382](http://127.0.0.1:2382)
          - ETCD_INITIAL_CLUSTER_STATE=new
        volumes:
          - etcd-data-3:/data
        networks:
          etcd-net:
            ipv4_address: 172.20.0.12
        depends_on:
          - etcd2

    volumes:
      etcd-data-1:
      etcd-data-2:
      etcd-data-3:

    networks:
      etcd-net:
        driver: bridge
        ipam:
          config:
            - subnet: 172.20.0.0/24
    ```

    *Используем разные порты (`2379`, `2389`, `2409` для клиентских запросов и `2380`, `2381`, `2382` для peer-коммуникаций) и IP-адреса в подсети Docker для эмуляции нескольких нод на localhost.*

2.  **Запуск кластера:**

    Перейти в директорию, где находится файл `docker-compose.yml`, и выполнить команду:

    ```bash
    docker-compose up -d
    ```

3.  **Проверка статуса кластера:**

    Выполнить следующую команду:

    ```bash
    docker exec -it etcd1 etcdctl endpoint health --endpoints=http://localhost:2379,http://localhost:2389,http://localhost:2409
    ```

    Ожидаемый вывод:

    ```
    http://localhost:2379: healthy
    http://localhost:2389: healthy
    http://localhost:2409: healthy
    ```

### Проверка отказоустойчивости Etcd

1.  **Запись значения в кластер:**

    ```bash
    docker exec -it etcd1 etcdctl put mykey myvalue
    ```

2.  **Чтение значения из кластера:**

    ```bash
    docker exec -it etcd1 etcdctl get mykey
    ```

    Убедиться, что получаем записанное значение (`myvalue`).

3.  **Остановка одной из нод Etcd:**

    ```bash
    docker stop etcd1
    ```

4.  **Проверка работоспособности кластера после остановки ноды:**

    Попробовать записать и прочитать новое значение, обращаясь к другой ноде через соответствующий порт:

    ```bash
    docker exec -it etcd2 etcdctl --endpoints=http://localhost:2389 put anotherkey anothervalue
    docker exec -it etcd2 etcdctl get anotherkey
    ```

    Должны успешно записать и прочитать значение.

5.  **Перезапуск остановленной ноды:**

    ```bash
    docker start etcd1
    ```

6.  **Проверка восстановления кластера:**

    Убедиться, что остановленная нода успешно присоединилась к кластеру:

    ```bash
    docker exec -it etcd1 etcdctl endpoint health --endpoints=http://localhost:2379,http://localhost:2389,http://localhost:2409
    ```

## Развертывание кластера Consul

1.  **Создание файла `docker-compose.yml`:**

    Создать файл `docker-compose.yml` в той же или новой директории WSL со следующим содержимым:

    ```yml
    version: '3.8'
    services:
      consul1:
        image: hashicorp/consul:1.17.1
        container_name: consul1
        ports:
          - "8500:8500"
          - "8300:8300"
          - "8301:8301/tcp"
          - "8301:8301/udp"
          - "8302:8302/tcp"
          - "8302:8302/udp"
          - "8600:8600/udp"
        command: agent -server -bootstrap-expect=3 -data-dir=/consul/data -node=consul-node-1 -advertise=172.21.0.10 -client=0.0.0.0 -retry-join=172.21.0.10 -retry-join=172.21.0.11 -retry-join=172.21.0.12
        volumes:
          - consul-data-1:/consul/data
        networks:
          consul-net:
            ipv4_address: 172.21.0.10

      consul2:
        image: hashicorp/consul:1.17.1
        container_name: consul2
        ports:
          - "8501:8500"
          - "8301:8300"
          - "8302:8301/tcp"
          - "8302:8301/udp"
          - "8303:8302/tcp"
          - "8303:8302/udp"
          - "8601:8600/udp"
        command: agent -server -data-dir=/consul/data -node=consul-node-2 -advertise=172.21.0.11 -client=0.0.0.0 -retry-join=172.21.0.10 -retry-join=172.21.0.11 -retry-join=172.21.0.12
        volumes:
          - consul-data-2:/consul/data
        networks:
          consul-net:
            ipv4_address: 172.21.0.11
        depends_on:
          - consul1

      consul3:
        image: hashicorp/consul:1.17.1
        container_name: consul3
        ports:
          - "8502:8500"
          - "8311:8300"
          - "8312:8301/tcp"
          - "8312:8301/udp"
          - "8313:8302/tcp"
          - "8313:8302/udp"
          - "8602:8600/udp"
        command: agent -server -data-dir=/consul/data -node=consul-node-3 -advertise=172.21.0.12 -client=0.0.0.0 -retry-join=172.21.0.10 -retry-join=172.21.0.11 -retry-join=172.21.0.12
        volumes:
          - consul-data-3:/consul/data
        networks:
          consul-net:
            ipv4_address: 172.21.0.12
        depends_on:
          - consul2

    volumes:
      consul-data-1:
      consul-data-2:
      consul-data-3:

    networks:
      consul-net:
        driver: bridge
        ipam:
          config:
            - subnet: 172.21.0.0/24
    ```

    *Также используем разные порты (`8500`, `8501`, `8502` для Web UI и другие для gRPC и связи между агентами) и IP-адреса в подсети Docker.*

2.  **Запуск кластера:**

    Перейти в директорию с файлом `docker-compose.yml` и выполнить:

    ```bash
    docker-compose up -d
    ```

3.  **Проверка статуса кластера:**

    Выполнить на одной из нод:

    ```bash
    docker exec -it consul1 consul members
    ```

    Ожидаемый вывод будет примерно таким:

    ```
    Node           Address           Status  Type    Build   Protocol  DC   Segment
    consul-node-1  172.21.0.10:8301  alive   server  1.17.1  2         default  <all>
    consul-node-2  172.21.0.11:8301  alive   server  1.17.1  2         default  <all>
    consul-node-3  172.21.0.12:8301  alive   server  1.17.1  2         default  <all>
    ```

    *Также можем открыть Web UI Consul, обратившись к `http://localhost:8500`, `http://localhost:8501` или `http://localhost:8502` в вашем браузере.*

### Проверка отказоустойчивости Consul

1.  **Регистрация сервиса:**

    ```bash
    docker exec -it consul1 consul services register -name=myservice -port=8080
    ```

2.  **Проверка регистрации сервиса:**

    ```bash
    docker exec -it consul1 consul services
    ```

3.  **Остановка одной из нод Consul:**

    ```bash
    docker stop consul1
    ```

4.  **Проверка состояния сервиса после остановки ноды:**

    Выполнить команду на другой ноде:

    ```bash
    docker exec -it consul2 consul services
    ```

    Сервис `myservice` должен по-прежнему отображаться.

5.  **Проверка информации о сервисе:**

    ```bash
    docker exec -it consul2 consul catalog service myservice
    ```

6.  **Перезапуск остановленной ноды:**

    ```bash
    docker start consul1
    ```

7.  **Проверка восстановления кластера:**

    ```bash
    docker exec -it consul1 consul members
    ```
