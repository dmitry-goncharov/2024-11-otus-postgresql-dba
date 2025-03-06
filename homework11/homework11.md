# Триггеры, поддержка заполнения витрин

## Цель:

- Создать триггер для поддержки витрины в актуальном состоянии;

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

### 3. Создадим БД (shopdb) с товарами (таблица goods) и продажами (таблица sales)

postgres: `CREATE DATABASE shopdb;`
```
CREATE DATABASE
```

### 4. Заходим в созданную базу данных под пользователем postgres

postgres: `\c shopdb`
```
You are now connected to database "shopdb" as user "postgres".
```

### 5. Создадим схему pract_functions

shopdb: `CREATE SCHEMA pract_functions;`
```
CREATE SCHEMA
```

shopdb: `SET search_path = pract_functions, public;`
```
SET
```

### 6. Создадим таблицу с товарами (goods)

shopdb: `CREATE TABLE goods
(
    goods_id    integer PRIMARY KEY,
    good_name   varchar(63) NOT NULL,
    good_price  numeric(12, 2) NOT NULL CHECK (good_price > 0.0)
);`
```
CREATE TABLE
```

shopdb: `INSERT INTO goods (goods_id, good_name, good_price)
VALUES 	(1, 'Спички хозайственные', .50),
		(2, 'Автомобиль Ferrari FXX K', 185000000.01);`
```
INSERT 0 2
```

### 7. Создадим таблицу с продажами (sales)

shopdb: `CREATE TABLE sales
(
    sales_id    integer GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    good_id     integer REFERENCES goods (goods_id),
    sales_time  timestamp with time zone DEFAULT now(),
    sales_qty   integer CHECK (sales_qty > 0)
);`
```
CREATE TABLE
```

shopdb: `INSERT INTO sales (good_id, sales_qty) VALUES (1, 10), (1, 1), (1, 120), (2, 1);`
```
INSERT 0 4
```

### 8. Создадим запрос для генерации отчета – сумма продаж по каждому товару

shopdb: `SELECT G.good_name, sum(G.good_price * S.sales_qty)
FROM goods G
INNER JOIN sales S ON S.good_id = G.goods_id
GROUP BY G.good_name;`
```
        good_name         |     sum
--------------------------+--------------
 Автомобиль Ferrari FXX K | 185000000.01
 Спички хозайственные     |        65.50
(2 rows)
```

### 9. Денормализуем БД, создадим таблицу витрина (good_sum_mart), структура которой повторяет структуру отчета

shopdb: `CREATE TABLE good_sum_mart
(
	good_name   varchar(63) NOT NULL,
	sum_sale	numeric(16, 2) NOT NULL
);`
```
CREATE TABLE
```

### 10.  Создадим триггер на таблице продаж (sales), для поддержки данных в витрине в актуальном состоянии (вычисляющий при каждой продаже сумму и записывающий её в витрину)

Так как в нашей таблице витрины нет идентификаторов товаров, то в триггерной функции будем вызывать перестроение всей таблицы витрины.

shopdb:
```
CREATE OR REPLACE FUNCTION tf_update_good_sum_mart()
RETURNS trigger
AS
$$
BEGIN
    IF TG_LEVEL = 'STATEMENT' THEN
        TRUNCATE TABLE good_sum_mart;
        INSERT INTO good_sum_mart (good_name, sum_sale)
        SELECT G.good_name, sum(G.good_price * S.sales_qty)
        FROM goods G
        INNER JOIN sales S ON S.good_id = G.goods_id
        GROUP BY G.good_name;
    END IF;
    RETURN NULL;
END;
$$ LANGUAGE plpgsql;
```
```
CREATE FUNCTION
```

shopdb:
```
CREATE TRIGGER trg_after_update_sales
AFTER INSERT OR UPDATE OR DELETE
ON sales
FOR EACH STATEMENT
EXECUTE FUNCTION tf_update_good_sum_mart();
```
```
CREATE TRIGGER
```

### 11. Проверяем триггер

shopdb: `SELECT * FROM good_sum_mart;`
```
 good_name | sum_sale
-----------+----------
(0 rows)
```

shopdb: `INSERT INTO sales (good_id, sales_qty) VALUES (1, 1);`
```
INSERT 0 1
```

shopdb: `SELECT * FROM good_sum_mart;`
```
        good_name         |   sum_sale
--------------------------+--------------
 Автомобиль Ferrari FXX K | 185000000.01
 Спички хозайственные     |        66.00
(2 rows)
```
