# Настройка autovacuum с учетом особенностей производительности

## Цель:

- Запустить нагрузочный тест pgbench.
- Настроить параметры autovacuum.
- Проверить работу autovacuum.

## Пошаговая инструкция и результаты

### 1. Создать экземпляр ВМ с 2 ядрами и 4 Гб ОЗУ и SSD 10GB

```
Платформа - Intel Ice Lake
Гарантированная доля vCPU - 100%
vCPU - 2
RAM - 4 ГБ
SSD - 10 ГБ
```

### 2. Устанавливаем на него PostgreSQL 17 с дефолтными настройками

myuser: `sudo apt update && sudo apt upgrade -y && sudo sh -c 'echo "deb http://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list' && wget --quiet -O - https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo apt-key add - && sudo apt-get update && sudo apt-get -y install postgresql && sudo apt install unzip && sudo apt -y install mc`

### 3. Проверяем, что кластер запущен

myuser: `sudo -u postgres pg_lsclusters`
```
Ver Cluster Port Status Owner    Data directory              Log file
17  main    5432 online postgres /var/lib/postgresql/17/main /var/log/postgresql/postgresql-17-main.log
```

### 4. Инициализируем утилиту pgbench

myuser: `pgbench -i postgres -U postgres`
```
creating tables...
generating data (client-side)...
vacuuming...
creating primary keys...
done in 1.03 s (drop tables 0.00 s, create tables 0.01 s, client-side generate 0.09 s, vacuum 0.04 s, primary keys 0.89 s).
```

### 5. Протестируем с параметрами по умолчанию

Конфигурация:
```
max_connections = 100
# Memory Configuration
shared_buffers = 128MB
effective_cache_size = 4GB
work_mem = 4MB
maintenance_work_mem = 64MB
# Checkpoint Related Configuration
min_wal_size = 80MB
max_wal_size = 1GB
wal_buffers = 4MB
```

- база данных postgres
- пользователь postgres
- количество клиентов 8
- показываем отчет о ходе выполнения каждые 6 сек
- продолжительность теста 60 сек
- конфигурацию тестируем 5 раз

myuser: `pgbench -d postgres -c 8 -P 6 -T 60 -U postgres`
```
tps = 733.623074 (without initial connection time)
tps = 776.646170 (without initial connection time)
tps = 720.638349 (without initial connection time)
tps = 727.667981 (without initial connection time)
tps = 755.318951 (without initial connection time)
```

### 6. Применить параметры настройки PostgreSQL из прикрепленного к материалам занятия файла и протестируем заново

Конфигурация:
```
max_connections = 40
shared_buffers = 1GB
effective_cache_size = 3GB
maintenance_work_mem = 512MB
checkpoint_completion_target = 0.9
wal_buffers = 16MB
default_statistics_target = 500
random_page_cost = 4
effective_io_concurrency = 2
work_mem = 6553kB
min_wal_size = 4GB
max_wal_size = 16GB
```

- база данных postgres
- пользователь postgres
- количество клиентов 8
- показываем отчет о ходе выполнения каждые 6 сек
- продолжительность теста 60 сек
- конфигурацию тестируем 5 раз

myuser: `pgbench -d postgres -c 8 -P 6 -T 60 -U postgres`
```
tps = 761.784697 (without initial connection time)
tps = 770.091137 (without initial connection time)
tps = 753.128280 (without initial connection time)
tps = 763.370737 (without initial connection time)
tps = 778.683396 (without initial connection time)
```

По результатам тестов наблюдаем незначительные изменения, вероятно, связанные с изменением shared_buffers.

### 7. Создать таблицу с текстовым полем и заполнить случайными или сгенерированными данным в размере 1млн строк

myuser: `psql -U postgres`
```
psql (17.2 (Ubuntu 17.2-1.pgdg24.04+1))
Type "help" for help.
```
postgres: `CREATE TABLE student(id serial, fio char(100));`
```
CREATE TABLE
```
postgres: `INSERT INTO student(fio) SELECT 'noname' FROM generate_series(1, 1000000);`
```
INSERT 0 1000000
```

### 8. Посмотрим размер файла с таблицей

postgres: `SELECT pg_size_pretty(pg_total_relation_size('student'));`
```
 pg_size_pretty
----------------
 135 MB
(1 row)
```

### 9. 5 раз обновим все строчки и добавим к каждой строчке любой символ

postgres: `UPDATE student SET fio = 'noname1';`
```
UPDATE 1000000
```
postgres: `UPDATE student SET fio = 'noname2';`
```
UPDATE 1000000
```
postgres: `UPDATE student SET fio = 'noname3';`
```
UPDATE 1000000
```
postgres: `UPDATE student SET fio = 'noname4';`
```
UPDATE 1000000
```
postgres: `UPDATE student SET fio = 'noname5';`
```
UPDATE 1000000
```

### 10. Посмотрим количество мертвых строчек в таблице и когда последний раз проходил autovacuum

postgres: `SELECT relname, n_live_tup, n_dead_tup, trunc(100*n_dead_tup/(n_live_tup+1))::float "ratio%", last_autovacuum FROM pg_stat_user_tables WHERE relname = 'student';`
```
relname | n_live_tup | n_dead_tup | ratio% |        last_autovacuum
---------+------------+------------+--------+-------------------------------
student |    1000000 |          0 |      0 | 2025-01-18 20:03:31.058633+00
(1 row)
```

В результате видим, что autovacuum уже проходил.

### 11. 5 раз обновим все строчки и добавим к каждой строчке любой символ

postgres: `UPDATE student SET fio = 'noname11';`
```
UPDATE 1000000
```
postgres: `UPDATE student SET fio = 'noname22';`
```
UPDATE 1000000
```
postgres: `UPDATE student SET fio = 'noname33';`
```
UPDATE 1000000
```
postgres: `UPDATE student SET fio = 'noname44';`
```
UPDATE 1000000
```
postgres: `UPDATE student SET fio = 'noname55';`
```
UPDATE 1000000
```

### 12. Посмотрим размер файла с таблицей

postgres: `SELECT pg_size_pretty(pg_total_relation_size('student'));`
```
 pg_size_pretty
----------------
 539 MB
(1 row)
```

### 13. Отключим autovacuum на таблице

postgres: `ALTER TABLE student SET (autovacuum_enabled = off);`
```
ALTER TABLE
```

### 14. 10 раз обновим все строчки и добавим к каждой строчке любой символ

postgres: `UPDATE student SET fio = 'noname11';`
```
UPDATE 1000000
```
postgres: `UPDATE student SET fio = 'noname222';`
```
UPDATE 1000000
```
postgres: `UPDATE student SET fio = 'noname3333';`
```
UPDATE 1000000
```
postgres: `UPDATE student SET fio = 'noname44444';`
```
UPDATE 1000000
```
postgres: `UPDATE student SET fio = 'noname555555';`
```
UPDATE 1000000
```
postgres: `UPDATE student SET fio = 'noname6666666';`
```
UPDATE 1000000
```
postgres: `UPDATE student SET fio = 'noname77777777';`
```
UPDATE 1000000
```
postgres: `UPDATE student SET fio = 'noname888888888';`
```
UPDATE 1000000
```
postgres: `UPDATE student SET fio = 'noname9999999999';`
```
UPDATE 1000000
```
postgres: `UPDATE student SET fio = 'noname00000000000';`
```
UPDATE 1000000
```

### 15. Посмотрим размер файла с таблицей

postgres: `SELECT pg_size_pretty(pg_total_relation_size('student'));`
```
 pg_size_pretty
----------------
 1482 MB
(1 row)
```

Размер увеличился, так как неиспользуемые строки занимают место.

### 16. Включим autovacuum на таблице

postgres: `ALTER TABLE student SET (autovacuum_enabled = on);`
```
ALTER TABLE
```
