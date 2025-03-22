# Работа с join'ами, статистикой

## Цель:

- Знать и уметь применять различные виды join'ов;
- Строить и анализировать план выполнения запроса;
- Оптимизировать запрос;
- Уметь собирать и анализировать статистику для таблицы;

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

### 3. Реализуем прямое соединение двух или более таблиц

#### 3.1 Создадим БД stud

postgres: `CREATE DATABASE stud;`
```
CREATE DATABASE
```

#### 3.2 Переключимся на БД stud

postgres: `\c stud`
```
You are now connected to database "stud" as user "postgres".
```

#### 3.3 Создадим таблицу student

stud: `CREATE TABLE student(id SERIAL PRIMARY KEY, name TEXT);`
```
CREATE TABLE
```

#### 3.4 Наполним таблицу student

stud: `INSERT INTO student(name) VALUES
('alex'),('dan'),('duk'),('ben'),('pilar'),('juli'),('doli'),('poli'),('vin'),('marta');`
```
INSERT 0 10
```

#### 3.5 Создадим таблицу email

stud: `CREATE TABLE email(id serial, student_id integer, email text,
FOREIGN KEY(student_id) REFERENCES student(id) ON DELETE CASCADE);`
```
CREATE TABLE
```

#### 3.6 Наполним таблицу email

stud: `INSERT INTO email(student_id,email) VALUES
(1,'alex@email.com'),(2,'dan@email.com'),(3,'duk@email.com'),(4,'ben@email.com'),(5,'pilar@email.com'),
(6,'juli@email.com'),(7,'doli@email.com'),(8,'poli@email.com'),(9,'vin@email.com'),(10,'marta@email.com');`
```
INSERT 0 10
```

#### 3.7 Сделаем выборку из таблиц student и email с прямым соединением

stud: `select s.name as student, e.email from student s inner join email e on e.student_id = s.id;`
```
 student |      email
---------+-----------------
 alex    | alex@email.com
 dan     | dan@email.com
 duk     | duk@email.com
 ben     | ben@email.com
 pilar   | pilar@email.com
 juli    | juli@email.com
 doli    | doli@email.com
 poli    | poli@email.com
 vin     | vin@email.com
 marta   | marta@email.com
(10 rows)
```

### 4. Реализуем левостороннее (или правостороннее) соединение двух или более таблиц

#### 4.1 Создадим таблицу addr

stud: `CREATE TABLE addr(id serial, student_id integer, city text,
FOREIGN KEY(student_id) REFERENCES student(id) ON DELETE CASCADE);`
```
CREATE TABLE
```

#### 4.2 Наполним таблицу addr

stud: `INSERT INTO addr(student_id,city) VALUES
(1,'Moscow'),(2,'London'),(5,'Madrid'),
(6,'Paris'),(7,'Rome'),(10,'SPb');`
```
INSERT 0 6
```

#### 4.3 Сделаем выборку из таблиц student и addr с левосторонним соединением

stud: `select s.name as student, a.city from student s left join addr a on a.student_id = s.id;`
```
 student |  city
---------+--------
 alex    | Moscow
 dan     | London
 pilar   | Madrid
 juli    | Paris
 doli    | Rome
 marta   | SPb
 poli    |
 ben     |
 duk     |
 vin     |
(10 rows)
```

### 5. Реализуем кросс соединение двух или более таблиц

#### 5.1 Создадим таблицу subject

stud: `CREATE TABLE subject(id SERIAL PRIMARY KEY, name TEXT);`
```
CREATE TABLE
```

#### 5.2 Наполним таблицу subject

stud: `INSERT INTO subject(name) VALUES
('math'),('geo'),('lang');`
```
INSERT 0 3
```

#### 5.3 Сделаем выборку из таблиц student и subject с перекрестным соединением

stud: `select s.name as student, sb.name as subject from student s cross join subject sb;`
```
 student | subject
---------+---------
 alex    | math
 alex    | geo
 alex    | lang
 dan     | math
 dan     | geo
 dan     | lang
 duk     | math
 duk     | geo
 duk     | lang
 ben     | math
 ben     | geo
 ben     | lang
 pilar   | math
 pilar   | geo
 pilar   | lang
 juli    | math
 juli    | geo
 juli    | lang
 doli    | math
 doli    | geo
 doli    | lang
 poli    | math
 poli    | geo
 poli    | lang
 vin     | math
 vin     | geo
 vin     | lang
 marta   | math
 marta   | geo
 marta   | lang
(30 rows)
```

### 6. Реализуем полное соединение двух или более таблиц

#### 6.1 Создадим таблицу discount

stud: `CREATE TABLE discount(id serial, student_id integer, size integer,
FOREIGN KEY(student_id) REFERENCES student(id) ON DELETE CASCADE);`
```
CREATE TABLE
```

#### 6.2 Наполним таблицу discount

stud: `INSERT INTO discount(student_id,size) VALUES
(3,3),(4,4),(5,5),
(8,3),(9,4),(10,5);`
```
INSERT 0 6
```

#### 6.3 Сделаем выборку из таблиц addr и discount с полным соединением

stud: `select a.city, d.size as discount from addr a full join discount d on a.student_id = d.student_id;`
```
  city  | discount
--------+----------
 Moscow |
 London |
        |        3
        |        4
 Madrid |        5
 Paris  |
 Rome   |
        |        3
        |        4
 SPb    |        5
(10 rows)
```

### 7. Реализуем запрос, в котором будут использованы разные типы соединений

#### 7.1 Сделаем выборку из таблиц student, email, addr и discount с прямым и левосторонним соединениями

stud: `select s.name as student, e.email, a.city, d.size as discount from student s 
inner join email e on e.student_id = s.id 
left join addr a on a.student_id = s.id 
left join discount d on d.student_id = s.id;`
```
 student |      email      |  city  | discount
---------+-----------------+--------+----------
 duk     | duk@email.com   |        |        3
 ben     | ben@email.com   |        |        4
 pilar   | pilar@email.com | Madrid |        5
 poli    | poli@email.com  |        |        3
 vin     | vin@email.com   |        |        4
 marta   | marta@email.com | SPb    |        5
 juli    | juli@email.com  | Paris  |
 doli    | doli@email.com  | Rome   |
 dan     | dan@email.com   | London |
 alex    | alex@email.com  | Moscow |
(10 rows)
```
