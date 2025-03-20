# Репликация

## Цель:

- Реализовать свой миникластер на 3 ВМ;

## Пошаговая инструкция и результаты

### 1. На 1 ВМ создаем таблицы test1 для записи, test2 для запросов на чтение

Применяем конфигурацию для логической репликации
```
wal_level = logical
```

А также добавляем конфигурацию для доступности в listen_addresses и возможность подключения от ВМ 2 и 3 в pg_hba.

#### 1.1 Проверяем, что кластер запущен

myuser: `sudo -u postgres pg_lsclusters`
```
Ver Cluster Port Status Owner    Data directory              Log file
17  main    5432 online postgres /var/lib/postgresql/17/main /var/log/postgresql/postgresql-17-main.log
```

#### 1.2 Заходим под пользователем postgres

myuser: `psql -U postgres`
```
psql (17.2 (Ubuntu 17.2-1.pgdg24.04+1))
Type "help" for help.
```

#### 1.3 Создадим БД stud

postgres: `CREATE DATABASE stud;`
```
CREATE DATABASE
```

#### 1.4 Заходим в созданную базу данных под пользователем postgres

postgres: `\c stud`
```
You are now connected to database "stud" as user "postgres".
```

#### 1.5 Создадим таблицу test1 для записи

stud: `CREATE TABLE test1 AS 
SELECT 
  generate_series(1,10) as id,
  md5(random()::text)::char(10) as name;`
```
SELECT 10
```

#### 1.6 Создадим таблицу test2 для чтения

stud: `CREATE TABLE test2(id integer, name text);`
```
CREATE TABLE
```

#### 1.7. Создадим публикацию таблицы test1

stud: `CREATE PUBLICATION test1_pub FOR TABLE test1;`
```
CREATE PUBLICATION
```

stud: `\dRp+`
```
                           Publication test1_pub
  Owner   | All tables | Inserts | Updates | Deletes | Truncates | Via root
----------+------------+---------+---------+---------+-----------+----------
 postgres | f          | t       | t       | t       | t         | f
Tables:
    "public.test1"
```

### 2. На 2 ВМ создаем таблицы test1 для запросов на чтение, test2 для записи

Применяем конфигурацию для логической репликации
```
wal_level = logical
```

А также добавляем конфигурацию для доступности в listen_addresses и возможность подключения от ВМ 1 и 3 в pg_hba.

#### 2.1 Проверяем, что кластер запущен

myuser: `sudo -u postgres pg_lsclusters`
```
Ver Cluster Port Status Owner    Data directory              Log file
17  main    5432 online postgres /var/lib/postgresql/17/main /var/log/postgresql/postgresql-17-main.log
```

#### 2.2 Заходим под пользователем postgres

myuser: `psql -U postgres`
```
psql (17.2 (Ubuntu 17.2-1.pgdg24.04+1))
Type "help" for help.
```

#### 2.3 Создадим БД stud

postgres: `CREATE DATABASE stud;`
```
CREATE DATABASE
```

#### 2.4 Заходим в созданную базу данных под пользователем postgres

postgres: `\c stud`
```
You are now connected to database "stud" as user "postgres".
```

#### 2.5 Создадим таблицу test для чтения

stud: `CREATE TABLE test1(id integer, name text);`
```
CREATE TABLE
```

#### 2.6 Создадим таблицу test2 для записи

stud: `CREATE TABLE test2 AS 
SELECT 
  generate_series(1,10) as id,
  md5(random()::text)::char(10) as name;`
```
SELECT 10
```

#### 2.7. Создадим публикацию таблицы test2

stud: `CREATE PUBLICATION test2_pub FOR TABLE test2;`
```
CREATE PUBLICATION
```

stud: `\dRp+`
```
                           Publication test2_pub
  Owner   | All tables | Inserts | Updates | Deletes | Truncates | Via root
----------+------------+---------+---------+---------+-----------+----------
 postgres | f          | t       | t       | t       | t         | f
Tables:
    "public.test2"
```

### 3. На 3 ВМ создаем таблицы test1 и test2 для запросов на чтение

Применяем конфигурацию для логической репликации
```
wal_level = logical
```

А также добавляем конфигурацию для доступности в listen_addresses и возможность подключения от ВМ 1 и 2 в pg_hba.

#### 3.1 Проверяем, что кластер запущен

myuser: `sudo -u postgres pg_lsclusters`
```
Ver Cluster Port Status Owner    Data directory              Log file
17  main    5432 online postgres /var/lib/postgresql/17/main /var/log/postgresql/postgresql-17-main.log
```

#### 3.2 Заходим под пользователем postgres

myuser: `psql -U postgres`
```
psql (17.2 (Ubuntu 17.2-1.pgdg24.04+1))
Type "help" for help.
```

#### 3.3 Создадим БД stud

postgres: `CREATE DATABASE stud;`
```
CREATE DATABASE
```

#### 3.4 Заходим в созданную базу данных под пользователем postgres

postgres: `\c stud`
```
You are now connected to database "stud" as user "postgres".
```

#### 3.5 Создадим таблицу test для чтения

stud: `CREATE TABLE test1(id integer, name text);`
```
CREATE TABLE
```

#### 3.6 Создадим таблицу test2 для записи

stud: `CREATE TABLE test2(id integer, name text);`
```
CREATE TABLE
```

### 4. На 1 ВМ подпишемся на публикацию таблицы test2 с ВМ 2

stud: `CREATE SUBSCRIPTION test2_sub
CONNECTION 'host=158.160.146.226 port=5432 user=postgres password=12345 dbname=stud'
PUBLICATION test2_pub WITH (copy_data = true);`
```
NOTICE:  created replication slot "test2_sub" on publisher
CREATE SUBSCRIPTION
```

stud: `\dRs`
```
            List of subscriptions
   Name    |  Owner   | Enabled | Publication
-----------+----------+---------+-------------
 test2_sub | postgres | t       | {test2_pub}
(1 row)
```

### 5. На 2 ВМ подпишемся на публикацию таблицы test1 с ВМ 1

stud: `CREATE SUBSCRIPTION test1_sub
CONNECTION 'host=158.160.142.49 port=5432 user=postgres password=12345 dbname=stud'
PUBLICATION test1_pub WITH (copy_data = true);`
```
NOTICE:  created replication slot "test1_sub" on publisher
CREATE SUBSCRIPTION
```

stud: `\dRs`
```
            List of subscriptions
   Name    |  Owner   | Enabled | Publication
-----------+----------+---------+-------------
 test1_sub | postgres | t       | {test1_pub}
(1 row)
```

### 6. На 3 ВМ подпишемся на публикацию таблицы test1 с ВМ 1 и на публикацию таблицы test2 с ВМ 2

stud: `CREATE SUBSCRIPTION test1_sub3
CONNECTION 'host=158.160.142.49 port=5432 user=postgres password=12345 dbname=stud'
PUBLICATION test1_pub WITH (copy_data = true);`
```
NOTICE:  created replication slot "test1_sub3" on publisher
CREATE SUBSCRIPTION
```

stud: `CREATE SUBSCRIPTION test2_sub3
CONNECTION 'host=158.160.146.226 port=5432 user=postgres password=12345 dbname=stud'
PUBLICATION test2_pub WITH (copy_data = true);`
```
NOTICE:  created replication slot "test2_sub3" on publisher
CREATE SUBSCRIPTION
```

stud: `\dRs`
```
             List of subscriptions
    Name    |  Owner   | Enabled | Publication
------------+----------+---------+-------------
 test1_sub3 | postgres | t       | {test1_pub}
 test2_sub3 | postgres | t       | {test2_pub}
(2 rows)
```

### 7. Проверяем

#### 7.1 На 1 ВМ

stud: `select count(*) from test2;`
```
 count
-------
    10
(1 row)
```

#### 7.2 На 2 ВМ

stud: `select count(*) from test1;`
```
 count
-------
    10
(1 row)
```

#### 7.3 На 3 ВМ

stud: `select count(*) from test1;`
```
 count
-------
    10
(1 row)
```

stud: `select count(*) from test2;`
```
 count
-------
    10
(1 row)
```

#### 7.4 Делаем вставку на 1 ВМ в таблицу test1

stud: `INSERT INTO test1 values(11, 'qqq');`
```
INSERT 0 1
```

#### 7.5 Делаем вставку на 2 ВМ в таблицу test2

stud: `INSERT INTO test2 values(11, 'www');`
```
INSERT 0 1
```

#### 7.6 Проверяем 3 ВМ

stud: `select count(*) from test1;`
```
 count
-------
    11
(1 row)
```

stud: `select count(*) from test2;`
```
 count
-------
    11
(1 row)
```