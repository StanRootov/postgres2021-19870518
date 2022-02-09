# Virtual Machines (Compute Cloud)

Создание группы виртуальных машин:
https://cloud.yandex.ru/docs/compute/quickstart/ig

Создание виртуальной машины:
name vm: test-db-pg-vm-1

Создать сеть:
Каталог: default
Имя: test-vm-db-pg-net-1

Создать подсеть:
Имя: test-db-pg-subnet-1
CIDR: 10.0.0.0/24

Доступ
username: test

Сгенерировать ssh-key:
```bash
cd ~/test/PostgreSQL/6_Cloud_2021_09/34_PG_Yandex_Cloud/demo/
ssh-keygen -t rsa -b 2048
name ssh-key: test_key
#passphrase: pasS$12345
chmod 400 test_key.pub
cat test_key.pub

Подключение к VM:
https://cloud.yandex.ru/docs/compute/operations/vm-connect/ssh
#ssh test@62.84.115.150
ssh -i test_key test@84.201.135.235

Установка Postgres:
sudo apt update && sudo apt upgrade -y && sudo sh -c 'echo "deb http://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list' && wget --quiet -O - https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo apt-key add - && sudo apt-get update && sudo apt-get -y install postgresql && sudo apt install unzip && sudo apt -y install mc

pg_lsclusters

Установить пароль для Postgres:
sudo -u postgres psql
\password   #12345
\q

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
psql -h 84.201.135.235 -U postgres

\c postgres
\l
\du
\dt
\dt+
\d table
\q

exit
```

# Yandex Managed Service for PostgreSQL

Начало работы с интерфейсом командной строки (CLI)

#yc init
yc config list

OAuth token: sddsa1234567890rf32f2344f32fc243534434
cloud-id: 123456789082012326ic          
folder-id: b1gpuq4cpp911gufp13d
compute-default-zone: ru-central1-a

## Создание
https://cloud.yandex.ru/docs/managed-postgresql/operations/cluster-create
```bash

yc vpc network create --name test-db-pg-net-1 \
    --description "test-db-pg-net-1"

id: enp7vjcov6i47unsjh9h
folder_id: b1gpuq4cpp911gufp13d
created_at: "2022-02-08T19:34:13Z"
name: test-db-pg-net-1
description: test-db-pg-net-1

yc vpc network list

yc vpc subnet create --name test-db-pg-subnet-a-1 \
  --description "test-db-pg-subnet-a-1" \
  --folder-id b1gpuq4cpp911gufp13d \
  --network-id enp7vjcov6i47unsjh9h \
  --zone ru-central1-a \
  --range 10.135.0.0/24


id: e9b9v1akur83kno96fqk
folder_id: b1gpuq4cpp911gufp13d
created_at: "2022-02-08T19:36:32Z"
name: test-db-pg-subnet-a-1
description: test-db-pg-subnet-a-1
network_id: enp7vjcov6i47unsjh9h
zone_id: ru-central1-a
v4_cidr_blocks:
- 10.135.0.0/24


-- id: e9blbrcmohpdn9ec753j

yc vpc subnet create --name test-db-pg-subnet-b-1 \
  --description "test-db-pg-subnet-b-1" \
  --folder-id b1gpuq4cpp911gufp13d \
  --network-id enp7vjcov6i47unsjh9h \
  --zone ru-central1-b \
  --range 10.136.0.0/24

-- id: e2lfes0rbm3vbtsc7v5s

yc vpc subnet list

yc managed-postgresql cluster create --help

yc managed-postgresql cluster create \
   --name test-db-pg-msp-1 \
   --environment production \
   --network-name test-db-pg-net-1 \
   --host zone-id=ru-central1-a,subnet-id=e9b9v1akur83kno96fqk,assign-public-ip \
   --host zone-id=ru-central1-b,subnet-id=e2lguvqod5uk3thafneo,assign-public-ip \
   --resource-preset b2.medium \
   --user name=test,password=pasS@12345 \
   --database name=test_db,owner=test \
   --disk-size 10 \
   --disk-type  network-ssd \
   --datalens-access \
   --websql-access \
   --serverless-access \
   --postgresql-version 13

--done (6m38s)
yc managed-postgresql cluster list
yc managed-postgresql database list \
     --cluster-name test-db-pg-msp-1
```

## Подключение
https://cloud.yandex.ru/docs/managed-postgresql/operations/connect

```bash
cd ~/test/PostgreSQL/6_Cloud_2021_09/34_PG_Yandex_Cloud/demo/

curl "https://storage.yandexcloud.net/cloud-certs/CA.pem" -o ya_ca.crt
chmod 0644 ya_ca.crt

yc managed-postgresql host add --help
yc managed-postgresql host list --cluster-name test-db-pg-msp-1

psql "host=rc1a-yplowi89elx5nqp6.mdb.yandexcloud.net,rc1b-0jhg51c38jczca60.mdb.yandexcloud.net \
      port=6432 \
      sslmode=verify-full \
      sslrootcert=ya_ca.crt \
      dbname=test_db \
      user=test \
      password=pasS@12345 \
      target_session_attrs=read-write"
\conninfo

psql "host=rc1a-yplowi89elx5nqp6.mdb.yandexcloud.net \
      port=6432 \
      sslmode=verify-full \
      sslrootcert=ya_ca.crt \
      dbname=test_db \
      user=test \
      password=pasS@12345 \
      target_session_attrs=read-write"

psql "host=rc1b-0jhg51c38jczca60.mdb.yandexcloud.net \
      port=6432 \
      sslmode=verify-full \
      sslrootcert=ya_ca.crt \
      dbname=test_db \
      user=test \
      password=pasS@12345 \
      target_session_attrs=any"

# \password #pasS@12345
```

```sql
SELECT * FROM pg_replication_slots \gx
SELECT * FROM pg_stat_replication \gx
select * from pg_settings where name like '%port%';

rc1b-n1me9z9mgj066q4v


CREATE TABLE test(s text);
INSERT INTO test SELECT 'Just a line' FROM generate_series(1,100000);
SELECT *, pg_current_wal_lsn() from pg_stat_replication \gx

cd ~/test/PostgreSQL/6_Cloud_2021_09/34_PG_Yandex_Cloud/demo/
psql "host=rc1b-y17qt9mysa6ieb4t.mdb.yandexcloud.net \
      port=6432 \
      sslmode=verify-full \
      sslrootcert=ya_ca.crt \
      dbname=test_db \
      user=test \
      password=pasS@12345 \
      target_session_attrs=any"

select count(*) from test;
INSERT INTO test VALUES ('Replica');

-- CREATE DATABASE test_db;
-- DROP DATABASE test_db;
-- \c test_db
```

# Yandex Data Transfer 
```bash
### Источник Azure

cd ~/test/PostgreSQL/6_Cloud_2021_09/33_PG_Azure/demo/
psql "host=test-ps-1.postgres.database.azure.com port=5432 dbname=postgres user=test@test-ps-1 password=pasS@12345 sslmode=verify-full sslrootcert=bctr.crt.pem"
CREATE DATABASE test_db;
-- DROP DATABASE test_db;
\c test_db

Имя pdb-1
host=test-ps-1.postgres.database.azure.com
     test-ps-1.postgres.database.azure.com
port=5432
dbname=test_db
user=test@test-ps-1
password=pasS@12345
table=public.test

### Приёмник
Имя pdb-2
dbname=test_db
user=test
password=pasS@12345

### Трансфер
transfer-1
```

Example Data File:
http://eforexcel.com/wp/downloads-18-sample-csv-files-data-sets-for-testing-sales/

Создать таблицу:
```sql
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

--alter table if exists "100000SR_csv" rename to test;
--truncate table test;
SELECT * FROM test limit 1 \gx
SELECT count(*) FROM test;
SELECT count(*) FROM test WHERE unitssold=124;
CREATE index test_idx on test(unitssold);
```

# Yandex Database
https://cloud.yandex.ru/docs/ydb/
https://www.youtube.com/watch?v=MlSdUq5RIN8

Name: test-db-ydb-1

```sql
CREATE TABLE series
(
    series_id Uint64,
    title Utf8,
    series_info Utf8,
    release_date Uint64,
    PRIMARY KEY (series_id)
);

CREATE TABLE seasons
(
    series_id Uint64,
    season_id Uint64,
    title Utf8,
    first_aired Uint64,
    last_aired Uint64,
    PRIMARY KEY (series_id, season_id)
);

CREATE TABLE episodes
(
    series_id Uint64,
    season_id Uint64,
    episode_id Uint64,
    title Utf8,
    air_date Uint64,
    PRIMARY KEY (series_id, season_id, episode_id)
);

ALTER TABLE episodes ADD COLUMN is_deleted Bool;
ALTER TABLE episodes DROP COLUMN is_deleted;

DROP TABLE actors;

REPLACE INTO episodes (series_id, season_id, episode_id, title) VALUES (1, 1, 1, "Yesterday's Jam");
REPLACE INTO episodes (series_id, season_id, episode_id, title) VALUES (1, 1, 2, "Calamity Jen");

REPLACE INTO series (series_id, title, release_date, series_info)
VALUES
    (
        1,
        "IT Crowd",
        CAST(Date("2006-02-03") AS Uint64),
        "The IT Crowd is a British sitcom produced by Channel 4, written by Graham Linehan, produced by Ash Atalla and starring Chris O'Dowd, Richard Ayoade, Katherine Parkinson, and Matt Berry."
    ),
    (
        2,
        "Silicon Valley",
        CAST(Date("2014-04-06") AS Uint64),
        "Silicon Valley is an American comedy television series created by Mike Judge, John Altschuler and Dave Krinsky. The series focuses on five young men who founded a startup company in Silicon Valley."
    )
    ;

REPLACE INTO seasons (series_id, season_id, title, first_aired, last_aired)
VALUES
    (1, 1, "Season 1", CAST(Date("2006-02-03") AS Uint64), CAST(Date("2006-03-03") AS Uint64)),
    (1, 2, "Season 2", CAST(Date("2007-08-24") AS Uint64), CAST(Date("2007-09-28") AS Uint64)),
    (2, 1, "Season 1", CAST(Date("2014-04-06") AS Uint64), CAST(Date("2014-06-01") AS Uint64)),
    (2, 2, "Season 2", CAST(Date("2015-04-12") AS Uint64), CAST(Date("2015-06-14") AS Uint64))
;

REPLACE INTO episodes (series_id, season_id, episode_id, title, air_date)
VALUES
    (1, 1, 1, "Yesterday's Jam", CAST(Date("2006-02-03") AS Uint64)),
    (1, 1, 2, "Calamity Jen", CAST(Date("2006-02-03") AS Uint64)),
    (2, 1, 1, "Minimum Viable Product", CAST(Date("2014-04-06") AS Uint64)),
    (2, 1, 2, "The Cap Table", CAST(Date("2014-04-13") AS Uint64))
;

UPSERT INTO episodes
(
    series_id,
    season_id,
    episode_id,
    title,
    air_date
)
VALUES
(
    2,
    1,
    3,
    "Test Episode",
    CAST(Date("2018-08-27") AS Uint64)
)
;

SELECT
    series_id,
    title AS series_title,
    CAST (release_date AS Date) AS release_date
FROM series;


/* Параметризированный запрос
DECLARE $seriesId AS Uint64;
DECLARE $seasonId AS Uint64;

$seriesId = 1;
$seasonId = 2;

SELECT sa.title AS season_title, sr.title AS series_title
FROM seasons AS sa
INNER JOIN series AS sr
ON sa.series_id = sr.series_id
WHERE sa.series_id = $seriesId AND sa.season_id = $seasonId;
*/
```

# Yandex Database Работа с SDK
https://cloud.yandex.ru/docs/ydb/quickstart/yql-api/ydb-sdk#connect-to-db

cd ~/test/PostgreSQL/6_Cloud_2021_09/34_PG_Yandex_Cloud/demo/
git clone https://github.com/yandex-cloud/ydb-nodejs-sdk.git
cd ydb-nodejs-sdk/examples
npm install
npm run build