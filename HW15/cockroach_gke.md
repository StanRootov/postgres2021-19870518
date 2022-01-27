## Разворачиваем кластер GKE с конфигурацией e2-standard-2:

```bash
gcloud beta container --project "unified-firefly-327616" clusters create "cockroach-cluster" --zone "us-central1-c" --no-enable-basic-auth --cluster-version "1.21.6-gke.1500" --release-channel "regular" --machine-type "e2-standard-2" --image-type "COS_CONTAINERD" --disk-type "pd-standard" --disk-size "100" --metadata disable-legacy-endpoints=true --scopes "https://www.googleapis.com/auth/cloud-platform" --max-pods-per-node "110" --num-nodes "3" --logging=SYSTEM,WORKLOAD --monitoring=SYSTEM --enable-ip-alias --network "projects/unified-firefly-327616/global/networks/default" --subnetwork "projects/unified-firefly-327616/regions/us-central1/subnetworks/default" --no-enable-intra-node-visibility --default-max-pods-per-node "110" --no-enable-master-authorized-networks --addons HorizontalPodAutoscaling,HttpLoadBalancing,GcePersistentDiskCsiDriver --enable-autoupgrade --enable-autorepair --max-surge-upgrade 1 --max-unavailable-upgrade 0 --enable-shielded-nodes --node-locations "us-central1-c"
```
## Хельмим CockroachDB с опциями заданными в файле my_values.yaml:

```bash
git clone https://github.com/cockroachdb/helm-charts

cd helm-charts/cockroachdb

helm repo add cockroachdb https://charts.cockroachdb.com/
helm repo update
```
```text
statefulset:
  resources:
    limits:
      memory: 4Gi
      cpu: 500m
    requests:
      memory: 4Gi
      cpu: 500m
conf:
  cache: 10%
  max-sql-memory: 25%
tls:
  enabled: false
```
```bash
helm install cockroach . --values my_values.yaml
```
## Пробрасываем порт для подключения к ноде:
```bash
kubectl port-forward cockroach-cockroachdb-0 26257
psql -h localhost -p 26257 -U root
```
## Создаем таблицу и импортируем в нее данные из GC бакета:
```sql
use postgres;

create table taxi_trips (unique_key text, taxi_id text, trip_start_timestamp TIMESTAMP, trip_end_timestamp TIMESTAMP, trip_seconds bigint, trip_miles numeric, pickup_census_tract bigint, dropoff_census_tract bigint, pickup_community_area bigint, dropoff_community_area bigint, fare numeric, tips numeric, tolls numeric, extras numeric, trip_total numeric, 
payment_type text, company text, pickup_latitude numeric, pickup_longitude numeric, pickup_location text, dropoff_latitude numeric, dropoff_longitude numeric, dropoff_location text);
```
```bash
for ((i=0; i <= 41; i++))
do
if (( i <= 9 )); then
  f="0"${i}
else
  f=${i}
fi
echo "taxi_trips_0000000000"${f}
cockroach sql --insecure \
              --host=localhost:26257 \
              --database="postgres" \
              --execute="IMPORT INTO taxi_trips CSV DATA ('gs://chicago_taxi_trips_10g/taxi_trips_0000000000${f}?AUTH=implicit') WITH delimiter = ',', nullif = '', skip = '1';"
done

cockroach sql --insecure \
              --host=localhost:26257 \
              --database="postgres" \
              --execute="SELECT COUNT(*) FROM taxi_trips;"
```
```text
   count
------------
  27 351 228
(1 row)

Time: 178.958s
```
## Выполним запрос:
```bash
cockroach sql --insecure \
              --host=localhost:26257 \
              --database="postgres" \
              --execute="SELECT payment_type,round(sum(tips)/sum(trip_total)*100, 0) + 0 as tips_percent,count(*) as c FROM taxi_trips group by payment_type order by 3;"
```
```text
  payment_type | tips_percent |    c
---------------+--------------+-----------
  Way2ride     |           15 |       92
  Prepaid      |            0 |       96
  Pcard        |            3 |     5025
  Dispute      |            0 |    16138
  Mobile       |           15 |    32838
  Prcard       |            1 |    73718
  Unknown      |            2 |    96638
  No Charge    |            3 |   144805
  Credit Card  |           17 | 11380018
  Cash         |            0 | 15601860
(10 rows)


Time: 88.071s
```
## Построим индекс и заново выполним запрос:
```bash
cockroach sql --insecure \
              --host=localhost:26257 \
              --database="postgres" \
              --execute="CREATE INDEX IDX_payment_type_INCL ON taxi_trips (payment_type) INCLUDE (tips,trip_total);"
```
```text
  payment_type | tips_percent |    c
---------------+--------------+-----------
  Way2ride     |           15 |       92
  Prepaid      |            0 |       96
  Pcard        |            3 |     5025
  Dispute      |            0 |    16138
  Mobile       |           15 |    32838
  Prcard       |            1 |    73718
  Unknown      |            2 |    96638
  No Charge    |            3 |   144805
  Credit Card  |           17 | 11380018
  Cash         |            0 | 15601860
(10 rows)

Time: 35.357s
```

## Выводы:

**По сравнению с обычным инстаносом PG скорость выполнения запроса почти в 10 раз медленнее (Time: 4588.750 ms (00:04.589) см ДЗ10). Да и сама загрузка данных была ооочень долгой - в среднем по 40сек на 1 файл из 650К строк.**

**Первое с чем столкнулся это ошибки при загрузки данных (см ниже), загружались первые 5-6 частей бакета, потом все падало и обрубались соединения. Решилось правкой файла my_values.yaml и увеличинем лимитов в нем.

Еще заметил, что во время импорта напрочь блокируется таблица.
**            

```text
1) ERROR: driver: bad connection
   ERROR: server closed the connection

2) ERROR: relation "taxi_trips" is offline: importing
```


