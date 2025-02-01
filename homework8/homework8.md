# Работа с журналами

## Цель:

- Уметь работать с журналами и контрольными точками.
- Уметь настраивать параметры журналов.

## Пошаговая инструкция и результаты

### 1. Проверяем, что кластер запущен

myuser: `sudo -u postgres pg_lsclusters`
```
Ver Cluster Port Status Owner    Data directory              Log file
17  main    5432 online postgres /var/lib/postgresql/17/main /var/log/postgresql/postgresql-17-main.log
```

### 2. Настраиваем выполнение контрольной точки раз в 30 секунд

Добавим и применим параметры конфигурации:
```
checkpoint_timeout = 30s
```

### 3. 10 минут c помощью утилиты pgbench подаем нагрузку

#### 3.1. Инициализируем утилиту pgbench

myuser: `pgbench -i postgres -U postgres`
```
creating tables...
generating data (client-side)...
vacuuming...
creating primary keys...
done in 0.30 s (drop tables 0.06 s, create tables 0.02 s, client-side generate 0.10 s, vacuum 0.07 s, primary keys 0.05 s).
```

#### 3.2 Запускаем тест

- база данных postgres
- пользователь postgres
- продолжительность теста 600 сек

myuser: `pgbench -d postgres -T 600 -U postgres`
```
tps = 567.776841 (without initial connection time)
```

### 4. Измерим, какой объем журнальных файлов был сгенерирован за это время. Оценим, какой объем приходится в среднем на одну контрольную точку.

Посмотрим лог.

myuser: `tail -n 60 /var/log/postgresql/postgresql-17-main.log`
```
2025-01-25 14:19:09.812 UTC [1726] LOG:  checkpoint starting: time
2025-01-25 14:19:36.133 UTC [1726] LOG:  checkpoint complete: wrote 1699 buffers (10.4%); 0 WAL file(s) added, 0 removed, 0 recycled; write=26.273 s, sync=0.037 s, total=26.322 s; sync files=47, longest=0.019 s, average=0.001 s; distance=12863 kB, estimate=12863 kB; lsn=1/71138F78, redo lsn=1/6FF45C40
2025-01-25 14:19:39.137 UTC [1726] LOG:  checkpoint starting: time
2025-01-25 14:20:06.032 UTC [1726] LOG:  checkpoint complete: wrote 1960 buffers (12.0%); 0 WAL file(s) added, 2 removed, 0 recycled; write=26.859 s, sync=0.005 s, total=26.896 s; sync files=14, longest=0.005 s, average=0.001 s; distance=19112 kB, estimate=19112 kB; lsn=1/72497150, redo lsn=1/711EFC98
2025-01-25 14:20:09.035 UTC [1726] LOG:  checkpoint starting: time
2025-01-25 14:20:36.045 UTC [1726] LOG:  checkpoint complete: wrote 1778 buffers (10.9%); 0 WAL file(s) added, 1 removed, 0 recycled; write=26.960 s, sync=0.014 s, total=27.010 s; sync files=7, longest=0.010 s, average=0.002 s; distance=19733 kB, estimate=19733 kB; lsn=1/737FD350, redo lsn=1/725350E0
2025-01-25 14:20:39.048 UTC [1726] LOG:  checkpoint starting: time
2025-01-25 14:21:06.056 UTC [1726] LOG:  checkpoint complete: wrote 1976 buffers (12.1%); 0 WAL file(s) added, 1 removed, 0 recycled; write=26.961 s, sync=0.025 s, total=27.008 s; sync files=15, longest=0.021 s, average=0.002 s; distance=19987 kB, estimate=19987 kB; lsn=1/74DBE970, redo lsn=1/738B9F10
2025-01-25 14:21:09.059 UTC [1726] LOG:  checkpoint starting: time
2025-01-25 14:21:36.066 UTC [1726] LOG:  checkpoint complete: wrote 1967 buffers (12.0%); 0 WAL file(s) added, 1 removed, 0 recycled; write=26.960 s, sync=0.011 s, total=27.007 s; sync files=7, longest=0.006 s, average=0.002 s; distance=22214 kB, estimate=22214 kB; lsn=1/76337EE0, redo lsn=1/74E6B8A8
2025-01-25 14:21:39.069 UTC [1726] LOG:  checkpoint starting: time
2025-01-25 14:22:06.083 UTC [1726] LOG:  checkpoint complete: wrote 2069 buffers (12.6%); 0 WAL file(s) added, 2 removed, 0 recycled; write=26.959 s, sync=0.011 s, total=27.015 s; sync files=12, longest=0.009 s, average=0.001 s; distance=21998 kB, estimate=22192 kB; lsn=1/778958D8, redo lsn=1/763E71B0
2025-01-25 14:22:09.086 UTC [1726] LOG:  checkpoint starting: time
2025-01-25 14:22:36.069 UTC [1726] LOG:  checkpoint complete: wrote 1955 buffers (11.9%); 0 WAL file(s) added, 1 removed, 0 recycled; write=26.957 s, sync=0.011 s, total=26.983 s; sync files=6, longest=0.008 s, average=0.002 s; distance=21862 kB, estimate=22159 kB; lsn=1/78C82D10, redo lsn=1/77940CF8
2025-01-25 14:22:39.072 UTC [1726] LOG:  checkpoint starting: time
2025-01-25 14:23:06.065 UTC [1726] LOG:  checkpoint complete: wrote 2021 buffers (12.3%); 0 WAL file(s) added, 1 removed, 0 recycled; write=26.961 s, sync=0.007 s, total=26.993 s; sync files=13, longest=0.004 s, average=0.001 s; distance=20383 kB, estimate=21982 kB; lsn=1/7A01A520, redo lsn=1/78D28BE0
2025-01-25 14:23:09.068 UTC [1726] LOG:  checkpoint starting: time
2025-01-25 14:23:36.064 UTC [1726] LOG:  checkpoint complete: wrote 1901 buffers (11.6%); 0 WAL file(s) added, 2 removed, 0 recycled; write=26.958 s, sync=0.012 s, total=26.997 s; sync files=6, longest=0.010 s, average=0.002 s; distance=20063 kB, estimate=21790 kB; lsn=1/7B42C6A0, redo lsn=1/7A0C0BA8
2025-01-25 14:23:39.067 UTC [1726] LOG:  checkpoint starting: time
2025-01-25 14:24:06.082 UTC [1726] LOG:  checkpoint complete: wrote 2017 buffers (12.3%); 0 WAL file(s) added, 1 removed, 0 recycled; write=26.971 s, sync=0.006 s, total=27.015 s; sync files=12, longest=0.004 s, average=0.001 s; distance=20602 kB, estimate=21671 kB; lsn=1/7C8236A8, redo lsn=1/7B4DF778
2025-01-25 14:24:09.085 UTC [1726] LOG:  checkpoint starting: time
2025-01-25 14:24:36.027 UTC [1726] LOG:  checkpoint complete: wrote 1866 buffers (11.4%); 0 WAL file(s) added, 1 removed, 0 recycled; write=26.884 s, sync=0.007 s, total=26.942 s; sync files=6, longest=0.004 s, average=0.002 s; distance=20388 kB, estimate=21543 kB; lsn=1/7DBCF2C0, redo lsn=1/7C8C8860
2025-01-25 14:24:39.030 UTC [1726] LOG:  checkpoint starting: time
2025-01-25 14:25:06.027 UTC [1726] LOG:  checkpoint complete: wrote 1982 buffers (12.1%); 0 WAL file(s) added, 1 removed, 0 recycled; write=26.961 s, sync=0.011 s, total=26.997 s; sync files=13, longest=0.007 s, average=0.001 s; distance=20085 kB, estimate=21397 kB; lsn=1/7EECDE28, redo lsn=1/7DC65D78
2025-01-25 14:25:09.030 UTC [1726] LOG:  checkpoint starting: time
2025-01-25 14:25:36.026 UTC [1726] LOG:  checkpoint complete: wrote 1840 buffers (11.2%); 0 WAL file(s) added, 1 removed, 0 recycled; write=26.960 s, sync=0.007 s, total=26.996 s; sync files=6, longest=0.005 s, average=0.002 s; distance=19459 kB, estimate=21203 kB; lsn=1/802C3218, redo lsn=1/7EF66A88
2025-01-25 14:25:39.029 UTC [1726] LOG:  checkpoint starting: time
2025-01-25 14:26:06.039 UTC [1726] LOG:  checkpoint complete: wrote 1968 buffers (12.0%); 0 WAL file(s) added, 2 removed, 0 recycled; write=26.965 s, sync=0.017 s, total=27.011 s; sync files=10, longest=0.009 s, average=0.002 s; distance=20537 kB, estimate=21137 kB; lsn=1/816315E8, redo lsn=1/803751C8
2025-01-25 14:26:09.042 UTC [1726] LOG:  checkpoint starting: time
2025-01-25 14:26:36.036 UTC [1726] LOG:  checkpoint complete: wrote 1830 buffers (11.2%); 0 WAL file(s) added, 1 removed, 0 recycled; write=26.961 s, sync=0.008 s, total=26.995 s; sync files=6, longest=0.003 s, average=0.002 s; distance=19779 kB, estimate=21001 kB; lsn=1/829EAE00, redo lsn=1/816C5FC8
2025-01-25 14:26:39.039 UTC [1726] LOG:  checkpoint starting: time
2025-01-25 14:27:06.027 UTC [1726] LOG:  checkpoint complete: wrote 2135 buffers (13.0%); 0 WAL file(s) added, 1 removed, 0 recycled; write=26.963 s, sync=0.010 s, total=26.988 s; sync files=12, longest=0.006 s, average=0.001 s; distance=20250 kB, estimate=20926 kB; lsn=1/83DD23E0, redo lsn=1/82A8C7C8
2025-01-25 14:27:09.030 UTC [1726] LOG:  checkpoint starting: time
2025-01-25 14:27:36.145 UTC [1726] LOG:  checkpoint complete: wrote 1837 buffers (11.2%); 0 WAL file(s) added, 1 removed, 0 recycled; write=27.057 s, sync=0.024 s, total=27.116 s; sync files=6, longest=0.014 s, average=0.004 s; distance=20444 kB, estimate=20878 kB; lsn=1/85194138, redo lsn=1/83E83B18
2025-01-25 14:27:39.148 UTC [1726] LOG:  checkpoint starting: time
2025-01-25 14:28:06.038 UTC [1726] LOG:  checkpoint complete: wrote 1903 buffers (11.6%); 0 WAL file(s) added, 2 removed, 0 recycled; write=26.861 s, sync=0.008 s, total=26.890 s; sync files=10, longest=0.008 s, average=0.001 s; distance=20018 kB, estimate=20792 kB; lsn=1/864F7308, redo lsn=1/85210570
2025-01-25 14:28:09.041 UTC [1726] LOG:  checkpoint starting: time
2025-01-25 14:28:36.080 UTC [1726] LOG:  checkpoint complete: wrote 1814 buffers (11.1%); 0 WAL file(s) added, 1 removed, 0 recycled; write=26.977 s, sync=0.021 s, total=27.040 s; sync files=6, longest=0.014 s, average=0.004 s; distance=19994 kB, estimate=20712 kB; lsn=1/878532B8, redo lsn=1/86596E20
2025-01-25 14:28:39.083 UTC [1726] LOG:  checkpoint starting: time
2025-01-25 14:29:06.091 UTC [1726] LOG:  checkpoint complete: wrote 2138 buffers (13.0%); 0 WAL file(s) added, 1 removed, 0 recycled; write=26.962 s, sync=0.016 s, total=27.009 s; sync files=12, longest=0.010 s, average=0.002 s; distance=19870 kB, estimate=20628 kB; lsn=1/88BCBDC0, redo lsn=1/878FE8E8
2025-01-25 14:29:09.094 UTC [1726] LOG:  checkpoint starting: time
2025-01-25 14:29:36.108 UTC [1726] LOG:  checkpoint complete: wrote 1818 buffers (11.1%); 0 WAL file(s) added, 1 removed, 0 recycled; write=26.965 s, sync=0.017 s, total=27.015 s; sync files=6, longest=0.008 s, average=0.003 s; distance=19991 kB, estimate=20564 kB; lsn=1/89C85E58, redo lsn=1/88C84650
2025-01-25 14:30:39.171 UTC [1726] LOG:  checkpoint starting: time
2025-01-25 14:31:06.045 UTC [1726] LOG:  checkpoint complete: wrote 852 buffers (5.2%); 0 WAL file(s) added, 1 removed, 0 recycled; write=26.844 s, sync=0.013 s, total=26.875 s; sync files=12, longest=0.013 s, average=0.002 s; distance=16408 kB, estimate=20148 kB; lsn=1/89C8A820, redo lsn=1/89C8A790
2025-01-25 14:32:18.856 UTC [1725] LOG:  received fast shutdown request
2025-01-25 14:32:18.859 UTC [1725] LOG:  aborting any active transactions
2025-01-25 14:32:18.865 UTC [1725] LOG:  background worker "logical replication launcher" (PID 1731) exited with exit code 1
2025-01-25 14:32:18.865 UTC [1726] LOG:  shutting down
2025-01-25 14:32:18.867 UTC [1726] LOG:  checkpoint starting: shutdown immediate
2025-01-25 14:32:18.875 UTC [1726] LOG:  checkpoint complete: wrote 0 buffers (0.0%); 0 WAL file(s) added, 0 removed, 0 recycled; write=0.001 s, sync=0.001 s, total=0.011 s; sync files=0, longest=0.000 s, average=0.000 s; distance=0 kB, estimate=18133 kB; lsn=1/89C8A8D0, redo lsn=1/89C8A8D0
2025-01-25 14:32:18.884 UTC [1725] LOG:  database system is shut down
```

В логе видно, что каждые 30 секунд создавалась контрольная точка, время записи 27 секунд.
В среднем контрольная точка оценивается в ~21 kB.
Все контрольные точки выполнялись по расписанию, это значит, что при текущей нагрузке генерируется объем журнальных записей не превышающий общий допустимый объем.
Последняя контрольная точка была вызвана при выключении и, если посмотрим состояние кластера через утилиту pg_controldata, то увидим что номер последней контрольной точки из лога совпадает с выводом состояния кластера.

myuser: `sudo /usr/lib/postgresql/17/bin/pg_controldata /var/lib/postgresql/17/main/`

```
pg_control version number:            1700
Catalog version number:               202406281
Database system identifier:           7461308807679983570
Database cluster state:               shut down
pg_control last modified:             Sat 25 Jan 2025 02:32:18 PM UTC
Latest checkpoint location:           1/89C8A8D0
Latest checkpoint's REDO location:    1/89C8A8D0
Latest checkpoint's REDO WAL file:    000000010000000100000089
Latest checkpoint's TimeLineID:       1
Latest checkpoint's PrevTimeLineID:   1
Latest checkpoint's full_page_writes: on
Latest checkpoint's NextXID:          0:839953
Latest checkpoint's NextOID:          24647
Latest checkpoint's NextMultiXactId:  1
Latest checkpoint's NextMultiOffset:  0
Latest checkpoint's oldestXID:        730
Latest checkpoint's oldestXID's DB:   1
Latest checkpoint's oldestActiveXID:  0
Latest checkpoint's oldestMultiXid:   1
Latest checkpoint's oldestMulti's DB: 1
Latest checkpoint's oldestCommitTsXid:0
Latest checkpoint's newestCommitTsXid:0
Time of latest checkpoint:            Sat 25 Jan 2025 02:32:18 PM UTC
Fake LSN counter for unlogged rels:   0/3E8
Minimum recovery ending location:     0/0
Min recovery ending loc's timeline:   0
Backup start location:                0/0
Backup end location:                  0/0
End-of-backup record required:        no
wal_level setting:                    replica
wal_log_hints setting:                off
max_connections setting:              100
max_worker_processes setting:         8
max_wal_senders setting:              10
max_prepared_xacts setting:           0
max_locks_per_xact setting:           64
track_commit_timestamp setting:       off
Maximum data alignment:               8
Database block size:                  8192
Blocks per segment of large relation: 131072
WAL block size:                       8192
Bytes per WAL segment:                16777216
Maximum length of identifiers:        64
Maximum columns in an index:          32
Maximum size of a TOAST chunk:        1996
Size of a large-object chunk:         2048
Date/time type storage:               64-bit integers
Float8 argument passing:              by value
Data page checksum version:           0
Mock authentication nonce:            f1cbefc948be36d39799796e3b51c0371ba6dbd48ca64fa96667d49f57feb44e
```

### 5. Сравним tps в синхронном/асинхронном режиме утилитой pgbench. Объясните полученный результат.

Добавим и применим параметры конфигурации:
```
synchronous_commit = off
```

Запускаем тест
- база данных postgres
- пользователь postgres
- продолжительность теста 60 сек

myuser: `pgbench -d postgres -T 60 -U postgres`
```
tps = 2120.543917 (without initial connection time)
```

Добавим и применим параметры конфигурации:
```
synchronous_commit = on
```

Запускаем тест
- база данных postgres
- пользователь postgres
- продолжительность теста 60 сек

myuser: `pgbench -d postgres -T 60 -U postgres`
```
tps = 571.410978 (without initial connection time)
```

По результатам тестов мы видим, что без синхронизации коммитов tps выше, потому что не тратится время на ожидание коммита.

### 6. Создадим новый кластер с включенной контрольной суммой страниц. Создадим таблицу. Вставим несколько значений. Выключим кластер. Изменим пару байт в таблице. Включим кластер и сделаем выборку из таблицы. Что и почему произошло? Как проигнорировать ошибку и продолжить работу?

В Postresql 17 включить контрольную сумму страниц можно в существующем кластере. Для этого его надо предварительно выключить. Воспользуемся этим, чтобы не пересоздавать кластер.

Выключим кластер:

myuser: `sudo pg_ctlcluster 17 main stop`

Включим контрольную сумму страниц:

myuser: `su - postgres -c '/usr/lib/postgresql/17/bin/pg_checksums --enable -D "/var/lib/postgresql/17/main"'`
```
Checksum operation completed
Files scanned:   965
Blocks scanned:  22322
Files written:  799
Blocks written: 22322
pg_checksums: syncing data directory
pg_checksums: updating control file
Checksums enabled in cluster
```

Включим кластер:

myuser: `sudo pg_ctlcluster 17 main start`

Подключимся и проверим:

myuser: `sudo -u postgres psql`
```
psql (17.2 (Ubuntu 17.2-1.pgdg24.04+1))
Type "help" for help.
```

postgres: `show data_checksums;`
```
data_checksums
----------------
on
(1 row)
```

Создадим таблицу:

postgres: `show data_checksums;`

Выйдем и создадим табличное пространство:

postgres: `\q`

myuser: `mkdir /data`
myuser: `mkdir /data/dbs`
myuser: `chown postgres:postgres /data/dbs`

Подключимся и создадим:

myuser: `sudo -u postgres psql`
```
psql (17.2 (Ubuntu 17.2-1.pgdg24.04+1))
Type "help" for help.
```

postgres: `CREATE TABLESPACE dbspace LOCATION '/data/dbs';`
```
CREATE TABLESPACE
```

Создадим таблицу и вставим несколько записей:

postgres: `CREATE TABLE cinemas (id serial, name text) TABLESPACE dbspace;`
```
CREATE TABLE
```

postgres: `INSERT INTO cinemas VALUES (1,'cinema1'), (2,'cinema2'), (3,'cinema3');`
```
INSERT 0 3
```

Выключим кластер:

postgres: `\q`

myuser: `sudo pg_ctlcluster 17 main stop`

Изменим пару байт в таблице. Включим кластер и сделаем выборку из таблицы.

myuser: `sudo pg_ctlcluster 17 main start`

myuser: `sudo -u postgres psql`
```
psql (17.2 (Ubuntu 17.2-1.pgdg24.04+1))
Type "help" for help.
```

postgres: `SELECT * FROM cinemas;`
```
 id | name
----+------
(0 rows)
```

Записи не получены, так как файл таблицы поврежден.

Попробуем сделать выборку с игнорированием чексуммы для этого применим настройку.

```
ignore_checksum_failure=on
```

postgres: `SELECT * FROM cinemas;`
```
 id | name
----+------
(0 rows)
```

Попробуем автоматически удалять поврежденные блоки, для этого применим настройку.

```
 zero_damaged_pages = on
```

postgres: `SELECT * FROM cinemas;`
```
 id | name
----+------
(0 rows)
```

Идей больше нет, надо восстанавливать через pg_dump/restore.
