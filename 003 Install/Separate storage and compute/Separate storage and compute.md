# [Separate storage and compute](https://docs.starrocks.io/docs/quick_start/shared-data/)
## Prerequisites
4 GB RAM assigned to Docker
```
docker stats
```
Обратите внимание на столбцы MEM USAGE и LIMIT.

Hot _**data is cached locally**_ and When the cache is hit, the query performance is comparable to that of storage-compute coupled architecture.  
Compute nodes _**(CN) can be added**_ or removed on demand _**within seconds**_.  
This architecture _**reduces storage cost**_, ensures _**better resource isolation**_, and provides _**elasticity and scalability**_.

