# Data Lake House

## Trino & Apache Iceberg & Metastore on postgres:17
### Хранилища
- postgres:17
- Minio
- Yandex Cloud Storage
- AWS S3
- Google Cloud Storage<hr>

### Необходимо:
- Иметь полновесные регистрации в `Yandex Cloud` & `AWS` & `Google cloud` (либо удалить ненужное из `docker-compose.yml` и файлы в `etc/catalog`)
- Для `Yandex Cloud` & `AWS` в соответствующих консолях завести ключи доступа от сервисных аккаунтов с минимальными ролями типа `storage editor`
- Прописать это в `.env_template` и переимeновать в `.env`
- Для `Google cloud` скачать ключ в виде `json`, и прописать в корень проекта под именем `google.json`

- AWS S3 создать бакет `awstrino`
- Yandex Cloud создать бакет `ycstrino`
- Google Cloud создать бакет `triner`

### docker compose up

В командной строке хоста
### docker exec -it trin trino
*trino>* `show catalogs;`
Выведется
```
  Catalog  
-----------
 aws3      
 google    
 metastore 
 minio     
 pg        
 system    
 tpcds     
 tpch      
 ycs3      
```
где 
- ***aws3 google minio ycs3*** - это S3 хранилища
- ***pg*** - postgres БД
- ***metastore*** - это metastore, используйте как Read Only
- ***tpcds tpch***  см. TPCDS.md TPCH.md

- *trino>* `help;`
```
Supported commands:
QUIT
EXIT
CLEAR
EXPLAIN [ ( option [, ...] ) ] <query>
    options: FORMAT { TEXT | GRAPHVIZ | JSON }
             TYPE { LOGICAL | DISTRIBUTED | VALIDATE | IO }
DESCRIBE <table>
SHOW COLUMNS FROM <table>
SHOW FUNCTIONS
SHOW CATALOGS [LIKE <pattern>]
SHOW SCHEMAS [FROM <catalog>] [LIKE <pattern>]
SHOW TABLES [FROM <schema>] [LIKE <pattern>]
USE [<catalog>.]<schema>
```
<hr>

- Необязательно, но надо - скачать/установить `DBeaver`
- Запустить `DBeaver`, `Создать Соединение`, Выбрать `Trino`, `Connect by host`, `localhost 8080`, Пользователь `admin`, пароль оставить пустым
- DBeaver поначалу подгрузит необходимые библиотеки (Trino)

<hr>

Далее ... 
*удобнее/нагляднее в DBeaver но можно из хоста в CLI Trino* 

В Minio - создаём схему `mini`
`CREATE SCHEMA minio.mini WITH (location = 's3a://tiny/');`

В схеме `mini` каталога `minio` создаём таблицу `client` - копию первых 10 строк таблицы `tpch.tiny.customer`
```
CREATE TABLE minio.mini.client
WITH (
    format = 'ORC',
    location = 's3a://tiny/customer/'
) 
AS SELECT * FROM tpch.tiny.customer limit 10;
```
В Yandex Cloud - создаём схему `maxi`
`CREATE SCHEMA ycs3.maxi WITH (location = 's3a://ycstrino/');`
В схеме `maxi` каталога `ycs3` создаём таблицу `people` - копию таблицы `minio.mini.client`
```
CREATE TABLE minio.mini.people
WITH (
    format = 'ORC',
    location = 's3a://tiny/customer/'
) 
AS SELECT * FROM minio.mini.client;
```

и т.д.

<hr>

https://trino.io/docs/current/object-storage/metastores.html#iceberg-jdbc-catalog:~:text=view%20management.-,JDBC%20catalog,-%23

- The Iceberg JDBC catalog is supported for the Iceberg connector. At a minimum, iceberg.jdbc-catalog.driver-class, iceberg.jdbc-catalog.connection-url, iceberg.jdbc-catalog.default-warehouse-dir, and iceberg.jdbc-catalog.catalog-name must be configured. When using any database besides PostgreSQL, a JDBC driver jar file must be placed in the plugin directory.

The following example shows a minimal catalog configuration using an Iceberg JDBC metadata catalog:
```
connector.name=iceberg
iceberg.catalog.type=jdbc
iceberg.jdbc-catalog.catalog-name=test
iceberg.jdbc-catalog.driver-class=org.postgresql.Driver
iceberg.jdbc-catalog.connection-url=jdbc:postgresql://example.net:5432/database
iceberg.jdbc-catalog.connection-user=admin
iceberg.jdbc-catalog.connection-password=test
iceberg.jdbc-catalog.default-warehouse-dir=s3://bucket
```

CREATE TABLE google.mini.client
    -> WITH (
    ->     format = 'ORC',
    ->     location = 'gs://triner/customer/'
    -> ) 
    -> AS SELECT * FROM tpch.tiny.customer limit 10;





schematool -dbType postgres -info



**Telegram @IBM2702**

# Trino & Minio & Hive Metastore w Postgres

### В Пособии "Trino: The Definitive Guide"
https://datafinder.ru/files/downloads/01/Trino---The-Definitive-Guide-2023.pdf есть ссылка на код  
- https://github.com/bitsondatadev/trino-getting-started/tree/main/hive/trino-minio

С примером развёртывания контейнеров Trino & Minio & Hive Metastore
```
MinIO Example
MinIO is an S3-compatible, lightweight distributed storage system you can use with
Trino and the Hive connector. You can install it on your local workstation and
experiment with it and Trino locally. If you want to explore its use in more detail,
check out the example project from Brian Olsen.
```
где качестве БД metastore используется MariaDB <br>
(из "коробки", кстати, docker compose не запустился, пришлось допиливать)<br>
По мотивам вышеуказанного сделал с Postgres<hr>

### mkdir -p pg_logs && sudo chown -R 999:999 pg_logs


### Создаем папку jars (если не существует)
- mkdir -p ./jars

### Скачиваем aws-java-sdk-bundle-1.11.375.jar
- wget -P ./jars https://repo1.maven.org/maven2/com/amazonaws/aws-java-sdk-bundle/1.11.375/aws-java-sdk-bundle-1.11.375.jar

### Скачиваем hadoop-aws-3.2.4.jar
- wget -P ./jars https://repo1.maven.org/maven2/org/apache/hadoop/hadoop-aws/3.2.4/hadoop-aws-3.2.4.jar

### Запустите контейнеры
- ***docker compose up -d***
### Войдите в режим командной строки контейнера trino-coordinator-container 
- ***docker exec -it trino-coordinator-container trino***
### Выведите список каталогов
- trino> ***show catalogs;***

```
          Catalog           
----------------------------
 minio_catalog              
 postgres_metastore_catalog 
 system                     
 tpcds                      
 tpch                       
(5 rows)
```
Имена каталогов - это имена файлов .properties в ***etc/catalog*** (*Прим. Символ '-' в имени каталога недопустим, пользуем '_'*)
```
-rw-r--r-- 1 naeel naeel 581 Sep 20 08:51 etc/catalog/minio_catalog.properties
-rw-r--r-- 1 naeel naeel 144 Sep 20 13:58 etc/catalog/postgres_metastore_catalog.properties
-rw-r--r-- 1 naeel naeel  44 Sep 11 09:37 etc/catalog/tpcds.properties
-rw-r--r-- 1 naeel naeel  42 Sep 11 09:37 etc/catalog/tpch.properties
```
- Про **tpcds** ***etc/catalog/TPCDS.md***
- Про **tpch** ***etc/catalog/TPCH.md***

### Создание schema в каталоге minio_catalog
```
CREATE SCHEMA google.mini WITH (location = 's3a://triner/');

CREATE TABLE google.mini.client
WITH (
    format = 'ORC',
    location = 's3a://triner/customer/'
) 
AS SELECT * FROM tpch.tiny.customer limit 10;



CREATE SCHEMA minio_catalog.mini WITH (location = 's3a://tiny/');

CREATE SCHEMA ycs3.mini WITH (location = 's3a://ycstrino/');

CREATE TABLE ycs3.mini.client
WITH (
    format = 'ORC',
    location = 's3a://ycstrino/customer/'
) 
AS SELECT * FROM tpch.tiny.customer limit 10;
```


```
`s3a://tiny/` - это бакет в минио, создан контейнером ***createbuckets-service*** при запуске docker compose<br>
Удостоверяемся:
```
trino> show schemas from minio_catalog;
       Schema       
--------------------
 default            
 information_schema 
 mini
(3 rows)
```
### Создадим таблицу client в схеме mini каталога minio_catalog, это копия таблицы tpch.tiny.customer
```
CREATE TABLE minio_catalog.mini.client
WITH (
    format = 'ORC',
    location = 's3a://tiny/customer/'
) 
AS SELECT * FROM tpch.tiny.customer limit 10;
```

CREATE TABLE aws3.minia.client
WITH (
    format = 'ORC',
    location = 's3a://awstrino/customer/'
) 
AS SELECT * FROM tpch.tiny.customer limit 10;



### Откроем второе окно с командной строкой и войдём в контейнер metastore-db
- docker exec -it metastore-db /bin/bash
- Вход в CLI БД
`psql  -U "$POSTGRES_USER" -d metastore`
```
metastore=# SELECT * FROM "DBS";
 DB_ID |         DESC          |      DB_LOCATION_URI      |  NAME   | OWNER_NAME | OWNER_TYPE | CTLG_NAME 
-------+-----------------------+---------------------------+---------+------------+------------+-----------
     1 | Default Hive database | file:/user/hive/warehouse | default | public     | ROLE       | hive
     5 |                       | s3a://tiny/               | mini    | trino      | USER       | hive
(2 rows)
```

Если погружаться глубже в тему - https://github.com/bitsondatadev/trino-getting-started/blob/main/hive/trino-minio/README.md
С учётом иных имён каталогов, схем, таблиц.
И синтаксиса

Например, не
```
SELECT
 DB_ID,
 DB_LOCATION_URI,
 NAME, 
 OWNER_NAME,
 OWNER_TYPE,
 CTLG_NAME
FROM metastore_db.DBS;
```
но
```
SELECT
 "DB_ID",
 "DB_LOCATION_URI",
 "NAME", 
 "OWNER_NAME",
 "OWNER_TYPE",
 "CTLG_NAME"
FROM "DBS";
```


docker exec -it trino sh -c 'ls -l /usr/lib/trino/plugin/hive/tr*.jar'

docker exec -it trino sh -c 'jar tf /usr/lib/trino/plugin/hive/lib/'

jar tf /path/to/hadoop-aws-3.3.6.jar | grep S3AFileSystem

/usr/lib/trino/plugin/hive/lib/

unzip trino-hive-476.zip -d trino-hive

# Создайте директорию для плагина Hive
mkdir -p trino-hive-plugin

# Переместите основной JAR плагина
mv trino-hive/trino-hive-476/io.trino_trino-hive-476.jar trino-hive-plugin/trino-hive-476.jar

# Создайте директорию lib и переместите все зависимости
mkdir trino-hive-plugin/lib
mv trino-hive/trino-hive-476/*.jar trino-hive-plugin/lib/

# Но оставьте основной JAR плагина на своем месте
mv trino-hive-plugin/lib/io.trino_trino-hive-476.jar trino-hive-plugin/


