# Проектная работа
## Тема: Синхронизация обновлений данных между СУБД MS SQL Server и Postgres при помощи Debezium

**Цель:**


```text


```

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

```bash
curl https://packages.microsoft.com/keys/microsoft.asc | sudo apt-key add -

curl https://packages.microsoft.com/config/ubuntu/20.04/prod.list | sudo tee /etc/apt/sources.list.d/msprod.list

sudo add-apt-repository "$(wget -qO- https://packages.microsoft.com/config/ubuntu/20.04/mssql-server-2019.list)"

sudo apt-get update
sudo apt-get install -y mssql-server

sudo /opt/mssql/bin/mssql-conf setup
 123456@Test

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

```sql
CREATE DATABASE TestDB
GO
USE TestDB
GO

CREATE TABLE taxitrips ( uniquekey varchar(100) PRIMARY KEY,taxiid varchar(1000), tripstarttimestamp varchar(40), tripendtimestamp varchar(40), tripseconds bigint, tripmiles float, pickupcensustract bigint, dropoffcensustract bigint, pickupcommunityarea bigint, dropoffcommunityarea bigint, fare float, tips float, tolls float, extras float, trip_total float, paymenttype varchar(40), company varchar(1000), pickuplatitude float, pickuplongitude float, pickuplocation varchar(1000), dropofflatitude float, dropofflongitude float, dropofflocation varchar(1000))
GO

```

for ((i=0; i <= 10; i++))
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

USE TestDB  
GO  
EXEC sys.sp_cdc_enable_db 

EXEC sys.sp_cdc_enable_table  @source_schema = N'dbo', @source_name   = N'taxitrips',  @role_name = N'cdc_role', @supports_net_changes = 1

SELECT job_id,name,enabled,date_created,date_modified FROM msdb.dbo.sysjobs ORDER BY date_created;

SELECT name,is_tracked_by_cdc FROM sys.tables WHERE object_id = OBJECT_ID('dbo.taxi_trips'); 


GKE


gcloud beta container --project "unified-firefly-327616" clusters create "cluster" --zone "us-central1-c" --no-enable-basic-auth --cluster-version "1.21.6-gke.1503" --release-channel "regular" --machine-type "e2-standard-2" --image-type "COS_CONTAINERD" --disk-type "pd-standard" --disk-size "50" --metadata disable-legacy-endpoints=true --scopes "https://www.googleapis.com/auth/cloud-platform" --max-pods-per-node "110" --num-nodes "3" --logging=SYSTEM,WORKLOAD --monitoring=SYSTEM --enable-ip-alias --network "projects/unified-firefly-327616/global/networks/default" --subnetwork "projects/unified-firefly-327616/regions/us-central1/subnetworks/default" --no-enable-intra-node-visibility --default-max-pods-per-node "110" --enable-autoscaling --min-nodes "0" --max-nodes "6" --no-enable-master-authorized-networks --addons HorizontalPodAutoscaling,HttpLoadBalancing,GcePersistentDiskCsiDriver --enable-autoupgrade --enable-autorepair --max-surge-upgrade 1 --max-unavailable-upgrade 0 --node-locations "us-central1-c"


kubectl create namespace kafka \
  && kubectl config set-context --current --namespace=kafka

wget https://github.com/strimzi/strimzi-kafka-operator/releases/download/0.28.0/strimzi-cluster-operator-0.28.0.yaml

cd ~/courses/OTUS/project_debezium
 
sed -i '' 's/namespace: .*/namespace: kafka/' yamls/strimzi-cluster-operator-0.28.0.yaml

kubectl apply -f yamls/strimzi-cluster-operator-0.28.0.yaml -n kafka
   
kubectl apply -f yamls/kafka-persistent.yaml -n kafka \
  && kubectl wait kafka/my-cluster --for=condition=Ready --timeout=300s -n kafka
  
Следите за развертыванием оператора Strimzi Kafka и следить за журналом оператора

kubectl get pod -n kafka --watch
kubectl logs deployment/strimzi-cluster-operator -n kafka -f

Создадим образ для разворачивания Kafka Connect с необходимыми коннекторами:

cd docker
docker buildx build --platform linux/amd64,linux/arm64 --push -t stanrootov/kafka-connect-debezium:1 .
cd ..

Выполним установку Kafka Connect:

kubectl apply -f yamls/kafka-connect.yaml -n kafka

# kubectl delete -f yamls/kafka-connect.yaml -n kafka
 
kubectl get deployments -n kafka

Postgres:

kubectl create namespace postgres

kubectl apply -f yamls/postgres/postgres-configmap.yaml -n postgres \
&& kubectl apply -f yamls/postgres/postgres-deployment.yaml -n postgres \
&& kubectl apply -f yamls/postgres/postgres-pvc.yaml -n postgres \
&& kubectl apply -f yamls/postgres/postgres-loadbalancer.yaml -n postgres

kubectl get all -n postgres

psql -p 5432 -h 34.121.40.138 -U postgresadmin -d testdb

pass: admin123


mssql source connector
 
cat debconnectors/sql-server-source.json | kubectl exec -i -n kafka my-cluster-kafka-0 -- curl -X POST -H "Accept:application/json" -H "Content-Type:application/json" http://my-connect-cluster-connect-api:8083/connectors -d @-

cat debconnectors/postgres-sink.json| kubectl exec -i -n kafka my-cluster-kafka-0 -- curl -X POST -H "Accept:application/json" -H "Content-Type:application/json" http://my-connect-cluster-connect-api:8083/connectors -d @-

проверка:

kubectl exec -i -n kafka my-cluster-kafka-0 -- curl -X GET http://my-connect-cluster-connect-api:8083/connectors/sql-server-connector/status \
 && kubectl exec -i -n kafka my-cluster-kafka-0 -- curl -X GET http://my-connect-cluster-connect-api:8083/connectors/postgres-sink/status

kubectl exec -i -n kafka my-cluster-kafka-0 -- curl -X GET http://my-connect-cluster-connect-api:8083/connectors


удаление

kubectl exec -i -n kafka my-cluster-kafka-0 -- curl -X DELETE http://my-connect-cluster-connect-api:8083/connectors/postgres-sink \
&& kubectl exec -i -n kafka my-cluster-kafka-0 -- curl -X DELETE http://my-connect-cluster-connect-api:8083/connectors/sql-server-connector


sql:

kubectl exec -n kafka my-cluster-kafka-0   -- bin/kafka-topics.sh --list --zookeeper localhost:9092

pg 


kubectl exec -i -n kafka my-cluster-kafka-0 -- curl -i -X POST -H "Accept:application/json" -H  "Content-Type:application/json" http://my-connect-cluster-connect-api:8083/connectors/ -d @register-sqlserver.json



connector status:


kubectl exec -i -n kafka my-cluster-kafka-0 -- curl -X GET http://my-connect-cluster-connect-api:8083/connectors/sql-server-connector/status


kubectl exec -n kafka my-cluster-kafka-0 -- bin/kafka-topics.sh --bootstrap-server 10.4.14.219:9092 --list


удаление коннектора

kubectl exec -i -n kafka my-cluster-kafka-0 -- curl -X DELETE http://my-connect-cluster-connect-api:8083/connectors/sql-server-connector

kubectl logs deployment.apps/my-connect-cluster-connect -n kafka -f  | grep "sql-server-connector"


просмотр данных в топиках


kubectl exec -i -n kafka my-cluster-kafka-0 -- bin/kafka-console-consumer.sh --bootstrap-server 10.4.14.219:9092 --topic mssql.dbo.taxitrips --max-messages 1 --from-beginning



kubectl logs deployment.apps/my-connect-cluster-connect -n kafka -f 

Он используется в Debezium MySqlConnector: отправляет данные в Kafka топики с именами {server_name}.{database_name}.{table_name}, а JdbcSinkConnector принимает {database_name}.{schema_name}.{table_name}. Так как целевая база и таблица могут отличаться по именам (и у вас вряд ли имя базы будет public), то этот коннектор изменяет целевое имя топика.

sqlcmd -S localhost -U SA -P "123456@Test"

USE testdb; 
GO

UPDATE taxitrips SET tips = 55 WHERE uniquekey = '00001b648cec5f97cf60e8823a42617d181ea2d6';
GO

SELECT * FROM taxitrips WHERE uniquekey = '00001b648cec5f97cf60e8823a42617d181ea2d6';
GO


SELECT * FROM taxi_trips WHERE unique_key = '00001b648cec5f97cf60e8823a42617d181ea2d6'\gx

monitoring

kubectl create namespace monitor



wget https://raw.githubusercontent.com/coreos/prometheus-operator/master/bundle.yaml 


sed -i '' 's/namespace: .*/namespace: monitor/' yamls/prometheus-install/prometheus-operator-deployment.yaml

kubectl replace -f yamls/prometheus-install/prometheus-operator-deployment.yaml -n monitor