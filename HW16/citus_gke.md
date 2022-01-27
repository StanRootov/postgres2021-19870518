## Разворачиваем кластер GKE с конфигурацией e2-standard-2:

```bash
gcloud beta container --project "unified-firefly-327616" clusters create "citus" --zone "us-central1-c" --no-enable-basic-auth --cluster-version "1.21.6-gke.1500" --release-channel "regular" --machine-type "e2-medium" --image-type "COS_CONTAINERD" --disk-type "pd-standard" --disk-size "100" --metadata disable-legacy-endpoints=true --scopes "https://www.googleapis.com/auth/cloud-platform" --max-pods-per-node "110" --num-nodes "3" --logging=SYSTEM,WORKLOAD --monitoring=SYSTEM --enable-ip-alias --network "projects/unified-firefly-327616/global/networks/default" --subnetwork "projects/unified-firefly-327616/regions/us-central1/subnetworks/default" --no-enable-intra-node-visibility --default-max-pods-per-node "110" --no-enable-master-authorized-networks --addons HorizontalPodAutoscaling,HttpLoadBalancing,GcePersistentDiskCsiDriver --enable-autoupgrade --enable-autorepair --max-surge-upgrade 1 --max-unavailable-upgrade 0 --enable-shielded-nodes --node-locations "us-central1-c"
```
## Кодируем пароль и пропишем его в манифесте secrets.yaml. Далее запустим поочередно(после создания мастера) создание мастера и воркеров:
```bash
echo -n 'otus321$' | base64

kubectl create -f secrets.yaml
kubectl create -f master.yaml
kubectl get all
kubectl create -f workers.yaml

kubectl get all
```
## Пробросим порт:
```bash
kubectl port-forward pod/citus-master-78ff549b8f-zk9sc 5432
```
## Cоздадим и шардируем таблицу:
```bash
psql -U postgres -h localhost
```
```sql
create table taxi_trips (unique_key text, taxi_id text, trip_start_timestamp TIMESTAMP, trip_end_timestamp TIMESTAMP, trip_seconds bigint, trip_miles numeric, pickup_census_tract bigint, dropoff_census_tract bigint, pickup_community_area bigint, dropoff_community_area bigint, fare numeric, tips numeric, tolls numeric, extras numeric, trip_total numeric, 
payment_type text, company text, pickup_latitude numeric, pickup_longitude numeric, pickup_location text, dropoff_latitude numeric, dropoff_longitude numeric, dropoff_location text);

SELECT create_distributed_table('taxi_trips', 'unique_key');
```

## Загрузим бакет с данными на локальную машину:
```bash
gsutil -m cp -R gs://chicago_taxi_trips_10g .
```

## Загрузим данные в таблицу:
```bash
for f in /Users/sig/courses/OTUS/taxi_trips_10g/chicago_taxi_trips_10g/taxi_trips* 
do
	echo -e "Processing $f file..."
	psql "host=localhost port=5432 dbname=postgres user=postgres password=otus321$" -c "\\COPY taxi_trips FROM PROGRAM 'cat $f' CSV HEADER"
done
```

**Данные загрузились примерно за 5 имнут, что сопоставимо с загрузкой этих данных в обычный инстанс.**

## Проверим производительность запроса:
```sql
\timing

SELECT payment_type,round(sum(tips)/sum(trip_total)*100, 0) + 0 as tips_percent,count(*) as c FROM taxi_trips group by payment_type order by 3;
```
```text
 payment_type | tips_percent |    c     
--------------+--------------+----------
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

Time: 474088,648 ms (07:54,089)
```
## Хотел сохдать индекс и повторно выполним запрос, но получил ошибку, возможно связанную с тем, что закончилась память на координаторе(не менял значение в файлике - 10ГБ):
```sql
CREATE INDEX IDX_payment_type_INCL ON taxi_trips (payment_type,tips,trip_total);
```

ERROR:  server closed the connection unexpectedly
	This probably means the server terminated abnormally
	before or while processing the request.
CONTEXT:  while executing command on citus-worker-0.citus-workers:5432