# Установка и настройка PostgteSQL в контейнере Docker

## Цель:

- Установить PostgreSQL в Docker контейнере.
- Настроить контейнер для внешнего подключения.

## Пошаговая инструкция и результаты

### 1. Ставим Docker Engine

myuser: `curl -fsSL https://get.docker.com -o get-docker.sh && sudo sh get-docker.sh && rm get-docker.sh && sudo usermod -aG docker $USER && newgrp docker`

#### 1.1 Проверяем 

myuser: `docker --version`
```
Docker version 27.4.0, build bde2b89
```

#### 1.2 Создаем docker-сеть:

myuser: `sudo docker network create pg-net`
```
d15cf7a52af4ed85792ef9319ffbbf736343629951fe570f55a2e79c104b1b2c
```

### 2. Создаем каталог /opt/postgres

### 3. Разворавчиваем контейнер с PostgreSQL 17 смонтировав в него /opt/postgres

myuser: `sudo docker run --name pg-server --network pg-net -e POSTGRES_PASSWORD=postgres -d -p 5432:5432 -v /opt/postgres:/var/lib/postgresql/data postgres:17`

#### 3.1 Проверяем

myuser: `sudo docker ps -a`
```
649bbf4466c8   postgres:17   "docker-entrypoint.s…"   4 minutes ago   Up 3 minutes   0.0.0.0:5432->5432/tcp, :::5432->5432/tcp   pg-server
```

### 4. Разворавчиваем контейнер с клиентом postgres и подключаемся из контейнера с клиентом к контейнеру с сервером

myuser: `sudo docker run -it --rm --network pg-net --name pg-client postgres:17 psql -h pg-server -U postgres`

### 5. Создаем базу и таблицу с парой строк

postgres: `CREATE DATABASE otus;`
```
CREATE DATABASE
```

postgres: `create table persons(id serial, first_name text, second_name text);`<br>
```
CREATE TABLE
```

postgres: `insert into persons(first_name, second_name) values('ivan', 'ivanov');`<br>
```
INSERT 0 1
```

postgres: `insert into persons(first_name, second_name) values('petr', 'petrov');`<br>
```
INSERT 0 1
```

### 6. Подключаемся к контейнеру с сервером

myuser: `psql -h localhost -U postgres -d postgres`

postgres: `\l`
```
                                                    List of databases
   Name    |  Owner   | Encoding | Locale Provider |  Collate   |   Ctype    | Locale | ICU Rules |   Access privileges
-----------+----------+----------+-----------------+------------+------------+--------+-----------+-----------------------
 otus      | postgres | UTF8     | libc            | en_US.utf8 | en_US.utf8 |        |           |
 postgres  | postgres | UTF8     | libc            | en_US.utf8 | en_US.utf8 |        |           |
 template0 | postgres | UTF8     | libc            | en_US.utf8 | en_US.utf8 |        |           | =c/postgres          +
           |          |          |                 |            |            |        |           | postgres=CTc/postgres
 template1 | postgres | UTF8     | libc            | en_US.utf8 | en_US.utf8 |        |           | =c/postgres          +
           |          |          |                 |            |            |        |           | postgres=CTc/postgres
(4 rows)
```

postgres: `\dt`
```
List of relations
Schema |  Name   | Type  |  Owner
--------+---------+-------+----------
public | persons | table | postgres
(1 row)
```

### 7. Удалить контейнер с сервером

### 7.1 Получем идентификатор контейнера

myuser: `sudo docker ps -a`
```
CONTAINER ID   IMAGE         COMMAND                  CREATED          STATUS          PORTS                                       NAMES
649bbf4466c8   postgres:17   "docker-entrypoint.s…"   31 minutes ago   Up 31 minutes   0.0.0.0:5432->5432/tcp, :::5432->5432/tcp   pg-server
```

### 7.2 Удалем контейнер

myuser: `docker rm -f 649bbf4466c8`
```
649bbf4466c8
```

### 8. Создаем заново контейнер с сервером

myuser: `sudo docker run --name pg-server --network pg-net -e POSTGRES_PASSWORD=postgres -d -p 5432:5432 -v /opt/postgres:/var/lib/postgresql/data postgres:17`
```
18b3dfe71a7cd32d25853ddc1f03e5135ea512b5e768b529792c6656ea4a6887
```

### 9. Подключаемся снова из контейнера с клиентом к контейнеру с сервером

myuser: `sudo docker run -it --rm --network pg-net --name pg-client postgres:17 psql -h pg-server -U postgres`

### 10. Проверяем, что данные остались на месте

postgres: `\l`
```
                                                    List of databases
   Name    |  Owner   | Encoding | Locale Provider |  Collate   |   Ctype    | Locale | ICU Rules |   Access privileges
-----------+----------+----------+-----------------+------------+------------+--------+-----------+-----------------------
 otus      | postgres | UTF8     | libc            | en_US.utf8 | en_US.utf8 |        |           |
 postgres  | postgres | UTF8     | libc            | en_US.utf8 | en_US.utf8 |        |           |
 template0 | postgres | UTF8     | libc            | en_US.utf8 | en_US.utf8 |        |           | =c/postgres          +
           |          |          |                 |            |            |        |           | postgres=CTc/postgres
 template1 | postgres | UTF8     | libc            | en_US.utf8 | en_US.utf8 |        |           | =c/postgres          +
           |          |          |                 |            |            |        |           | postgres=CTc/postgres
(4 rows)
```

postgres: `\dt`
```
List of relations
Schema |  Name   | Type  |  Owner
--------+---------+-------+----------
public | persons | table | postgres
(1 row)
```

## Вывод

- При работе с PostgreSQL в Docker контейнере данные остаются на месте, если volume для них монтируется.
