# Настройка PostgreSQL
1. Установил PostgreSQL 14 в Ubuntu Server 22.04 на VirtualBox

2. Сгенерировал оптимальные параметры для сервера через утилиту pgtune, получил такие данные
```bash
# DB Version: 14
# OS Type: linux
# DB Type: mixed
# Total Memory (RAM): 2048 MB
# CPUs num: 2
# Connections num: 100
# Data Storage: ssd

max_connections = 100
shared_buffers = 512MB
effective_cache_size = 1536MB
maintenance_work_mem = 128MB
checkpoint_completion_target = 0.9
wal_buffers = 16MB
default_statistics_target = 100
random_page_cost = 1.1
effective_io_concurrency = 200
work_mem = 2621kB
min_wal_size = 1GB
max_wal_size = 4GB
max_worker_processes = 2
max_parallel_workers_per_gather = 1
max_parallel_workers = 2
max_parallel_maintenance_workers = 1
```

3. Скопировал данные в файл pgtune.conf в директории /etc/postgresql/14/main/conf.d. Проверил, что данная директория "включена" в основном конфиге
```bash
include_dir = 'conf.d'
```

4. Перезапустил кластер
``systemctl restart postgresql@14-main.service``

5. Проверил, что мои настройки используются кластером
```sql
select * from pg_file_settings;
                 sourcefile                  | sourceline | seqno |               name               |                 setting                 | applied | error 
---------------------------------------------+------------+-------+----------------------------------+-----------------------------------------+---------+-------
 /etc/postgresql/14/main/postgresql.conf     |         42 |     1 | data_directory                   | /var/lib/postgresql/14/main             | t       | 
 /etc/postgresql/14/main/postgresql.conf     |         44 |     2 | hba_file                         | /etc/postgresql/14/main/pg_hba.conf     | t       | 
 /etc/postgresql/14/main/postgresql.conf     |         46 |     3 | ident_file                       | /etc/postgresql/14/main/pg_ident.conf   | t       | 
 /etc/postgresql/14/main/postgresql.conf     |         50 |     4 | external_pid_file                | /var/run/postgresql/14-main.pid         | t       | 
 /etc/postgresql/14/main/postgresql.conf     |         64 |     5 | port                             | 5432                                    | t       | 
 /etc/postgresql/14/main/postgresql.conf     |         65 |     6 | max_connections                  | 100                                     | f       | 
 /etc/postgresql/14/main/postgresql.conf     |         67 |     7 | unix_socket_directories          | /var/run/postgresql                     | t       | 
 /etc/postgresql/14/main/postgresql.conf     |        105 |     8 | ssl                              | on                                      | t       | 
 /etc/postgresql/14/main/postgresql.conf     |        107 |     9 | ssl_cert_file                    | /etc/ssl/certs/ssl-cert-snakeoil.pem    | t       | 
 /etc/postgresql/14/main/postgresql.conf     |        110 |    10 | ssl_key_file                     | /etc/ssl/private/ssl-cert-snakeoil.key  | t       | 
 /etc/postgresql/14/main/postgresql.conf     |        127 |    11 | shared_buffers                   | 128MB                                   | f       | 
 /etc/postgresql/14/main/postgresql.conf     |        150 |    12 | dynamic_shared_memory_type       | posix                                   | t       | 
 /etc/postgresql/14/main/postgresql.conf     |        240 |    13 | max_wal_size                     | 1GB                                     | f       | 
 /etc/postgresql/14/main/postgresql.conf     |        241 |    14 | min_wal_size                     | 80MB                                    | f       | 
 /etc/postgresql/14/main/postgresql.conf     |        542 |    15 | log_line_prefix                  | %m [%p] %q%u@%d                         | t       | 
 /etc/postgresql/14/main/postgresql.conf     |        580 |    16 | log_timezone                     | Etc/UTC                                 | t       | 
 /etc/postgresql/14/main/postgresql.conf     |        587 |    17 | cluster_name                     | 14/main                                 | t       | 
 /etc/postgresql/14/main/postgresql.conf     |        604 |    18 | stats_temp_directory             | /var/run/postgresql/14-main.pg_stat_tmp | t       | 
 /etc/postgresql/14/main/postgresql.conf     |        694 |    19 | datestyle                        | iso, mdy                                | t       | 
 /etc/postgresql/14/main/postgresql.conf     |        696 |    20 | timezone                         | Etc/UTC                                 | t       | 
 /etc/postgresql/14/main/postgresql.conf     |        710 |    21 | lc_messages                      | en_US.UTF-8                             | t       | 
 /etc/postgresql/14/main/postgresql.conf     |        712 |    22 | lc_monetary                      | en_US.UTF-8                             | t       | 
 /etc/postgresql/14/main/postgresql.conf     |        713 |    23 | lc_numeric                       | en_US.UTF-8                             | t       | 
 /etc/postgresql/14/main/postgresql.conf     |        714 |    24 | lc_time                          | en_US.UTF-8                             | t       | 
 /etc/postgresql/14/main/postgresql.conf     |        717 |    25 | default_text_search_config       | pg_catalog.english                      | t       | 
 /etc/postgresql/14/main/conf.d/pg_tune.conf |          1 |    26 | max_connections                  | 100                                     | t       | 
 /etc/postgresql/14/main/conf.d/pg_tune.conf |          2 |    27 | shared_buffers                   | 512MB                                   | t       | 
 /etc/postgresql/14/main/conf.d/pg_tune.conf |          3 |    28 | effective_cache_size             | 1536MB                                  | t       | 
 /etc/postgresql/14/main/conf.d/pg_tune.conf |          4 |    29 | maintenance_work_mem             | 128MB                                   | t       | 
 /etc/postgresql/14/main/conf.d/pg_tune.conf |          5 |    30 | checkpoint_completion_target     | 0.9                                     | t       | 
 /etc/postgresql/14/main/conf.d/pg_tune.conf |          6 |    31 | wal_buffers                      | 16MB                                    | t       | 
 /etc/postgresql/14/main/conf.d/pg_tune.conf |          7 |    32 | default_statistics_target        | 100                                     | t       | 
 /etc/postgresql/14/main/conf.d/pg_tune.conf |          8 |    33 | random_page_cost                 | 1.1                                     | t       | 
 /etc/postgresql/14/main/conf.d/pg_tune.conf |          9 |    34 | effective_io_concurrency         | 200                                     | t       | 
 /etc/postgresql/14/main/conf.d/pg_tune.conf |         10 |    35 | work_mem                         | 2621kB                                  | t       | 
 /etc/postgresql/14/main/conf.d/pg_tune.conf |         11 |    36 | min_wal_size                     | 1GB                                     | t       | 
 /etc/postgresql/14/main/conf.d/pg_tune.conf |         12 |    37 | max_wal_size                     | 4GB                                     | t       | 
 /etc/postgresql/14/main/conf.d/pg_tune.conf |         13 |    38 | max_worker_processes             | 2                                       | t       | 
 /etc/postgresql/14/main/conf.d/pg_tune.conf |         14 |    39 | max_parallel_workers_per_gather  | 1                                       | t       | 
 /etc/postgresql/14/main/conf.d/pg_tune.conf |         15 |    40 | max_parallel_workers             | 2                                       | t       | 
 /etc/postgresql/14/main/conf.d/pg_tune.conf |         16 |    41 | max_parallel_maintenance_workers | 1                                       | t       | 
(41 rows)
```
6. Создал БД для тестирования pgbench
```sql
create database pgbench_test;
```

7. Инициализировал в ней pgbench
```bash
pgbench -i pgbench_test
```

8. Запустил pgbench
```bash
pgbench -c8 -P 15 -T 120 -U postgres pgbench_test
```
Получил результаты
```bash
pgbench (14.5 (Ubuntu 14.5-1.pgdg22.04+1))
starting vacuum...end.
progress: 15.0 s, 310.2 tps, lat 25.691 ms stddev 26.856
progress: 30.0 s, 326.3 tps, lat 24.505 ms stddev 26.774
progress: 45.0 s, 308.7 tps, lat 25.943 ms stddev 26.942
progress: 60.0 s, 307.5 tps, lat 26.000 ms stddev 27.021
progress: 75.0 s, 304.3 tps, lat 26.298 ms stddev 27.205
progress: 90.0 s, 321.5 tps, lat 24.874 ms stddev 26.741
progress: 105.0 s, 303.8 tps, lat 26.325 ms stddev 27.137
progress: 120.0 s, 313.9 tps, lat 25.491 ms stddev 27.037
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 8
number of threads: 1
duration: 120 s
number of transactions actually processed: 37450
latency average = 25.628 ms
latency stddev = 26.967 ms
initial connection time = 41.051 ms
tps = 312.108721 (without initial connection time)
```

9. Позапускал pgbench в разных режимах
```bash
pgbench -c 50 -j 2 -P 10 -T 30 pgbench_test
pgbench (14.5 (Ubuntu 14.5-1.pgdg22.04+1))
starting vacuum...end.
progress: 10.0 s, 279.2 tps, lat 174.377 ms stddev 138.381
progress: 20.0 s, 292.2 tps, lat 171.561 ms stddev 134.828
progress: 30.0 s, 284.2 tps, lat 175.974 ms stddev 139.452
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 50
number of threads: 2
duration: 30 s
number of transactions actually processed: 8606
latency average = 174.336 ms
latency stddev = 137.591 ms
initial connection time = 89.485 ms
tps = 285.967033 (without initial connection time)
```

```bash
pgbench -c 50 -C -j 2 -P 10 -T 30 -M extended pgbench_test
pgbench (14.5 (Ubuntu 14.5-1.pgdg22.04+1))
starting vacuum...end.
progress: 10.0 s, 170.8 tps, lat 272.603 ms stddev 287.115
progress: 20.0 s, 174.3 tps, lat 281.932 ms stddev 303.128
progress: 30.0 s, 169.8 tps, lat 294.965 ms stddev 387.775
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: extended
number of clients: 50
number of threads: 2
duration: 30 s
number of transactions actually processed: 5199
latency average = 284.757 ms
latency stddev = 331.029 ms
average connection time = 4.207 ms
tps = 172.286776 (including reconnection times)
```

Как видно, с увеличением количества клиентов падает значение tps в режиме синхронных коммитов.

10. Начал эксперименты с параметрами. Поменял shared_buffers 512Мb на 128Мb, запустил утилиту pgbench. Результаты
```bash
pgbench -c 50 -C -j 2 -P 10 -T 30 -M extended pgbench_test
pgbench (14.5 (Ubuntu 14.5-1.pgdg22.04+1))
starting vacuum...end.
progress: 10.0 s, 170.1 tps, lat 282.353 ms stddev 330.273
progress: 20.0 s, 177.1 tps, lat 277.677 ms stddev 274.129
progress: 30.0 s, 170.8 tps, lat 285.734 ms stddev 287.292
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: extended
number of clients: 50
number of threads: 2
duration: 30 s
number of transactions actually processed: 5230
latency average = 283.039 ms
latency stddev = 298.472 ms
average connection time = 4.100 ms
tps = 173.398306 (including reconnection times)
```
Как видно, результаты не сильно изменились с прошлого запуска. Возвращаю параметр на 512Mb.

11. Поменял параметр work_mem с 2621kB на 16МB. Проверяю параметр нагрузкой
```bash
pgbench -c 50 -C -j 2 -P 10 -T 60 -M extended pgbench_test
pgbench (14.5 (Ubuntu 14.5-1.pgdg22.04+1))
starting vacuum...end.
progress: 10.0 s, 169.2 tps, lat 275.686 ms stddev 303.572
progress: 20.0 s, 168.6 tps, lat 292.204 ms stddev 336.307
progress: 30.0 s, 173.4 tps, lat 289.129 ms stddev 303.932
progress: 40.0 s, 158.1 tps, lat 311.284 ms stddev 423.532
progress: 50.0 s, 161.8 tps, lat 303.910 ms stddev 380.303
progress: 60.0 s, 161.0 tps, lat 309.155 ms stddev 342.732
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: extended
number of clients: 50
number of threads: 2
duration: 60 s
number of transactions actually processed: 9971
latency average = 296.440 ms
latency stddev = 349.319 ms
average connection time = 4.522 ms
tps = 165.690179 (including reconnection times)
```
Результаты также не сильно изменились. Меняю параметр обратно.

12. Поменял параметр effective_io_concurrency с 200 на 500 (знаю, что для SSD рекомендуют ставить 200). Проверил результаты
```bash
pgbench -c 50 -C -j 2 -P 10 -T 60 -M extended pgbench_test
pgbench (14.5 (Ubuntu 14.5-1.pgdg22.04+1))
starting vacuum...end.
progress: 10.0 s, 165.1 tps, lat 290.203 ms stddev 312.007
progress: 20.0 s, 173.1 tps, lat 280.637 ms stddev 399.678
progress: 30.0 s, 170.7 tps, lat 290.060 ms stddev 324.469
progress: 40.0 s, 173.1 tps, lat 283.079 ms stddev 288.657
progress: 50.0 s, 178.1 tps, lat 280.427 ms stddev 341.955
progress: 60.0 s, 167.9 tps, lat 291.681 ms stddev 301.950
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: extended
number of clients: 50
number of threads: 2
duration: 60 s
number of transactions actually processed: 10330
latency average = 286.247 ms
latency stddev = 330.555 ms
average connection time = 4.237 ms
tps = 171.693071 (including reconnection times)
```
Как видно, tps немного выросло, но не в разы. Меняю значение параметра effective_io_concurrency на 200.

13. Теперь попробуем включить режим асинхронных транзакций.
```sql
ALTER SYSTEM SET synchronous_commit = off;
ALTER SYSTEM
postgres=# SELECT pg_reload_conf();
 pg_reload_conf 
----------------
 t
(1 row)
```

```bash
pgbench -c 50 -C -j 2 -P 10 -T 60 -M extended pgbench_test
pgbench (14.5 (Ubuntu 14.5-1.pgdg22.04+1))
starting vacuum...end.
progress: 10.0 s, 245.1 tps, lat 193.470 ms stddev 223.573
progress: 20.0 s, 248.6 tps, lat 195.504 ms stddev 200.323
progress: 30.0 s, 247.6 tps, lat 198.022 ms stddev 192.467
progress: 40.0 s, 248.0 tps, lat 194.841 ms stddev 168.123
progress: 50.0 s, 248.2 tps, lat 196.075 ms stddev 174.001
progress: 60.0 s, 249.4 tps, lat 194.843 ms stddev 182.556
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: extended
number of clients: 50
number of threads: 2
duration: 60 s
number of transactions actually processed: 14920
latency average = 195.713 ms
latency stddev = 191.167 ms
average connection time = 5.379 ms
tps = 248.314201 (including reconnection times)
```

Как видно, при асинхронных транзакциях tps выросло на 45% (со 171 до 248). 

На этом эксперименты думаю завершать.
