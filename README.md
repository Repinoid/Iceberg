**[Telegram IBM2702]  (https://t.me/IBM2702)**

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
*trino>* `show catalogs;`<br>
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
- `CREATE SCHEMA minio.mini WITH (location = 's3a://tiny/');`

- В схеме `mini` каталога `minio` создаём таблицу `client` - копию первых 10 строк таблицы `tpch.tiny.customer`
```
CREATE TABLE minio.mini.client
WITH (
    format = 'ORC',
    location = 's3a://tiny/customer/'
) 
AS SELECT * FROM tpch.tiny.customer limit 10;
```
В Yandex Cloud - создаём схему `maxi`
- `CREATE SCHEMA ycs3.maxi WITH (location = 's3a://ycstrino/');`
- В схеме `maxi` каталога `ycs3` создаём таблицу `people` - копию таблицы `minio.mini.client`
```
CREATE TABLE minio.mini.people
WITH (
    format = 'ORC',
    location = 's3a://tiny/customer/'
) 
AS SELECT * FROM minio.mini.client;
```

и т.д.
Ps Формат для google cloud `gs://triner`

<hr>
