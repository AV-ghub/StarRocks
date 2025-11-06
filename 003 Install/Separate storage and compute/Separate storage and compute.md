# [Separate storage and compute](https://docs.starrocks.io/docs/quick_start/shared-data/)
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


Hot _**data is cached locally**_ and When the cache is hit, the query performance is comparable to that of storage-compute coupled architecture.  
Compute nodes _**(CN) can be added**_ or removed on demand _**within seconds**_.  
This architecture _**reduces storage cost**_, ensures _**better resource isolation**_, and provides _**elasticity and scalability**_.

