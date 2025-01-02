# Работа с базами данных, пользователями и правами

## Цель:

- Создание новой базы данных, схемы и таблицы
- Создание роли для чтения данных из созданной схемы созданной базы данных
- Создание роли для чтения и записи из созданной схемы созданной базы данных

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

### 3. Создаем новую базу данных testdb

postgres: `CREATE DATABASE testdb;`
```
CREATE DATABASE
```

### 4. Заходим в созданную базу данных под пользователем postgres

postgres: `\c testdb`
```
You are now connected to database "testdb" as user "postgres".
```

### 5. Создаем новую схему testnm

testdb: `CREATE SCHEMA testnm;`
```
CREATE SCHEMA
```

### 6. Создаем новую таблицу t1 с одной колонкой c1 типа integer

testdb: `CREATE TABLE t1(c1 integer);`
```
CREATE TABLE
```

### 7. Вставляем строку со значением c1=1

testdb: `INSERT INTO t1 values(1);`
```
INSERT 0 1
```

### 8. Создаем новую роль readonly

testdb: `CREATE role readonly;`
```
CREATE ROLE
```

### 9. Задаем новой роли право на подключение к базе данных testdb

testdb: `grant connect on DATABASE testdb TO readonly;`
```
GRANT
```

### 10. Задаем новой роли право на использование схемы testnm

testdb: `grant usage on SCHEMA testnm to readonly;`
```
GRANT
```

### 11. Задаем новой роли право на select для всех таблиц схемы testnm

testdb: `grant SELECT on all TABLES in SCHEMA testnm TO readonly;`
```
GRANT
```

### 12. Создаем пользователя testread с паролем test123

testdb: `CREATE USER testread with password 'test123';`
```
CREATE ROLE
```

### 13. Задаем роль readonly пользователю testread

testdb: `grant ``` TO testread;`
```
GRANT ROLE
```

### 14. Заходим под пользователем testread в базу данных testdb

testdb: `\c testdb testread`
```
You are now connected to database "testdb" as user "testread".
```

### 15. Сделаем select * from t1;

testdb: `SELECT * FROM t1;`
```
ERROR:  permission denied for table t1
```

Нет прав на выборку из таблицы t1 для роли readonly, так как таблица t1 находится в схеме public.

testdb: `\dt`
```
List of relations
Schema | Name | Type  |  Owner
--------+------+-------+----------
public | t1   | table | postgres
(1 row)
```

Таблица t1 была создана в public, так как search_path содержит public.

testdb: `SHOW search_path;`
```
search_path
-----------------
"$user", public
(1 row)
```

### 16. Вернемся в базу данных testdb под пользователем postgres

testdb: `\c testdb postgres`
```
You are now connected to database "testdb" as user "postgres".
```

### 17. Удаляем таблицу t1

testdb: `DROP TABLE t1;`
```
DROP TABLE
```

### 18. Создаем ее заново, но уже с явным указанием имени схемы testnm

testdb: `CREATE TABLE testnm.t1(c1 integer);`
```
CREATE TABLE
```

### 19. Вставляем строку со значением c1=1

testdb: `INSERT INTO testnm.t1 values(1);`
```
INSERT 0 1
```

### 19. Заходим под пользователем testread в базу данных testdb

testdb: `\c testdb testread`
```
You are now connected to database "testdb" as user "testread".
```

### 20. Сделаем select * from testnm.t1;

testdb: `SELECT * FROM testnm.t1;`
```
ERROR:  permission denied for table t1
```

Нет прав на выборку из таблицы t1 для роли readonly, так как когда давали права на выборку таблицы еще не было.

### 21. Задаем роли readonly право на выборку для всех таблиц схемы testnm

testdb: `\c testdb postgres;`
```
You are now connected to database "testdb" as user "postgres".
```

testdb: `ALTER default privileges in SCHEMA testnm grant SELECT on TABLES to readonly;`
```
ALTER DEFAULT PRIVILEGES
```

testdb: `\c testdb testread`
```
You are now connected to database "testdb" as user "testread".
```

### 22. Сделаем select * from testnm.t1;

testdb: `SELECT * FROM testnm.t1;`
```
ERROR:  permission denied for table t1
```

Нет прав на выборку из таблицы t1 для роли readonly, так как ALTER default privileges будет действовать для новых таблиц.

### 23. Задаем роли readonly право на выборку для всех таблиц схемы testnm

testdb: `\c testdb postgres;`
```
You are now connected to database "testdb" as user "postgres".
```

testdb: `grant SELECT on all TABLES in SCHEMA testnm TO readonly;`
```
GRANT
```

testdb: `\c testdb testread`
```
You are now connected to database "testdb" as user "testread".
```

### 24. Сделаем select * from testnm.t1;

testdb: `SELECT * FROM testnm.t1;`
```
 c1
----
  1
(1 row)
```

### 25. Попробуем выполнить команду создания таблицы и вставки данных в нее

testdb: `create table t2(c1 integer); insert into t2 values(2);`
```
ERROR:  permission denied for schema public
LINE 1: create table t2(c1 integer);
                     ^
ERROR:  relation "t2" does not exist
LINE 1: insert into t2 values(2);
```

Нет прав на создание таблиц в схеме public, так как используется PostgreSQL версии 17. Это право было удалено в версии 15.

`PostgreSQL 15 release notes: Remove PUBLIC creation permission on the public schema.` Для более подробной информации смотри https://www.postgresql.org/docs/15/release-15.html
