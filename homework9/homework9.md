# Работа с индексами

## Цель:

- Знать и уметь применять основные виды индексов PostgreSQL
- Строить и анализировать план выполнения запроса
- Уметь оптимизировать запросы для с использованием индексов

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

### 3. Создадим индекс к какой-либо из таблиц БД

#### 3.1 Создадим таблицу продуктов

postgres: `CREATE TABLE products(
product_id   integer,
price        integer
);`
```
CREATE TABLE
```

postgres: `WITH random_data AS (
    SELECT
    num,
    random() AS rand
    FROM generate_series(1, 100000) AS s(num)
)
INSERT INTO products
    (product_id, price)
SELECT
    random_data.num,
    (random_data.rand * 100)::integer
    FROM random_data
    ORDER BY random();`
```
INSERT 0 100000
```

#### 3.2 Проверяем результат выборки по product_id

postgres: `EXPLAIN
SELECT * FROM products
WHERE product_id = 1;`
```
                        QUERY PLAN
-----------------------------------------------------------
 Seq Scan on products  (cost=0.00..1693.00 rows=1 width=8)
   Filter: (product_id = 1)
(2 rows)
```

По результату видим, что используется последовательное сканирование таблицы.

#### 3.3 Добавляем индекс B-Tree на поле product_id

postgres: `CREATE INDEX
idx_products_product_id
ON products(product_id);`
```
CREATE INDEX
```

#### 3.4 Проверяем результат выборки по product_id

postgres: `EXPLAIN
SELECT * FROM products
WHERE product_id = 1;`
```
                                       QUERY PLAN
----------------------------------------------------------------------------------------
 Index Scan using idx_products_product_id on products  (cost=0.29..8.31 rows=1 width=8)
   Index Cond: (product_id = 1)
(2 rows)
```

По результату видим, что используется индекс.

### 4. Создадим индекс для полнотекстового поиска

#### 4.1 Создадим таблицу авторов

postgres: `CREATE TABLE authors(
author_id       integer,
author_name     text,
author_name_tsv tsvector
);`
```
CREATE TABLE
```

postgres: `WITH random_data AS (
    SELECT
    num,
    random() AS rand
    FROM generate_series(1, 100000) AS s(num)
)
INSERT INTO authors
    (author_id, author_name)
SELECT
    random_data.num,
    md5(random_data.rand::text)
    FROM random_data
    ORDER BY random();`
```
INSERT 0 100000
```

postgres: `UPDATE authors SET author_name_tsv = to_tsvector(author_name);`
```
UPDATE 100000
```

#### 4.2 Проверяем результат выборки по author_name_tsv

postgres: `EXPLAIN
SELECT * FROM authors
WHERE author_name_tsv @@ to_tsquery('1a1af4554d2321b20cf2342ae3261aa4');`
```
                                        QUERY PLAN
-------------------------------------------------------------------------------------------
 Gather  (cost=1000.00..18707.48 rows=3 width=82)
   Workers Planned: 1
   ->  Parallel Seq Scan on authors  (cost=0.00..17707.18 rows=2 width=82)
         Filter: (author_name_tsv @@ to_tsquery('1a1af4554d2321b20cf2342ae3261aa4'::text))
(4 rows)
```

По результату видим, что используется параллельное последовательное сканирование таблицы.

#### 4.3 Добавляем индекс GIN на поле author_name_tsv

postgres: `CREATE INDEX
idx_authors_author_name
ON authors
USING GIN (author_name_tsv);`
```
CREATE INDEX
```

#### 4.4 Проверяем результат выборки по author_name_tsv

postgres: `EXPLAIN
SELECT * FROM authors
WHERE author_name_tsv @@ to_tsquery('1a1af4554d2321b20cf2342ae3261aa4');`
```
                                          QUERY PLAN
-----------------------------------------------------------------------------------------------
 Bitmap Heap Scan on authors  (cost=17.35..29.81 rows=3 width=82)
   Recheck Cond: (author_name_tsv @@ to_tsquery('1a1af4554d2321b20cf2342ae3261aa4'::text))
   ->  Bitmap Index Scan on idx_authors_author_name  (cost=0.00..17.35 rows=3 width=0)
         Index Cond: (author_name_tsv @@ to_tsquery('1a1af4554d2321b20cf2342ae3261aa4'::text))
(4 rows)
```

По результату видим, что используется индекс.

### 5. Создадим индекс на часть таблицы

#### 5.1 Добавим колонку is_available в таблицу products

postgres: `ALTER TABLE products ADD COLUMN is_available BOOLEAN;`
```
ALTER TABLE
```

postgres: `UPDATE products SET is_available = (
case when product_id % 2 = 0 then true else false end
);`
```
UPDATE 100000
```

postgres: `select is_available, count(*) from products group by is_available;`
```
 is_available | count
--------------+-------
 f            | 50000
 t            | 50000
(2 rows)
```

#### 5.2 Проверяем результат выборки по is_available = true

postgres: `EXPLAIN
SELECT * FROM products
WHERE is_available = true;`
```
                          QUERY PLAN
---------------------------------------------------------------
 Seq Scan on products  (cost=0.00..1984.00 rows=49910 width=9)
   Filter: is_available
(2 rows)
```

По результату видим, что используется последовательное сканирование таблицы.

#### 5.3 Добавляем частичный индекс B-Tree на поле is_available = true

postgres: `CREATE INDEX
idx_products_is_available_true
ON products(is_available)
WHERE is_available = true;`
```
CREATE INDEX
```

#### 5.4 Проверяем результат выборки по is_available = true

postgres: `EXPLAIN
SELECT * FROM products
WHERE is_available = true;`
```
                                            QUERY PLAN
---------------------------------------------------------------------------------------------------
 Bitmap Heap Scan on products  (cost=438.32..1921.42 rows=49910 width=9)
   Recheck Cond: is_available
   ->  Bitmap Index Scan on idx_products_is_available_true  (cost=0.00..425.84 rows=49910 width=0)
(3 rows)
```

По результату видим, что используется индекс.

### 6. Создадим индекс на несколько полей

#### 6.1 Проверяем результат выборки по полям price и is_available

postgres: `EXPLAIN
SELECT * FROM products
WHERE price > 100 AND is_available = true;`
```
                        QUERY PLAN
-----------------------------------------------------------
 Seq Scan on products  (cost=0.00..2234.00 rows=5 width=9)
   Filter: (is_available AND (price > 100))
(2 rows)
```

По результату видим, что используется последовательное сканирование таблицы.

#### 6.2 Добавляем составной индекс B-Tree на поля is_available и price

postgres: `CREATE INDEX
idx_products_is_available_price
ON products(is_available, price);`
```
CREATE INDEX
```

#### 6.3 Проверяем результат  выборки по полям price и is_available

postgres: `EXPLAIN
SELECT * FROM products
WHERE price > 100 AND is_available = true;`
```
                                           QUERY PLAN
-------------------------------------------------------------------------------------------------
 Index Scan using idx_products_is_available_price on products  (cost=0.29..22.12 rows=5 width=9)
   Index Cond: ((is_available = true) AND (price > 100))
(2 rows)
```

По результату видим, что используется индекс.
