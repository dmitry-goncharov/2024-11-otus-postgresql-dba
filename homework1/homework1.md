# Работа с уровнями изоляции транзакции в PostgreSQL

## Цель:

- Научиться управлять уровнем изоляции транзации в PostgreSQL.
- Понимать особенность работы уровней read commited и repeatable read.

## Пошаговая инструкция и результаты

### 1. Выполняем в первой сессии

myuser: `psql -U postgres`

postgres: `\set AUTOCOMMIT off`

postgres: `create table persons(id serial, first_name text, second_name text);`<br>
CREATE TABLE

postgres: `insert into persons(first_name, second_name) values('ivan', 'ivanov');`<br>
INSERT 0 1

postgres: `insert into persons(first_name, second_name) values('petr', 'petrov');`<br>
INSERT 0 1

postgres: `commit;`<br>
COMMIT

postgres: `show transaction isolation level;`<br>
read committed

### 2. Выполняем во второй сессии

myuser: `psql -U postgres`

postgres: `\set AUTOCOMMIT off`

### 3. Начинаем новую транзакцию в обоих сессиях с дефолтным (не меняя) уровнем изоляции

postgres: `begin;`<br>
BEGIN

### 4. В первой сессии делаем вставку

postgres: `insert into persons(first_name, second_name) values('sergey', 'sergeev');`<br>
INSERT 0 1

### 5. Делаем выборку во второй сессии

postgres: `select from persons;`<br>
2 rows

Не видим новую запись, потому что транзакция в первой сессии еще не закончена.

### 6. Завершаем транзакцию в первой сессии

postgres: `commit;`<br>
COMMIT

### 7. Делаем выборку во второй сессии

postgres: `select from persons;`<br>
3 rows

Видим новую запись, потому что транзакция в первой сессии закончена.

### 8. Завершаем транзакцию во второй сессии

postgres: `commit;`<br>
COMMIT

### 9. Начинаем новые, но уже repeatable read транзации в обеих сессиях

postgres: `set transaction isolation level repeatable read;`<br>
SET

### 10. В первой сессии добавляем новую запись

postgres: `insert into persons(first_name, second_name) values('sveta', 'svetova');`<br>
INSERT 0 1

### 11. Во второй сессии делаем выборку 

postgres: `select * from persons;`<br>
3 rows

Не видим новую запись, потому что транзакции еще не закончены.

### 12. Завершаем транзакцию в первой сессии

postgres: `commit;`<br>
COMMIT

### 13. Во второй сессии делаем выборку 

postgres: `select * from persons;`<br>
3 rows

Не видим новую запись, потому что транзакция во второй сессии еще не закончена.

### 14. Завершаем транзакцию во второй сессии

postgres: `commit;`<br>
COMMIT

### 15. Во второй сессии делаем выборку 

postgres: `select * from persons;`<br>
4 rows

Видим новую запись, потому что транзакция во второй сессии закончена.

## Вывод

- В режиме read commited видны только те данные, которые были зафиксированы.
- В режиме repeatable read видны только те данные, которые были зафиксированы до начала транзакции.
