# Проектная работа
## Тема: Синхронизация обновлений данных между СУБД MS SQL Server и PostgreSQL при помощи Debezium.

**Развернем инстанс GCE для MS SQL Server:**


```bash
gcloud compute instances create mssql --project=unified-firefly-327616 --zone=us-central1-a --machine-type=e2-medium --network-interface=network-tier=PREMIUM,subnet=default --maintenance-policy=MIGRATE --service-account=486739247397-compute@developer.gserviceaccount.com --scopes=https://www.googleapis.com/auth/cloud-platform --tags=http-server,https-server --create-disk=auto-delete=yes,boot=yes,device-name=mssql,image=projects/ubuntu-os-cloud/global/images/ubuntu-2004-focal-v20220204,mode=rw,size=30,type=projects/unified-firefly-327616/zones/us-central1-a/diskTypes/pd-balanced --no-shielded-secure-boot --shielded-vtpm --shielded-integrity-monitoring --reservation-affinity=any
```
```bash
gcloud compute ssh mssql

sudo parted /dev/sda
resizepart
1
100%
quit

sudo apt-get update 
sudo apt install curl
```
**Установим MS SQL Server 2019 и выполним его конфигурацию, зададим пароль 123456@Test для пользователя SA:**
```bash
curl https://packages.microsoft.com/keys/microsoft.asc | sudo apt-key add -

curl https://packages.microsoft.com/config/ubuntu/20.04/prod.list | sudo tee /etc/apt/sources.list.d/msprod.list

sudo add-apt-repository "$(wget -qO- https://packages.microsoft.com/config/ubuntu/20.04/mssql-server-2019.list)"

sudo apt-get update
sudo apt-get install -y mssql-server

sudo /opt/mssql/bin/mssql-conf setup
 
sudo apt-get update 
sudo apt-get install mssql-tools unixodbc-dev

echo 'export PATH="$PATH:/opt/mssql-tools/bin"' >> ~/.bash_profile
echo 'export PATH="$PATH:/opt/mssql-tools/bin"' >> ~/.bashrc
source ~/.bashrc

sudo /opt/mssql/bin/mssql-conf set sqlagent.enabled true 
sudo systemctl restart mssql-server

```bash
sqlcmd -S localhost -U SA -P "123456@Test"
```
**Создадим БД и таблицу taxitrips, наполним ее данными:**
```sql
CREATE DATABASE TestDB
GO
USE TestDB
GO

CREATE TABLE taxitrips ( uniquekey varchar(100) PRIMARY KEY,taxiid varchar(1000), tripstarttimestamp varchar(40), tripendtimestamp varchar(40), tripseconds bigint, tripmiles float, pickupcensustract bigint, dropoffcensustract bigint, pickupcommunityarea bigint, dropoffcommunityarea bigint, fare float, tips float, tolls float, extras float, trip_total float, paymenttype varchar(40), company varchar(1000), pickuplatitude float, pickuplongitude float, pickuplocation varchar(1000), dropofflatitude float, dropofflongitude float, dropofflocation varchar(1000))
GO
```
```bash
for ((i=0; i <= 2; i++))
do
if (( i <= 9 )); then
  f="0"${i}
else
  f=${i}
fi

scp /Users/sig/courses/OTUS/taxi_trips_10g/chicago_taxi_trips_10g/taxi_trips_0000000000${f} sig@34.69.110.49:/home/sig/test_dataset/taxi_trips_0000000000${f}.csv

done

for ((i=0; i < 2; i++))
do
if (( i <= 9 )); then
  f="0"${i}
else
  f=${i}
fi

    bcp "taxitrips" in \
    "/home/sig/test_dataset/taxi_trips_0000000000${f}.csv" \
    -S localhost \
    -U SA -P "123456@Test" \
    -d TestDB \
    -c \
    -t ","

done
```
**Включим CDC и проверим его активность для нужной таблицы:**
```sql
USE TestDB  
GO  
EXEC sys.sp_cdc_enable_db 

EXEC sys.sp_cdc_enable_table  @source_schema = N'dbo', @source_name   = N'taxitrips',  @role_name = N'cdc_role', @supports_net_changes = 1

SELECT job_id,name,enabled,date_created,date_modified FROM msdb.dbo.sysjobs ORDER BY date_created;

SELECT name,is_tracked_by_cdc FROM sys.tables WHERE object_id = OBJECT_ID('dbo.taxi_trips');
```


**Создадим кластер GKE:**
```bash
gcloud beta container --project "unified-firefly-327616" clusters create "cluster" --zone "us-central1-c" --no-enable-basic-auth --cluster-version "1.21.6-gke.1503" --release-channel "regular" --machine-type "e2-standard-2" --image-type "COS_CONTAINERD" --disk-type "pd-standard" --disk-size "50" --metadata disable-legacy-endpoints=true --scopes "https://www.googleapis.com/auth/cloud-platform" --max-pods-per-node "110" --num-nodes "3" --logging=SYSTEM,WORKLOAD --monitoring=SYSTEM --enable-ip-alias --network "projects/unified-firefly-327616/global/networks/default" --subnetwork "projects/unified-firefly-327616/regions/us-central1/subnetworks/default" --no-enable-intra-node-visibility --default-max-pods-per-node "110" --enable-autoscaling --min-nodes "0" --max-nodes "6" --no-enable-master-authorized-networks --addons HorizontalPodAutoscaling,HttpLoadBalancing,GcePersistentDiskCsiDriver --enable-autoupgrade --enable-autorepair --max-surge-upgrade 1 --max-unavailable-upgrade 0 --node-locations "us-central1-c"
```
**Для удобства создадим пространство имен kafka:**
```bash
kubectl create namespace kafka \
  && kubectl config set-context --current --namespace=kafka
```

**Скачаем и задеплоим Strimzi Cluster Operator последней версии, не забываем указать пространство имен и необходимый лимит ресурсов:**

```bash
wget https://github.com/strimzi/strimzi-kafka-operator/releases/download/0.28.0/strimzi-cluster-operator-0.28.0.yaml

cd ~/courses/OTUS/project_debezium
 
sed -i '' 's/namespace: .*/namespace: kafka/' yamls/strimzi-cluster-operator-0.28.0.yaml

kubectl apply -f yamls/strimzi-cluster-operator-0.28.0.yaml -n kafka
```

**Выполним установку кластера Kafka(3 реплики кластера + 3 реплики Zookeper), не забываем указать пространство имен и необходимый лимит ресурсов:**

```bash   
kubectl apply -f yamls/kafka-persistent.yaml -n kafka \
  && kubectl wait kafka/my-cluster --for=condition=Ready --timeout=300s -n kafka
```
 
 **Можно следить за развертыванием подов и за журналом деплоя:**

```bash
kubectl get pod -n kafka --watch
kubectl logs deployment/strimzi-cluster-operator -n kafka -f
```

 **Далее необходимо создать образ Kafka Connect c необходимыми коннекторами, для этого необходимо скачать эти коннекторы и поместить их в папку plugins рядом с файлом Dockerfile:**
```docker 
FROM quay.io/strimzi/kafka:0.28.0-kafka-3.1.0
USER root:root
RUN mkdir -p /opt/kafka/plugins/debezium
COPY ./plugins/ /opt/kafka/plugins/debezium/
USER 1001
```
```bash
cd docker
docker buildx build --platform linux/amd64,linux/arm64 --push -t stanrootov/kafka-connect-debezium:1 .
cd ..
```

**Выполним установку Kafka Connect:**

```bash
kubectl apply -f yamls/kafka-connect.yaml -n kafka

# kubectl delete -f yamls/kafka-connect.yaml -n kafka
 
kubectl get deployments -n kafka
```

**Выполним установку Postgres, для удобства используем отдельное пространство имен:**

```bash
kubectl create namespace postgres

kubectl apply -f yamls/postgres/postgres-configmap.yaml -n postgres \
&& kubectl apply -f yamls/postgres/postgres-deployment.yaml -n postgres \
&& kubectl apply -f yamls/postgres/postgres-pvc.yaml -n postgres \
&& kubectl apply -f yamls/postgres/postgres-loadbalancer.yaml -n postgres

kubectl get all -n postgres

psql -p 5432 -h 34.121.40.138 -U postgresadmin -d testdb

#pass: admin123
```

**Создаем 2 коннектора - sql-server-source и postgres-sink, не забываем указать IP подов и IP инстанса MS SQL:**

```bash 
cat debconnectors/sql-server-source.json | kubectl exec -i -n kafka my-cluster-kafka-0 -- curl -X POST -H "Accept:application/json" -H "Content-Type:application/json" http://my-connect-cluster-connect-api:8083/connectors -d @-

cat debconnectors/postgres-sink.json| kubectl exec -i -n kafka my-cluster-kafka-0 -- curl -X POST -H "Accept:application/json" -H "Content-Type:application/json" http://my-connect-cluster-connect-api:8083/connectors -d @-
```

**Проверим их создание и статусы:**

```bash 
kubectl exec -i -n kafka my-cluster-kafka-0 -- curl -X GET http://my-connect-cluster-connect-api:8083/connectors/sql-server-connector/status

kubectl exec -i -n kafka my-cluster-kafka-0 -- curl -X GET http://my-connect-cluster-connect-api:8083/connectors/sql-server-connector/status \
 && kubectl exec -i -n kafka my-cluster-kafka-0 -- curl -X GET http://my-connect-cluster-connect-api:8083/connectors/postgres-sink/status
```

**Также по логу можно наблюдать за созданием коннекторов, инициализации снапшота на источнике и синхронизации данных в PG:**

```bash 

kubectl logs deployment.apps/my-connect-cluster-connect -n kafka -f

kubectl logs deployment.apps/my-connect-cluster-connect -n kafka -f  | grep "sql-server-connector"

kubectl logs deployment.apps/my-connect-cluster-connect -n kafka -f  | grep "ERROR"

```

**Посмотрим список топиков и попробуем прочитать сообщение из очереди:**

```bash 
kubectl exec -n kafka my-cluster-kafka-0 -- bin/kafka-topics.sh --bootstrap-server 10.4.14.219:9092 --list

kubectl exec -i -n kafka my-cluster-kafka-0 -- bin/kafka-console-consumer.sh --bootstrap-server 10.4.14.219:9092 --topic mssql.dbo.taxitrips --max-messages 1 --from-beginning
```

```text
**ПРИМЕЧАНИЕ: Debezium MySqlConnector отправляет данные в Kafka топики с именами {server_name}.{database_name}.{table_name}, а JdbcSinkConnector принимает {database_name}.{schema_name}.{table_name}, чтобы управлять этим и не только, используем transform свойства при создании коннектора.
```

**Выполним проверку работы функционала, ждля этого подключаемся к MS SQL и меняем данные:**

```bash
sqlcmd -S localhost -U SA -P "123456@Test"
```

```sql
USE testdb; 
GO

UPDATE taxitrips SET tips = 55 WHERE uniquekey = '00001b648cec5f97cf60e8823a42617d181ea2d6';
GO

SELECT * FROM taxitrips WHERE uniquekey = '00001b648cec5f97cf60e8823a42617d181ea2d6';
GO
```
**Идем в PG и проверяем изменения:**

```bash
psql -p 5432 -h 34.121.40.138 -U postgresadmin -d testdb

#pass: admin123
```

```sql
SELECT * FROM taxi_trips WHERE unique_key = '00001b648cec5f97cf60e8823a42617d181ea2d6'\gx
```

