# Секционирование таблицы

## Цель:

- Научиться выполнять секционирование таблиц в PostgreSQL;
- Повысить производительность запросов и упростив управление данными;

## Пошаговая инструкция и результаты

### 1. Проверяем, что кластер запущен

myuser: `sudo -u postgres pg_lsclusters`
```
Ver Cluster Port Status Owner    Data directory              Log file
17  main    5432 online postgres /var/lib/postgresql/17/main /var/log/postgresql/postgresql-17-main.log
```

### 2. Скачиваем и устанавливаем БД demo

myuser: `wget https://edu.postgrespro.ru/demo_small.zip && sudo apt install unzip && unzip demo_small.zip && sudo -u postgres psql -d postgres -f /home/yc-user/demo_small.sql -c 'alter database demo set search_path to bookings'`

### 3. Подключаемся БД demo

myuser: `psql -U postgres`
```
psql (17.2 (Ubuntu 17.2-1.pgdg24.04+1))
Type "help" for help.
```
postgres: `\c demo`
```
You are now connected to database "demo" as user "postgres".
```

### 4. Выбираем самую большую таблицу

demo: `select pg_size_pretty(pg_table_size('aircrafts')) as aircrafts,
pg_size_pretty(pg_table_size('airports')) as airports,
pg_size_pretty(pg_table_size('boarding_passes')) as boarding_passes,
pg_size_pretty(pg_table_size('bookings')) as bookings,
pg_size_pretty(pg_table_size('flights')) as flights,
pg_size_pretty(pg_table_size('seats')) as seats,
pg_size_pretty(pg_table_size('ticket_flights')) as ticket_flights,
pg_size_pretty(pg_table_size('tickets')) as tickets;`
```
aircrafts       | 16 kB
airports        | 48 kB
boarding_passes | 34 MB
bookings        | 13 MB
flights         | 3168 kB
seats           | 96 kB
ticket_flights  | 69 MB
tickets         | 48 MB
```

Самая большая таблица flights.

### 5. Секционируем таблицу flights

Посмотрим описание таблицы.

demo: `\d flights`
```
flight_id           | integer                  |           | not null | nextval('flights_flight_id_seq'::regclass)
flight_no           | character(6)             |           | not null |
scheduled_departure | timestamp with time zone |           | not null |
scheduled_arrival   | timestamp with time zone |           | not null |
departure_airport   | character(3)             |           | not null |
arrival_airport     | character(3)             |           | not null |
status              | character varying(20)    |           | not null |
aircraft_code       | character(3)             |           | not null |
actual_departure    | timestamp with time zone |           |          |
actual_arrival      | timestamp with time zone |           |          |
```

Так как нет информации по видам запросов по этой таблице, то предположим секционирование по хэшу по идентификатору.

Проверим корреляцию данных.

demo: `SELECT attname, correlation
FROM pg_stats
WHERE tablename = 'flights';`
```
flight_id           |              1
flight_no           |     0.09120633
scheduled_departure | -0.00029294132
scheduled_arrival   |   3.973967e-05
departure_airport   |    -0.11708804
arrival_airport     |     0.07445189
status              |      0.4683061
aircraft_code       |      0.3199894
actual_departure    |  -0.0018220871
actual_arrival      |  -0.0012785785
```

Значение корреляции flight_id равно 1, значит значения колонки хранятся строго по возрастанию.

Посмотрим количество записей в таблице.

demo: `select count(*) from flights;`
```
33121
```

Разделим на 3 секции.

demo: `explain analyze select * from flights where flight_id = 10000;`
```
 Index Scan using flights_pkey on flights  (cost=0.29..8.31 rows=1 width=63) (actual time=0.020..0.021 rows=1 loops=1)
   Index Cond: (flight_id = 10000)
 Planning Time: 0.069 ms
 Execution Time: 0.037 ms
```

demo: `explain analyze select * from flights where flight_id = 20000;`
```
 Index Scan using flights_pkey on flights  (cost=0.29..8.31 rows=1 width=63) (actual time=0.078..0.080 rows=1 loops=1)
   Index Cond: (flight_id = 20000)
 Planning Time: 0.069 ms
 Execution Time: 0.096 ms
```

demo: `explain analyze select * from flights where flight_id = 30000;`
```
 Index Scan using flights_pkey on flights  (cost=0.29..8.31 rows=1 width=63) (actual time=0.082..0.084 rows=1 loops=1)
   Index Cond: (flight_id = 30000)
 Planning Time: 0.069 ms
 Execution Time: 0.100 ms
```

Создадим новую секционированную таблицу:

demo: `create table by_hash_flights (
flight_id            integer generated always as identity primary key,
flight_no            character(6)                         not null, 
scheduled_departure  timestamp with time zone             not null, 
scheduled_arrival    timestamp with time zone             not null, 
departure_airport    character(3)                         not null, 
arrival_airport      character(3)                         not null, 
status               character varying(20)                not null, 
aircraft_code        character(3)                         not null, 
actual_departure     timestamp with time zone                     , 
actual_arrival       timestamp with time zone                      
) partition by hash (flight_id);`
```
CREATE TABLE
```

Создадим секции:

demo: `CREATE TABLE by_hash_flights_p1 PARTITION OF by_hash_flights FOR VALUES WITH (MODULUS 3, REMAINDER 0);`
```
CREATE TABLE
```
demo: `CREATE TABLE by_hash_flights_p2 PARTITION OF by_hash_flights FOR VALUES WITH (MODULUS 3, REMAINDER 1);`
```
CREATE TABLE
```
demo: `CREATE TABLE by_hash_flights_p3 PARTITION OF by_hash_flights FOR VALUES WITH (MODULUS 3, REMAINDER 2);`
```
CREATE TABLE
```

Перенесем данные:

demo: `insert into by_hash_flights (flight_no, scheduled_departure, scheduled_arrival, departure_airport, arrival_airport, status, aircraft_code, actual_departure, actual_arrival)
select flight_no, scheduled_departure, scheduled_arrival, departure_airport, arrival_airport, status, aircraft_code, actual_departure, actual_arrival from flights;`
```
INSERT 0 33121
```

Проверим:

demo: `select count(*) from by_hash_flights;`
```
33121
```

demo: `explain analyze select * from by_hash_flights where flight_id = 10000;`
```
 Index Scan using by_hash_flights_p2_pkey on by_hash_flights_p2 by_hash_flights  (cost=0.29..8.30 rows=1 width=63) (actual time=0.007..0.008 rows=1 loops=1)
   Index Cond: (flight_id = 10000)
 Planning Time: 0.278 ms
 Execution Time: 0.022 ms
```

demo: `explain analyze select * from by_hash_flights where flight_id = 15000;`
```
 Index Scan using by_hash_flights_p3_pkey on by_hash_flights_p3 by_hash_flights  (cost=0.29..8.30 rows=1 width=63) (actual time=0.007..0.008 rows=1 loops=1)
   Index Cond: (flight_id = 15000)
 Planning Time: 0.279 ms
 Execution Time: 0.022 ms
```

demo: `explain analyze select * from by_hash_flights where flight_id = 18000;`
```
 Index Scan using by_hash_flights_p1_pkey on by_hash_flights_p1 by_hash_flights  (cost=0.29..8.30 rows=1 width=63) (actual time=0.008..0.008 rows=1 loops=1)
   Index Cond: (flight_id = 18000)
 Planning Time: 0.256 ms
 Execution Time: 0.022 ms
```

По результатам видим, что используются секции, негативного влияния нет, все работает корректно.
