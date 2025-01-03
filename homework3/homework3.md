# Установка и настройка PostgreSQL

## Цель:

- Создать и подключить дополнительный диск для уже существующей виртуальной машины.
- Перенести содержимое базы данных PostgreSQL на дополнительный диск.

## Пошаговая инструкция и результаты

### 1. Проверяем, что кластер запущен

myuser: `sudo -u postgres pg_lsclusters`
```
Ver Cluster Port Status Owner    Data directory              Log file
17  main    5432 online postgres /var/lib/postgresql/17/main /var/log/postgresql/postgresql-17-main.log
```

### 2. Заходим под пользователем postgres в psql и делаем произвольную таблицу с произвольным содержимым

myuser: `psql -U postgres`

postgres: `create table test(c1 text);`
```
CREATE TABLE
```
postgres: `insert into test values('1');`
```
INSERT 0 1
```

### 3. Останавливаем postgres

myuser: `sudo -u postgres pg_ctlcluster 17 main stop`

### 4. Создаем новый диск размером 10GB и присоединяем его к ВМ

#### 4.1 Проверяем, подключен ли диск как устройство, и узнаем его путь в системе

myuser: `ls -la /dev/disk/by-id`
```
...
lrwxrwxrwx 1 root root   9 Dec 30 15:47 virtio-homework3-disk -> ../../vdb
```

#### 4.2 Размечаем диск

myuser: `sudo fdisk /dev/vdb`
```
Device     Boot Start      End  Sectors Size Id Type
/dev/vdb1        2048 20971519 20969472  10G 83 Linux
```

#### 4.3 Отформатируем раздел в нужную файловую систему

myuser: `sudo mkfs.ext4 /dev/vdb1`

#### 4.4 Смонтируем раздел диска

myuser: `sudo mkdir /mnt/data && sudo mount /dev/vdb1 /mnt/data`

### 5. Сделаем пользователя postgres владельцем /mnt/data

myuser: `chown -R postgres:postgres /mnt/data/`

### 6. Переносим содержимое /var/lib/postgres/17 в /mnt/data

myuser: `mv /var/lib/postgresql/17 /mnt/data`

### 7. Запускаем кластер

myuser: `sudo -u postgres pg_ctlcluster 17 main start`
```
Error: /var/lib/postgresql/17/main is not accessible or does not exist
```

Запустить не удалось, потому что директория с данными была перенесена.

### 8. Ищем конфигурационный параметр данных в файле конфигурации

Директория: `/etc/postgresql/17/main`

Файл: `postgresql.conf`

Добавляем в конце файла параметр конфигурации: `data_directory = '/mnt/data/17/main'`

### 9. Запускаем кластер

myuser: `sudo -u postgres pg_ctlcluster 17 main start`

#### 9.1 Проверяем, что кластер запущен

myuser: `sudo -u postgres pg_lsclusters`
```
Ver Cluster Port Status Owner    Data directory    Log file
17  main    5432 online postgres /mnt/data/17/main /var/log/postgresql/postgresql-17-main.log
```

Запустить удалось, потому что директория с данными была переопределена в файле конфигурации.

### 10. Заходим через psql и проверяем содержимое ранее созданной таблицы

myuser: `psql -U postgres`

postgres: `select * from test;`
```
 c1
----
 1
(1 row)
```

## Вывод

- При работе с PostgreSQL есть возможность перенести содержимое базы данных на дополнительный диск.
