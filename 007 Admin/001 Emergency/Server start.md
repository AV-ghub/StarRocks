##### [Deploy StarRocks manually](https://docs.starrocks.io/docs/deployment/deploy_manually/)
```bash
./fe/bin/start_fe.sh --daemon
./be/bin/start_be.sh --daemon
```
##### [Table not available (Backend node inBlacklist)](https://github.com/AV-ghub/StarRocks/blob/main/007%20Admin/001%20Emergency/Table%20not%20available%20(Backend%20node%20inBlacklist).md)
```bash
SHOW PROC '/backends';

CANCEL DECOMMISSION BACKEND "sl11-demostand-sr-01:9050";

-- Удаляем BE (осторожно, убедитесь, что это единственный BE!)
ALTER SYSTEM DROP BACKEND "sl11-demostand-sr-01:9050" FORCE;

-- Добавляем его снова
ALTER SYSTEM ADD BACKEND "sl11-demostand-sr-01:9050";
```
