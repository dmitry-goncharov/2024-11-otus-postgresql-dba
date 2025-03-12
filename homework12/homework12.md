# Бэкапы

## Цель:

- Применить логический бэкап. Восстановиться из бэкапа;

## Пошаговая инструкция и результаты

### 1. Проверяем, что кластер запущен

myuser: `sudo -u postgres pg_lsclusters`
```
Ver Cluster Port Status Owner    Data directory              Log file
17  main    5432 online postgres /var/lib/postgresql/17/main /var/log/postgresql/postgresql-17-main.log
```

### 2. Заходим под пользователем postgres

myuser: `psql -U postgres`
```
psql (17.2 (Ubuntu 17.2-1.pgdg24.04+1))
Type "help" for help.
```

### 3. Создадим БД hw12db

postgres: `CREATE DATABASE hw12db;`
```
CREATE DATABASE
```

### 4. Заходим в созданную базу данных под пользователем postgres

postgres: `\c hw12db`
```
You are now connected to database "hw12db" as user "postgres".
```

### 5. Создадим схему hw12sch

hw12db: `CREATE SCHEMA hw12sch;`
```
CREATE SCHEMA
```

hw12db: `SET search_path = hw12sch, public;`
```
SET
```

### 6. Создадим таблицу student c 100 авто-сгенерированными записями

hw12db: `CREATE TABLE student AS 
SELECT 
  generate_series(1,100) as id,
  md5(random()::text)::char(10) as name;`
```
SELECT 100
```

hw12db: `\q`

### 7. Создадим каталог для бэкапов

myuser: `sudo mkdir /tmp/backup`

myuser: `sudo chmod 777 /tmp/backup`

### 8. Сделаем логический бэкап используя утилиту COPY

myuser: `psql -U postgres`
```
psql (17.2 (Ubuntu 17.2-1.pgdg24.04+1))
Type "help" for help.
```

postgres: `\c hw12db`
```
You are now connected to database "hw12db" as user "postgres".
```

hw12db: `SET search_path = hw12sch, public;`
```
SET
```

hw12db: `\copy student to '/tmp/backup/st.sql';`
```
COPY 100
```

### 9. Восстановим в 2 таблицу данные из бэкапа.

hw12db: `CREATE TABLE student2(id integer, name text);`
```
CREATE TABLE
```

hw12db: `select * from student2;`
```
 id | name
----+------
(0 rows)
```

hw12db: `\copy student2 from '/tmp/backup/st.sql';`
```
COPY 100
```

hw12db: `select count(*) from student2;`
```
 count
-------
   100
(1 row)
```

hw12db: `\q`

### 10. Используя утилиту pg_dump создадим бэкап в кастомном сжатом формате двух таблиц

myuser: `pg_dump -d hw12db -U postgres -Fc | gzip > /tmp/backup/arh.gz`

### 11. Используя утилиту pg_restore восстановим в новую БД только вторую таблицу

myuser: `psql -U postgres`
```
psql (17.2 (Ubuntu 17.2-1.pgdg24.04+1))
Type "help" for help.
```

postgres: `CREATE DATABASE hw12db2;`
```
CREATE DATABASE
```

postgres: `\c hw12db2`
```
You are now connected to database "hw12db2" as user "postgres".
```

hw12db2: `CREATE SCHEMA hw12sch;`
```
CREATE SCHEMA
```

hw12db2: `\q`

myuser: `gzip -d /tmp/backup/arh.gz`

myuser: `pg_restore -d hw12db2 --schema hw12sch --table student2 -U postgres /tmp/backup/arh`

myuser: `psql -U postgres`
```
psql (17.2 (Ubuntu 17.2-1.pgdg24.04+1))
Type "help" for help.
```

postgres: `\c hw12db2`
```
You are now connected to database "hw12db2" as user "postgres".
```

hw12db2: `\dnS`
```
            List of schemas
        Name        |       Owner
--------------------+-------------------
 hw12sch            | postgres
 information_schema | postgres
 pg_catalog         | postgres
 pg_toast           | postgres
 public             | pg_database_owner
(5 rows)
```

hw12db2: `SET search_path = hw12sch, public;`
```
SET
```

hw12db2: `\d`
```
           List of relations
 Schema  |   Name   | Type  |  Owner
---------+----------+-------+----------
 hw12sch | student2 | table | postgres
(1 row)
```

hw12db2: `select count(*) from student2;`
```
 count
-------
   100
(1 row)
```
