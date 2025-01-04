# Нагрузочное тестирование и тюнинг PostgreSQL

## Цель:

- Сделать нагрузочное тестирование PostgreSQL.
- Настроить параметры PostgreSQL для достижения максимальной производительности для 100 соединений.

## Пошаговая инструкция и результаты

### 1. Подготовка к тестам

### 1.1 Проверяем, что кластер запущен

myuser: `sudo -u postgres pg_lsclusters`
```
Ver Cluster Port Status Owner    Data directory              Log file
17  main    5432 online postgres /var/lib/postgresql/17/main /var/log/postgresql/postgresql-17-main.log
```

### 1.2 Параметры ВМ

- Платформа Intel Ice Lake
- Гарантированная доля vCPU 100%
- vCPU 2
- RAM 2 ГБ
- HDD 30 ГБ

### 1.3 Параметры тестов

- база данных postgres
- пользователь postgres
- количество клиентов 97 (3 резервируем)
- количество потоков 4
- показываем отчет о ходе выполнения каждые 10 сек
- продолжительность теста 60 сек
- каждую конфигурацию тестируем 5 раз

### 1.4 Рекомендации по настройке параметров конфигурации

#### shared_buffers
Используется для кэширования данных. По умолчанию низкое значение (для поддержки как можно большего кол-ва ОС). Начать стоит с его изменения. Согласно документации, рекомендуемое значение для данного параметра - 25% от общей оперативной памяти на сервере. PostgreSQL использует 2 кэша - свой (изменяется **shared_buffers**) и ОС. Редко значение больше, чем 40% окажет влияние на производительность.

#### max_connections
Максимальное количество соединений. Для изменения данного параметра придётся перезапускать сервер. Если планируется использование PostgreSQL как DWH, то большое количество соединений не нужно. Данный параметр тесно связан с **work_mem**. Поэтому будьте пределено аккуратны с ним

#### effective_cache_size
Служит подсказкой для планировщика, сколько ОП у него в запасе. Можно определить как **shared_buffers** + ОП системы - ОП используемое самой ОС и другими приложениями. За счёт данного параметра планировщик может чаще использовать индексы, строить hash таблицы. Наиболее часто используемое значение 75% ОП от общей на сервере.

#### work_mem
Используется для сортировок, построения hash таблиц. Это позволяет выполнять данные операции в памяти, что гораздо быстрее обращения к диску. В рамках одного запроса данный параметр может быть использован множество раз. Если ваш запрос содержит 5 операций сортировки, то память, которая потребуется для его выполнения уже как минимум **work_mem** * 5. Т.к. скорее-всего на сервере вы не одни и сессий много, то каждая из них может использовать этот параметр по нескольку раз, поэтому не рекомендуется делать его слишком большим. Можно выставить небольшое значение для глобального параметра в конфиге и потом, в случае сложных запросов, менять этот параметр локально (для текущей сессии)

#### maintenance_work_mem
Определяет максимальное количество ОП для операций типа VACUUM, CREATE INDEX, CREATE FOREIGN KEY. Увеличение этого параметра позволит быстрее выполнять эти операции. Не связано с **work_mem** поэтому можно ставить в разы больше, чем **work_mem**

#### wal_buffers
Объём разделяемой памяти, который будет использоваться для буферизации данных WAL, ещё не записанных на диск. Если у вас большое количество одновременных подключений, увеличение параметра улучшит производительность. По умолчанию -1, определяется автоматически, как 1/32 от **shared_buffers**, но не больше, чем 16 МБ (в ручную можно задавать большие значения). Обычно ставят 16 МБ

#### max_wal_size
Максимальный размер, до которого может вырастать WAL между автоматическими контрольными точками в WAL. Значение по умолчанию — 1 ГБ. Увеличение этого параметра может привести к увеличению времени, которое потребуется для восстановления после сбоя, но позволяет реже выполнять операцию сбрасывания на диск. Так же сбрасывание может выполниться и при достижении нужного времени, определённого параметром **checkpoint_timeout**

#### checkpoint_timeout
Чем реже происходит сбрасывание, тем дольше будет восстановление БД после сбоя. Значение по умолчанию 5 минут, рекомендуемое - от 30 минут до часа.
Необходимо "синхронизировать" два этих параметра. Для этого можно поставить **checkpoint_timeout** в выбранный промежуток, включить параметр **log_checkpoints** и по нему отследить, сколько было записано буферов. После чего подогнать параметр **max_wal_size**

### 2. Инициализируем утилиту pgbench

myuser: `pgbench -i postgres -U postgres`
```
creating tables...
generating data (client-side)...
vacuuming...
creating primary keys...
done in 0.52 s (drop tables 0.03 s, create tables 0.09 s, client-side generate 0.20 s, vacuum 0.11 s, primary keys 0.09 s).
```

### 3. Протестируем кластер через утилиту pgbench с конфигурацией по умолчанию

Конфигурация:
```
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

myuser: `pgbench -d postgres -c 97 -j 4 -P 10 -T 60 -U postgres`
```
tps = 397.420697 (without initial connection time)
tps = 409.630824 (without initial connection time)
tps = 396.104165 (without initial connection time)
tps = 405.460913 (without initial connection time)
tps = 367.554520 (without initial connection time)
```

### 3. Протестируем кластер через утилиту pgbench с кастомными конфигурациями

### 3.1 Конфигурация 1:
```
# Memory Configuration
shared_buffers = 512MB
effective_cache_size = 2GB
work_mem = 8MB
maintenance_work_mem = 128MB
# Checkpoint Related Configuration
min_wal_size = 2GB
max_wal_size = 3GB
wal_buffers = 16MB
```

myuser: `pgbench -d postgres -c 97 -j 4 -P 10 -T 60 -U postgres`
```
tps = 351.272813 (without initial connection time)
tps = 360.268913 (without initial connection time)
tps = 325.636296 (without initial connection time)
tps = 349.977907 (without initial connection time)
tps = 367.691647 (without initial connection time)
```

### 3.2 Конфигурация 2:
```
# Memory Configuration
shared_buffers = 256MB
effective_cache_size = 2GB
work_mem = 6MB
maintenance_work_mem = 128MB
# Checkpoint Related Configuration
min_wal_size = 1GB
max_wal_size = 2GB
wal_buffers = 8MB
```

myuser: `pgbench -d postgres -c 97 -j 4 -P 10 -T 60 -U postgres`
```
tps = 359.814279 (without initial connection time)
tps = 400.293038 (without initial connection time)
tps = 370.114632 (without initial connection time)
tps = 377.768560 (without initial connection time)
tps = 358.463360 (without initial connection time)
```

### 3.2 Конфигурация 3:
```
# Memory Configuration
shared_buffers = 64MB
effective_cache_size = 1GB
work_mem = 4MB
maintenance_work_mem = 64MB
# Checkpoint Related Configuration
min_wal_size = 100MB
max_wal_size = 1GB
wal_buffers = 2MB
```

myuser: `pgbench -d postgres -c 97 -j 4 -P 10 -T 60 -U postgres`
```
tps = 398.713053 (without initial connection time)
tps = 374.152305 (without initial connection time)
tps = 374.784243 (without initial connection time)
tps = 393.744305 (without initial connection time)
tps = 409.644818 (without initial connection time)
```

## Анализ результатов

Для 100 соединений самые лучшие результаты получены с конфигурацией по умолчанию и кастомной конфигурацией 3, в который даны самые минимальные ресурсы, меньше чем в конфигурации по умолчанию.

Рекомендуемые значения параметров выше чем значения по умолчанию и ожидалось, что с рекомендуемыми значениями параметров результат будет лучше, однако на практике это не подтвердилось.

Из всего выше изложенного можно сделать вывод, что нельзя слепо полагаться на рекомендации, надо делать тесты, так как результат может быть не очевиден.
