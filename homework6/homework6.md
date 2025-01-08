# Механизм блокировок

## Цель:

- Понимать как работает механизм блокировок объектов и строк.

## Пошаговая инструкция и результаты

### 1. Проверяем, что кластер запущен

myuser: `sudo -u postgres pg_lsclusters`
```
Ver Cluster Port Status Owner    Data directory              Log file
17  main    5432 online postgres /var/lib/postgresql/17/main /var/log/postgresql/postgresql-17-main.log
```

### 2. Настраиваем сервер так, чтобы в журнал сообщений сбрасывалась информация о блокировках, удерживаемых более 200 миллисекунд

### 2.1 Добавим конфигурацию

```
log_lock_waits = on
deadlock_timeout = 200 ms
```

### 2.2 Применим конфигурацию

postgres: `SELECT pg_reload_conf();`
```
pg_reload_conf
----------------
t
(1 row)
```

### 2.3 Воспроизведем ситуацию, при которой в журнале появятся такие сообщения

#### Сессия #1
myuser: `psql -U postgres`
```
psql (17.2 (Ubuntu 17.2-1.pgdg24.04+1))
Type "help" for help.
```
postgres: `create database locks;`
```
CREATE DATABASE
```
postgres: `\c locks`
```
You are now connected to database "locks" as user "postgres".
```
locks: `CREATE TABLE accounts(acc_no integer PRIMARY KEY, amount numeric);`
```
CREATE TABLE
```
locks: `INSERT INTO accounts VALUES (1,1000.00), (2,2000.00), (3,3000.00);`
```
INSERT 0 3
```

#### Сессия #2
myuser: `psql -U postgres`
```
psql (17.2 (Ubuntu 17.2-1.pgdg24.04+1))
Type "help" for help.
```
postgres: `\c locks`
```
You are now connected to database "locks" as user "postgres".
```
locks: `BEGIN;`
```
BEGIN
```
locks: `UPDATE accounts SET amount = amount + 100 WHERE acc_no = 1;`
```
UPDATE 1
```

#### Сессия #1
locks: `BEGIN;`
```
BEGIN
```
locks: `CREATE INDEX ON accounts(acc_no);`

Ждем секунду и завершаем транзакцию во второй сессии.

#### Сессия #2
locks: `COMMIT;`
```
COMMIT
```

В первой сессии завершается команда создания индекса.

#### Сессия #1
```
CREATE INDEX
```
locks: `COMMIT;`
```
COMMIT
```

Смотрим журнал.

```
2025-01-07 16:43:00.369 UTC [8802] postgres@locks LOG:  process 8802 still waiting for ShareLock on relation 32790 of database 32789 after 200.153 ms
2025-01-07 16:43:00.369 UTC [8802] postgres@locks DETAIL:  Process holding the lock: 9060. Wait queue: 8802.
2025-01-07 16:43:00.369 UTC [8802] postgres@locks STATEMENT:  CREATE INDEX ON accounts(acc_no);
2025-01-07 16:44:54.308 UTC [8802] postgres@locks LOG:  process 8802 acquired ShareLock on relation 32790 of database 32789 after 114139.527 ms
2025-01-07 16:44:54.308 UTC [8802] postgres@locks STATEMENT:  CREATE INDEX ON accounts(acc_no);
```

### 3. Смоделируем ситуацию обновления одной и той же строки тремя командами UPDATE в разных сессиях.

Для удобства создадим представление, показывающее только интересующую нас информацию. 

locks: `CREATE VIEW locks_v AS
SELECT pid,
locktype,
CASE locktype
WHEN 'relation' THEN relation::regclass::text
WHEN 'transactionid' THEN transactionid::text
WHEN 'tuple' THEN relation::regclass::text||':'||tuple::text
END AS lockid,
mode,
granted
FROM pg_locks
WHERE locktype in ('relation','transactionid','tuple')
AND (locktype != 'relation' OR relation = 'accounts'::regclass);`
```
CREATE VIEW
```

#### Сессия #1
locks: `BEGIN;`
```
BEGIN
```
locks: `SELECT txid_current(), pg_backend_pid();`
```
 txid_current | pg_backend_pid
--------------+----------------
       537809 |           1978
(1 row)
```
locks: `UPDATE accounts SET amount = amount + 100 WHERE acc_no = 1;`
```
UPDATE 1
```
locks: `SELECT locktype, mode, granted, pid, pg_blocking_pids(pid) AS wait_for FROM pg_locks WHERE relation = 'accounts'::regclass;`
```
 locktype |       mode       | granted | pid  | wait_for
----------+------------------+---------+------+----------
 relation | RowExclusiveLock | t       | 1978 | {}
(1 row)
```
locks: `SELECT * FROM locks_v WHERE pid = 1978;`
```
 pid  |   locktype    |  lockid  |       mode       | granted
------+---------------+----------+------------------+---------
 1978 | relation      | accounts | RowExclusiveLock | t
 1978 | transactionid | 537809   | ExclusiveLock    | t
(2 rows)
```

В результате мы видим две строки:
1. блокировка в режиме Row Exclusive на таблицу 
2. блокировка в режиме Exclusive на собственный номер

#### Сессия #2
locks: `BEGIN;`
```
BEGIN
```
locks: `SELECT txid_current(), pg_backend_pid();`
```
 txid_current | pg_backend_pid
--------------+----------------
       537810 |           1997
(1 row)
```
locks: `UPDATE accounts SET amount = amount + 100 WHERE acc_no = 1;`

Команда в сессии 2 блокируется. Посмотрим блокировки в сессии 1.

locks: `SELECT locktype, mode, granted, pid, pg_blocking_pids(pid) AS wait_for FROM pg_locks WHERE relation = 'accounts'::regclass;`
```
 locktype |       mode       | granted | pid  | wait_for
----------+------------------+---------+------+----------
 relation | RowExclusiveLock | t       | 1978 | {}
 relation | RowExclusiveLock | t       | 1997 | {1978}
 tuple    | ExclusiveLock    | t       | 1997 | {1978}
(3 rows)
```
locks: `SELECT * FROM locks_v WHERE pid = 1997;`
```
 pid  |   locktype    |   lockid    |       mode       | granted
------+---------------+-------------+------------------+---------
 1997 | relation      | accounts    | RowExclusiveLock | t
 1997 | transactionid | 537810      | ExclusiveLock    | t
 1997 | transactionid | 537809      | ShareLock        | f
 1997 | tuple         | accounts:12 | ExclusiveLock    | t
(4 rows)
```

В результате мы видим 4 строки: 
1. блокировка в режиме Row Exclusive на таблицу
2. блокировка в режиме Exclusive на собственный номер
3. блокировка в режиме Share, которая ждет выполнения первой (granted = f)
4. блокировка в режиме Exclusive сгруппированная по строке

#### Сессия #3
locks: `BEGIN;`
```
BEGIN
```
locks: `SELECT txid_current(), pg_backend_pid();`
```
 txid_current | pg_backend_pid
--------------+----------------
       537811 |           2001
(1 row)
```
locks: `UPDATE accounts SET amount = amount + 100 WHERE acc_no = 1;`

Команда в сессии 3 блокируется. Посмотрим блокировки в сессии 1.

locks: `SELECT locktype, mode, granted, pid, pg_blocking_pids(pid) AS wait_for FROM pg_locks WHERE relation = 'accounts'::regclass;`
```
 locktype |       mode       | granted | pid  | wait_for
----------+------------------+---------+------+----------
 relation | RowExclusiveLock | t       | 1978 | {}
 relation | RowExclusiveLock | t       | 1997 | {1978}
 relation | RowExclusiveLock | t       | 2001 | {1997}
 tuple    | ExclusiveLock    | f       | 2001 | {1997}
 tuple    | ExclusiveLock    | t       | 1997 | {1978}
(5 rows)
```
locks: `SELECT * FROM locks_v WHERE pid = 2001;`
```
 pid  |   locktype    |   lockid    |       mode       | granted
------+---------------+-------------+------------------+---------
 2001 | relation      | accounts    | RowExclusiveLock | t
 2001 | tuple         | accounts:12 | ExclusiveLock    | f
 2001 | transactionid | 537811      | ExclusiveLock    | t
(3 rows)
```

В результате мы видим 3 строки:
1. блокировка в режиме Row Exclusive на таблицу
2. блокировка в режиме Exclusive сгруппированная по строке, которая ждет выполнения второй (granted = f)
3. блокировка в режиме Exclusive на собственный номер

Общую картину текущих ожиданий можно увидеть в представлении pg_stat_activity, добавив информацию о блокирующих процессах:

locks: `SELECT pid, wait_event_type, wait_event, pg_blocking_pids(pid)
FROM pg_stat_activity
WHERE backend_type = 'client backend' ORDER BY pid;`

Получается очередь, в которой есть первый, кто удерживает блокировку версии строки и все остальные, выстроившиеся за первым.

```
 pid  | wait_event_type |  wait_event   | pg_blocking_pids
------+-----------------+---------------+------------------
 1978 |                 |               | {}
 1997 | Lock            | transactionid | {1978}
 2001 | Lock            | tuple         | {1997}
(3 rows)
```

### 4. Воспроизведите взаимоблокировку трех транзакций. Можно ли разобраться в ситуации постфактум, изучая журнал сообщений?

#### Сессия #1
locks: `BEGIN;`
```
BEGIN
```
locks: `UPDATE accounts SET amount = amount + 100 WHERE acc_no = 1;`
```
UPDATE 1
```

#### Сессия #2
locks: `BEGIN;`
```
BEGIN
```
locks: `UPDATE accounts SET amount = amount + 100 WHERE acc_no = 2;`
```
UPDATE 1
```

#### Сессия #3
locks: `BEGIN;`
```
BEGIN
```
locks: `UPDATE accounts SET amount = amount + 100 WHERE acc_no = 3;`
```
UPDATE 1
```

#### Сессия #1
locks: `UPDATE accounts SET amount = amount - 100 WHERE acc_no = 3;`

#### Сессия #2
locks: `UPDATE accounts SET amount = amount - 100 WHERE acc_no = 1;`

#### Сессия #3
locks: `UPDATE accounts SET amount = amount - 100 WHERE acc_no = 2;`
```
ERROR:  deadlock detected
DETAIL:  Process 4859 waits for ShareLock on transaction 537817; blocked by process 4723.
Process 4723 waits for ShareLock on transaction 537816; blocked by process 4714.
Process 4714 waits for ShareLock on transaction 537818; blocked by process 4859.
HINT:  See server log for query details.
CONTEXT:  while updating tuple (0,2) in relation "accounts"
```
locks: `ROLLBACK;`
```
ROLLBACK
```

#### Сессия #1
```
UPDATE 1
```
locks: `ROLLBACK;`
```
ROLLBACK
```

#### Сессия #2
```
UPDATE 1
```
locks: `ROLLBACK;`
```
ROLLBACK
```

Смотрим журнал:

```
2025-01-08 17:29:43.817 UTC [4859] postgres@locks LOG:  process 4859 detected deadlock while waiting for ShareLock on transaction 537817 after 200.096 ms
2025-01-08 17:29:43.817 UTC [4859] postgres@locks DETAIL:  Process holding the lock: 4723. Wait queue: .
2025-01-08 17:29:43.817 UTC [4859] postgres@locks CONTEXT:  while updating tuple (0,2) in relation "accounts"
2025-01-08 17:29:43.817 UTC [4859] postgres@locks STATEMENT:  UPDATE accounts SET amount = amount - 100 WHERE acc_no = 2;
2025-01-08 17:29:43.817 UTC [4859] postgres@locks ERROR:  deadlock detected
2025-01-08 17:29:43.817 UTC [4859] postgres@locks DETAIL:  Process 4859 waits for ShareLock on transaction 537817; blocked by process 4723.
        Process 4723 waits for ShareLock on transaction 537816; blocked by process 4714.
        Process 4714 waits for ShareLock on transaction 537818; blocked by process 4859.
        Process 4859: UPDATE accounts SET amount = amount - 100 WHERE acc_no = 2;
        Process 4723: UPDATE accounts SET amount = amount - 100 WHERE acc_no = 1;
        Process 4714: UPDATE accounts SET amount = amount - 100 WHERE acc_no = 3;
```

По логу видно, что процесс 4859 (сессия 3) обнаружил взаимоблокировку, так как хотел обновить строку, 
которая была заблокирована в процессе 4723 (сессия 2), 
которая была заблокирована в процессе 4714 (сессия 1),
которая была заблокирована в процессе 4859 (сессия 3).

### 5. Могут ли две транзакции, выполняющие единственную команду UPDATE одной и той же таблицы (без where), заблокировать друг друга?

В большинстве случаев взаимоблокировки команды UPDATE не должно быть. 
Однако, так как эта команда блокирует строки по мере их обновления, а это происходит не одномоментно, то,
если одна команда будет обновлять строки в одном порядке, а другая — в другом, они могут взаимозаблокироваться.
