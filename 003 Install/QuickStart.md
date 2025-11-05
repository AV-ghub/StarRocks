# [Deploy StarRocks with Docker](https://docs.starrocks.io/docs/quick_start/shared-nothing/)
[ClickHouse уже не один: StarRocks показывает, что lakehouse-аналитика может быть проще и быстрее»](https://habr.com/ru/articles/956334/)

```
docker run -p 9030:9030 -p 8030:8030 -p 8040:8040 -itd --name quickstart starrocks/allin1-ubuntu
```

```
sudo docker exec -it quickstart   mysql -P 9030 -h 127.0.0.1 -u root --prompt="StarRocks > "
```
