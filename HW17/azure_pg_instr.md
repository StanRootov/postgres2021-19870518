# Virtual Machines

Создание виртуальной машины:
group name: test-gr-vm 
name vm: test-db-pg-vm-1
username: test
name ssh-key: test_key

Подключение к виртуальной машине:
https://docs.microsoft.com/ru-ru/azure/virtual-machines/linux/quick-create-portal

cd ~/test/PostgreSQL/6_Cloud_2021_09/33_PG_Azure/demo/
chmod 400 test_key.pem
ssh -i test_key.pem test@40.84.2.214

Установка Postgres:
sudo apt update && sudo apt upgrade -y && sudo sh -c 'echo "deb http://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list' && wget --quiet -O - https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo apt-key add - && sudo apt-get update && sudo apt-get -y install postgresql && sudo apt install unzip && sudo apt -y install mc

pg_lsclusters

Установить пароль для Postgres:
sudo -u postgres psql
\password   #12345
exit

Добавить правило pg-rl входящего порта 5432 в Azure

Добавить сетевые правила для подключения к Postgres:
cd /etc/postgresql/14/main/
sudo nano /etc/postgresql/14/main/postgresql.conf
#listen_addresses = 'localhost'
listen_addresses = '*'

sudo nano /etc/postgresql/14/main/pg_hba.conf
#host    all             all             127.0.0.1/32            md5 password
host    all             all             0.0.0.0/0               md5 

sudo pg_ctlcluster 14 main restart

Подключение к Postgres:
psql -h 40.84.2.214 -U postgres

\c postgres
\l
\du
\dt
\dt+
\d table

# Azure Database for PostgreSQL
https://docs.microsoft.com/ru-ru/azure/postgresql/flexible-server/concepts-compare-single-server-flexible-server

## Создание
```bash
# Вход с помощью Azure CLI:
# az login --use-device-code
# https://aka.ms/devicelogin
# az login --tenant 8ee46cc5-a1da-4c38-a8eb-b4e74482760e

cd ~/test/PostgreSQL/6_Cloud_2021_09/33_PG_Azure/demo/
az login
Сохраняем в буфере id:
"id": "7f6c4d0d-e0e3-437a-8cf8-a11bc14d02b1"

az account list
az account show

Выбор конкретного идентификатора подписки:
az account set --subscription 7f6c4d0d-e0e3-437a-8cf8-a11bc14d02b1

Создание группы ресурсов Azure:
az group create --name test-gr --location eastus
az group list

Создание сервера Базы данных Azure для PostgreSQL:
az postgres server list-skus -l eastus
--sku-name B_Gen5_2 — ценовая категория "Базовый", поколение 5, 2 виртуальных ядра;
--sku-name GP_Gen5_32 — "Общего назначения", поколение 5, 32 виртуальных ядра;
--sku-name MO_Gen5_2 — "Оптимизированная для операций в памяти", поколение 5, 2 виртуальных ядра.
az postgres server create --resource-group test-gr --name test-db-pg-ss-1  --location eastus --admin-user test --admin-password pasS@12345 --sku-name B_Gen5_2 --storage-size 10240 --version 11

az postgres server list

Настройка правила брандмауэра на уровне сервера:
az postgres server firewall-rule create --resource-group test-gr --server test-db-pg-ss-1 --name AllowMyIP --start-ip-address 46.138.200.102 --end-ip-address 46.138.200.102

Получение сведений о подключении:
az postgres server show --resource-group test-gr --name test-db-pg-ss-1

Конфигурация:
az postgres server configuration list --resource-group test-gr --server-name test-db-pg-ss-1
az postgres server configuration show --name autovacuum --resource-group test-gr --server-name test-db-pg-ss-1
```

## Подключение
Azure Postgres Commands: https://docs.microsoft.com/en-us/cli/azure/postgres
https://docs.microsoft.com/ru-ru/cli/azure/postgres/server?view=azure-cli-latest

```bash
cd ~/test/PostgreSQL/6_Cloud_2021_09/33_PG_Azure/demo/

Подключение к Базе данных Azure для PostgreSQL с помощью psql:
#az postgres server update --resource-group test-gr --name test-db-pg-ss-1 --ssl-enforcement Enabled

https://docs.microsoft.com/en-us/azure/postgresql/concepts-ssl-connection-security

curl "https://cacerts.digicert.com/BaltimoreCyberTrustRoot.crt.pem" -o bctr.crt.pem
chmod 0644 bctr.crt.pem

#psql "host=test-db-pg-ss-1.postgres.database.azure.com port=5432 dbname=postgres user=test@test-db-pg-ss-1 password=pasS@12345 sslmode=disable"
#psql "host=test-db-pg-ss-1.postgres.database.azure.com port=5432 dbname=postgres user=test@test-db-pg-ss-1 password=12345 sslmode=require"
psql "host=test-db-pg-ss-1.postgres.database.azure.com port=5432 dbname=postgres user=test@test-db-pg-ss-1 password=pasS@12345 sslmode=verify-full sslrootcert=bctr.crt.pem"
```

```sql
CREATE DATABASE test_db;
-- DROP DATABASE test_db;

\c test_db

CREATE TABLE weather (
    city            varchar(80),
    temp_lo         int,           -- low temperature
    temp_hi         int,           -- high temperature
    prcp            real,          -- precipitation
    date            date
);
```

```bash
Создание реплики и подключение к реплики:

az postgres server replica create -n test-db-pg-ss-1-rp -g test-gr -s test-db-pg-ss-1
az postgres server replica list -g test-gr -s test-db-pg-ss-1

cd ~/test/PostgreSQL/6_Cloud_2021_09/33_PG_Azure/demo/
psql "host=test-db-pg-ss-1-rp.postgres.database.azure.com port=5432 dbname=postgres user=test@test-db-pg-ss-1-rp password=pasS@12345 sslmode=verify-full sslrootcert=bctr.crt.pem"

az postgres server firewall-rule create --resource-group test-gr --server test-db-pg-ss-1-rp --name AllowMyIP --start-ip-address 95.52.16.219 --end-ip-address 95.52.16.219

psql "host=test-db-pg-ss-1-rp.postgres.database.azure.com port=5432 dbname=postgres user=test@test-db-pg-ss-1-rp password=pasS@12345 sslmode=verify-full sslrootcert=bctr.crt.pem"

\conninfo
\timing
```

```sql
--  на мастере
select * from weather;
--  на реплике
select * from weather;

--  на мастере
insert into weather values ('San Francisco', 46, 50, 0.25, '1994-11-27');
--  на реплике
select * from weather;
```

```bash
az postgres server replica stop -g test-gr -n test-db-pg-ss-1-rp
```

```sql
--  на мастере
insert into weather values ('New York', 46, 50, 0.25, '1994-11-27');
--  на реплике
insert into weather values ('Washington', 46, 50, 0.25, '1994-11-27');
--  на мастере и на реплике
select * from weather;
```
```sql
create table taxi_trips (unique_key text, taxi_id text, trip_start_timestamp TIMESTAMP, trip_end_timestamp TIMESTAMP, trip_seconds bigint, trip_miles numeric, pickup_census_tract bigint, dropoff_census_tract bigint, pickup_community_area bigint, dropoff_community_area bigint, fare numeric, tips numeric, tolls numeric, extras numeric, trip_total numeric, 
payment_type text, company text, pickup_latitude numeric, pickup_longitude numeric, pickup_location text, dropoff_latitude numeric, dropoff_longitude numeric, dropoff_location text);
```
```bash
for f in /Users/sig/courses/OTUS/taxi_trips_10g/chicago_taxi_trips_10g/taxi_trips* 
do
	echo -e "Processing $f file..."
	psql "host=test-db-pg-ss-1.postgres.database.azure.com port=5432 dbname=test_db user=test@test-db-pg-ss-1 password=pasS@12345 sslmode=verify-full sslrootcert=bctr.crt.pem" -c "\\COPY taxi_trips FROM PROGRAM 'cat $f' CSV HEADER"
done
```

# Azure Data Factory
http://eforexcel.com/wp/downloads-18-sample-csv-files-data-sets-for-testing-sales/

```sql
-- Создать таблицу:

CREATE TABLE test (
    Region VARCHAR(50),
    Country VARCHAR(50),
    ItemType VARCHAR(50),
    SalesChannel VARCHAR(20),
    OrderPriority VARCHAR(10),
    OrderDate VARCHAR(10),
    OrderID int,
    ShipDate VARCHAR(10),
    UnitsSold int,
    UnitPrice decimal(12,2),
    UnitCost decimal(12,2),
    TotalRevenue decimal(12,2),
    TotalCost decimal(12,2),
    TotalProfit decimal(12,2)
);


-- Создать Учетная запись хранения: testas1
-- Включить иерархическое пространство имен
-- Создать контейнер: test-as-c1

-- Создать Data Factory: test-df-1
-- host=test-db-pg-ss-1.postgres.database.azure.com port=5432 
-- dbname=postgres 
-- user=test@test-db-pg-ss-1 
-- password=pasS@12345


SELECT count(*) FROM test;
SELECT count(*) FROM test WHERE unitssold=1517;

truncate table test;
SELECT count(*) FROM test;
CREATE index test_idx on test(unitssold);
```


# Очистка ресурсов
az group list --query "[?location=='eastus']"
az group delete --name test-gr 
az group delete --name NetworkWatcherRG