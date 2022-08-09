# Журналирование
1. Зашел на ВМ, установил выполнение контрольных точек раз в 30 сек. Для этого отредактировал файл postgresql.conf
```bash
sudo nano /etc/postgresql/14/main/postgresql.conf
```

В нем установил значение параметра checkpoint_timeout = 30s;
Перезагрузил инстанс PG
2. Запустил утилиту pgbench на 10 минут
```bash
pgbench -c8 -P 60 -T 600 -U postgres postgres;
```
Дождался результатов.

3. Проверил, насколько изменилось дисковое пространство
```
bash
4.0K	./archive_status
97M	.
```
Проверил состояние файлов, все созданы сегодняшним днем (08.08.2022). То есть прирост дискогового пространства составил 97 Мб при выполнении контрольной точки 1 раз в 30 сек.
4. Запустил pgbench в синхронном и асинхронном режиме
Синхронный режим
```bash
pgbench -P 1 -T 10 buffer_temp
pgbench (14.4 (Ubuntu 14.4-1.pgdg22.04+1))
starting vacuum...end.
progress: 1.0 s, 557.0 tps, lat 1.782 ms stddev 0.624
progress: 2.0 s, 554.0 tps, lat 1.804 ms stddev 0.619
progress: 3.0 s, 573.0 tps, lat 1.745 ms stddev 0.576
progress: 4.0 s, 537.0 tps, lat 1.862 ms stddev 0.754
progress: 5.0 s, 348.0 tps, lat 2.866 ms stddev 1.568
progress: 6.0 s, 190.0 tps, lat 5.278 ms stddev 2.810
progress: 7.0 s, 295.0 tps, lat 3.383 ms stddev 1.254
progress: 8.0 s, 315.0 tps, lat 3.177 ms stddev 1.181
progress: 9.0 s, 302.0 tps, lat 3.311 ms stddev 1.547
progress: 10.0 s, 309.0 tps, lat 3.227 ms stddev 1.168
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 1
number of threads: 1
duration: 10 s
number of transactions actually processed: 3981
latency average = 2.510 ms
latency stddev = 1.477 ms
initial connection time = 6.507 ms
tps = 398.284645 (without initial connection time)
```
Асинхронный режим
```bash
pgbench -P 1 -T 10 buffer_temp
pgbench (14.4 (Ubuntu 14.4-1.pgdg22.04+1))
starting vacuum...end.
progress: 1.0 s, 1067.9 tps, lat 0.930 ms stddev 0.523
progress: 2.0 s, 1140.1 tps, lat 0.877 ms stddev 0.393
progress: 3.0 s, 1270.0 tps, lat 0.787 ms stddev 0.369
progress: 4.0 s, 1107.0 tps, lat 0.903 ms stddev 0.587
progress: 5.0 s, 1178.0 tps, lat 0.848 ms stddev 0.379
progress: 6.0 s, 1164.1 tps, lat 0.858 ms stddev 0.459
progress: 7.0 s, 1201.9 tps, lat 0.832 ms stddev 0.400
progress: 8.0 s, 1163.6 tps, lat 0.858 ms stddev 0.439
progress: 9.0 s, 1256.4 tps, lat 0.796 ms stddev 0.354
progress: 10.0 s, 1156.9 tps, lat 0.864 ms stddev 0.474
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 1
number of threads: 1
duration: 10 s
number of transactions actually processed: 11707
latency average = 0.854 ms
latency stddev = 0.443 ms
initial connection time = 5.598 ms
tps = 1170.996965 (without initial connection time)
```

5. Включил чек-суммы на кластере. Для этого сначала остановил его, потом выполнил команду
``` bash
/usr/lib/postgresql/14/bin/pg_checksums --enable -D "/var/lib/postgresql/14/main"
Checksum operation completed
Files scanned:  1265
Blocks scanned: 10690
pg_checksums: syncing data directory
pg_checksums: updating control file
Checksums enabled in cluster
```
После этого вставил данные в таблицу test в БД buffer_temp.
```sql
INSERT INTO test SELECT s.id FROM generate_series(1,100) AS s(id);
INSERT 0 100
```
Выключил кластер, поправил данные в таблице напрямую
``` bash
sudo dd if=/dev/zero of=/var/lib/postgresql/14/main/base/26538/26539 oflag=dsync conv=notrunc bs=1 count=8
```
Запустил кластер. Делаю выборку из таблицы и получаю ошибку
```sql
SELECT * FROM test;
WARNING:  page verification failed, calculated checksum 36483 but expected 28551
ERROR:  invalid page in block 0 of relation base/26538/26539
```
Ошибка из-за неправильной чек-суммы. Чтобы проигнорировать ошибку нужно перед select выполнить команду 
```sql
SET ignore_checksum_failure = on;
```
После этого select выполняется без ошибок, но с предупреждением
```sql
WARNING:  page verification failed, calculated checksum 36483 but expected 28551
```
