# [Separate storage and compute](https://docs.starrocks.io/docs/quick_start/shared-data/)

Hot _**data is cached locally**_ and When the cache is hit, the query performance is comparable to that of storage-compute coupled architecture.  
Compute nodes _**(CN) can be added**_ or removed on demand _**within seconds**_.  
This architecture _**reduces storage cost**_, ensures _**better resource isolation**_, and provides _**elasticity and scalability**_.

## Prerequisites
4 GB RAM assigned to Docker
```
docker stats
```
Обратите внимание на столбцы MEM USAGE и LIMIT.

### Глобальное расположение данных Docker
```
sudo docker info | grep "Docker Root Dir"
 Docker Root Dir: /var/lib/docker
```
### Найти все смонтированные тома
```
sudo docker inspect -f '{{ range .Mounts }}{{ .Source }} -> {{ .Destination }}{{ "\n" }}{{ end }}' quickstart

```
### Увидеть рабочую файловую систему
```
sudo docker inspect -f '{{ .GraphDriver.Data.MergedDir }}' quickstart
/var/lib/docker/overlay2/61b9118a06f02d00703837142aad9b964f3f006441e9642778b3566e19dd99b7/merged
```

Итого делаем вывод, что база у нас в докере, а докер на руте.   
Т.е. загрузка больших данных у нас просадит рутовый диск, но место на нем у нас еще есть:
```
df -h /dev/mapper/almalinux_igonin--vl-root 
Файловая система                      Размер Использовано  Дост Использовано% Cмонтировано в
/dev/mapper/almalinux_igonin--vl-root    70G          32G   39G           46% /
```
## [Установка](https://docs.starrocks.io/docs/quick_start/shared-data/#deploy-starrocks-and-minio)
В процессе установки у нас проблема
```
[+] Running 4/5
 ✔ Network ssc_default       Created                                                                                                                                                                       4.0s 
 ⠼ Container minio           Starting                                                                                                                                                                     74.9s 
 ✔ Container starrocks-fe    Created                                                                                                                                                                       0.2s 
 ✔ Container ssc-minio_mc-1  Created                                                                                                                                                                       0.2s 
 ✔ Container starrocks-cn    Created                                                                                                                                                                       0.3s 
Error response from daemon: failed to set up container networking: driver failed programming external connectivity on endpoint minio (5c65c33ad8dff4293f10a18ae3c1d193d6cd19c4022b352a2d80fccbc3e39add): failed to bind host port for 0.0.0.0:9000:172.19.0.2:9000/tcp: address already in use
```
что понятно, ибо у нас ClickHouse.  
Далее.   

Ошибка `address already in use` возникает, когда порт 9000 на вашей машине уже занят другим процессом (в вашем случае — ClickHouse).  
Решается это изменением конфигурации портов в файле `docker-compose.yml`.

### 🔍 Проверка занятых портов

Чтобы убедиться, что порт 9000 занят, и найти свободные порты, можно использовать следующие команды.

- **Проверить, какой процесс использует порт 9000:**
    ```bash
    sudo lsof -i :9000
    COMMAND    PID       USER   FD   TYPE    DEVICE SIZE/OFF NODE NAME
    clickhous 8331 clickhouse   43u  IPv4 145100358      0t0  TCP *:cslistener (LISTEN)
    ```
    Или, если `lsof` не установлен:
    ```bash
    sudo netstat -tulpn | grep :9000
    tcp        0      0 0.0.0.0:9000            0.0.0.0:*               LISTEN      8331/clickhouse-ser 
    ```
    Команда покажет PID и имя процесса, который занял порт.
  
- **Проверить свободные порты:**
  ```
  sudo netstat -tulpn | grep :9
  tcp        0      0 0.0.0.0:9030            0.0.0.0:*               LISTEN      855130/docker-proxy 
  tcp        0      0 0.0.0.0:9009            0.0.0.0:*               LISTEN      8331/clickhouse-ser 
  tcp        0      0 0.0.0.0:9002            0.0.0.0:*               LISTEN      478156/docker-proxy 
  tcp        0      0 0.0.0.0:9001            0.0.0.0:*               LISTEN      478138/docker-proxy 
  tcp        0      0 0.0.0.0:9000            0.0.0.0:*               LISTEN      8331/clickhouse-ser 
  tcp        0      0 0.0.0.0:9005            0.0.0.0:*               LISTEN      8331/clickhouse-ser 
  tcp        0      0 0.0.0.0:9004            0.0.0.0:*               LISTEN      8331/clickhouse-ser 
  tcp        0      0 0.0.0.0:9901            0.0.0.0:*               LISTEN      478181/docker-proxy 
  tcp        0      0 0.0.0.0:9900            0.0.0.0:*               LISTEN      478453/docker-proxy 
  ```
  ```
  sudo netstat -tulpn | grep :8
  tcp        0      0 0.0.0.0:8030            0.0.0.0:*               LISTEN      855074/docker-proxy 
  tcp        0      0 0.0.0.0:8040            0.0.0.0:*               LISTEN      855115/docker-proxy 
  tcp        0      0 0.0.0.0:8085            0.0.0.0:*               LISTEN      478624/docker-proxy 
  tcp        0      0 0.0.0.0:8123            0.0.0.0:*               LISTEN      8331/clickhouse-ser 
  tcp        0      0 0.0.0.0:8113            0.0.0.0:*               LISTEN      479376/docker-proxy 
  tcp        0      0 0.0.0.0:8112            0.0.0.0:*               LISTEN      479359/docker-proxy 
  ```
  
- **Проверить статус ClickHouse:**
    ```bash
    sudo systemctl status clickhouse-server
    ```

### 🛠️ Исправление конфигурации Docker Compose

Чтобы устранить конфликт, нужно изменить маппинг портов для MinIO в файле `docker-compose.yml`, скачанный для установки StarRocks.

1.  **Откройте файл `docker-compose.yml`** в текстовом редакторе.
2.  **Найдите секцию, описывающую сервис `minio`.** Она будет выглядеть примерно так:
    ```yaml
    minio:
      # ... другие параметры ...
      ports:
        - 9001:9001
        - 9000:9000
    ```
3.  **Измените внешний порт для `9000`** на любой свободный (например, `9500`):
    ```yaml
      ports:
        - 9501:9001
        - 9500:9000  # Изменяем левую часть (хост:контейнер)
    ```
    Это означает, что порт 9000 внутри контейнера MinIO будет доступен на вашей машине через порт 9500.

    Итого, конечный вид yml файла
    ```
    services:
    minio:
      container_name: minio
      environment:
        MINIO_ROOT_USER: miniouser
        MINIO_ROOT_PASSWORD: miniopassword
      image: minio/minio:latest
      ports:
        - "9501:9001"
        - "9500:9000"
      entrypoint: sh
      command: '-c ''mkdir -p /minio_data/starrocks && minio server /minio_data --console-address ":9001"'''
      healthcheck:
        test: ["CMD", "mc", "ready", "local"]
        interval: 5s
        timeout: 5s
        retries: 5
  
    minio_mc:
      # This service is short lived, it does this:
      # - starts up
      # - checks to see if the MinIO service `minio` is ready
      # - creates a MinIO Access Key that the StarRocks services will use
      # - exits
      image: minio/mc:latest
      entrypoint:
        - sh
        - -c
        - |
          until mc ls minio > /dev/null 2>&1; do
            sleep 0.5
          done
  
          mc alias set myminio http://minio:9000 miniouser miniopassword
          mc admin user svcacct add --access-key AAAAAAAAAAAAAAAAAAAA \
          --secret-key BBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBB \
          myminio \
          miniouser
      depends_on:
          minio:
            condition: service_healthy
  
    starrocks-fe:
      image: starrocks/fe-ubuntu:3.5-latest
      hostname: starrocks-fe
      container_name: starrocks-fe
      user: root
      command:
        - /bin/bash
        - -c
        - |
          echo "# enable shared data, set storage type, set endpoint" >> /opt/starrocks/fe/conf/fe.conf
          echo "run_mode = shared_data" >> /opt/starrocks/fe/conf/fe.conf
          echo "cloud_native_storage_type = S3" >> /opt/starrocks/fe/conf/fe.conf
          echo "aws_s3_endpoint = minio:9000" >> /opt/starrocks/fe/conf/fe.conf
  
          echo "# set the path in MinIO" >> /opt/starrocks/fe/conf/fe.conf
          echo "aws_s3_path = starrocks" >> /opt/starrocks/fe/conf/fe.conf
  
          echo "# credentials for MinIO object read/write" >> /opt/starrocks/fe/conf/fe.conf
          echo "aws_s3_access_key = AAAAAAAAAAAAAAAAAAAA" >> /opt/starrocks/fe/conf/fe.conf
          echo "aws_s3_secret_key = BBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBB" >> /opt/starrocks/fe/conf/fe.conf
          echo "aws_s3_use_instance_profile = false" >> /opt/starrocks/fe/conf/fe.conf
          echo "aws_s3_use_aws_sdk_default_behavior = false" >> /opt/starrocks/fe/conf/fe.conf
  
          echo "# Set this to false if you do not want default" >> /opt/starrocks/fe/conf/fe.conf
          echo "# storage created in the object storage using" >> /opt/starrocks/fe/conf/fe.conf
          echo "# the details provided above" >> /opt/starrocks/fe/conf/fe.conf
          echo "enable_load_volume_from_conf = true" >> /opt/starrocks/fe/conf/fe.conf
  
          /opt/starrocks/fe/bin/start_fe.sh --host_type FQDN
      ports:
        - 8530:8030
        - 9520:9020
        - 9530:9030
      healthcheck:
        test: 'mysql -u root -h starrocks-fe -P 9030 -e "show frontends\G" |grep "Alive: true"'
        interval: 10s
        timeout: 5s
        retries: 3
      depends_on:
          minio:
              condition: service_healthy
  
    starrocks-cn:
      image: starrocks/cn-ubuntu:3.5-latest
      command:
        - /bin/bash
        - -c
        - |
          sleep 15s;
          ulimit -u 65535;
          ulimit -n 65535;
          mysql --connect-timeout 2 -h starrocks-fe -P9030 -uroot -e "ALTER SYSTEM ADD COMPUTE NODE \"starrocks-cn:9050\";"
          /opt/starrocks/cn/bin/start_cn.sh
      environment:
        - HOST_TYPE=FQDN
      ports:
        - 8540:8040
      hostname: starrocks-cn
      container_name: starrocks-cn
      user: root
      depends_on:
        starrocks-fe:
          condition: service_healthy
          restart: true
        minio:
          condition: service_healthy
      healthcheck:
        test: 'mysql -u root -h starrocks-fe -P 9030 -e "SHOW COMPUTE NODES\G" |grep "Alive: true"'
        interval: 10s
        timeout: 5s
        retries: 3
    ```

    
5.  **Сохраните файл.**

### 🔄 Перезапуск установки

После внесения изменений необходимо полностью пересоздать окружение, так как простое изменение файла не обновит уже созданные контейнеры.

1.  **Остановите и удалите текущие контейнеры.** Выполните команду в директории с `docker-compose.yml`:
    ```bash
    sudo docker compose down
    ```
2.  **Запустите контейнеры заново:**
    ```bash
    sudo docker compose up -d
    ```
    Теперь MinIO должен запуститься без ошибок, а его веб-интерфейс будет доступен по адресу `http://localhost:9901`, а S3 API — на порту `9900`.

### 💡 Дополнительные замечания

- **Проверьте другие порты:** Убедитесь, что другие порты, которые использует StarRocks (9030, 8030, 8040), также свободны. При необходимости их можно аналогичным образом переназначить в секциях сервисов `starrocks-fe` и `starrocks-be` в том же файле `docker-compose.yml`.
- **Обновите конфигурацию:** После изменения порта MinIO не забудьте использовать новый порт (`9900`) во всех последующих шагах руководства, например, при создании `STORAGE VOLUME` в StarRocks.

> ### Далее везде по ходу установки не забываем корректировать 9* и 8* на 95* и 85* соотв


### Зайти в контейнер снаружи
Из какталога с конфигурационным файликом (`docker-compose.yml`)
```
sudo docker compose exec -it starrocks-fe /bin/bash
```
Далее смотрим `cd /opt/starrocks/fe/conf/`
```
cat fe.conf 
```

### 🔧 Если нужно пересоздать всё заново
bash
```
# Останавливаем и удаляем всё
sudo docker compose down --remove-orphans --volumes
```
```
# Запускаем заново
sudo docker compose up -d
```
### Правильный вывод
```
sudo docker compose up -d
[+] Running 5/5
 ✔ Network ssc_default       Created                                                                                                                                                                       0.1s 
 ✔ Container minio           Healthy                                                                                                                                                                       7.1s 
 ✔ Container starrocks-fe    Healthy                                                                                                                                                                      27.9s 
 ✔ Container ssc-minio_mc-1  Started                                                                                                                                                                       6.3s 
 ✔ Container starrocks-cn    Started                                                                                                                                                                      28.2s 
[admin@dbcs01 ssc]$ sudo docker compose ps
NAME           IMAGE                            COMMAND                  SERVICE        CREATED         STATUS                        PORTS
minio          minio/minio:latest               "sh -c 'mkdir -p /mi…"   minio          2 minutes ago   Up 2 minutes (healthy)        0.0.0.0:9500->9000/tcp, [::]:9500->9000/tcp, 0.0.0.0:9501->9001/tcp, [::]:9501->9001/tcp
starrocks-cn   starrocks/cn-ubuntu:3.5-latest   "/bin/bash -c 'sleep…"   starrocks-cn   2 minutes ago   Up About a minute (healthy)   0.0.0.0:8540->8040/tcp, [::]:8540->8040/tcp
starrocks-fe   starrocks/fe-ubuntu:3.5-latest   "/bin/bash -c 'echo …"   starrocks-fe   2 minutes ago   Up About a minute (healthy)   0.0.0.0:8530->8030/tcp, [::]:8530->8030/tcp, 0.0.0.0:9520->9020/tcp, [::]:9520->9020/tcp, 0.0.0.0:9530->9030/tcp, [::]:9530->9030/tcp

```




























