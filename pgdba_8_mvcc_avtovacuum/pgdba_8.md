# MVVC и avtovacuum
1. Поставил PostgreSQL 14 на Ubuntu 22.04 в Virtualbox
```bash
pg_lsclusters
Ver Cluster Port Status Owner    Data directory              Log file
14  main    5432 online postgres /var/lib/postgresql/14/main /var/log/postgresql/postgresql-14-main.log
```
2. Поменял параметры согласно файла в ДЗ, перезапустил кластер
```bash
alexus@ubuntupg:~$ sudo systemctl stop postgresql@14-main
[sudo] password for alexus: 
alexus@ubuntupg:~$ sudo systemctl start postgresql@14-main || sudo systemctl status postgresql@14-main
alexus@ubuntupg:~$ sudo systemctl status postgresql@14-main
● postgresql@14-main.service - PostgreSQL Cluster 14-main
     Loaded: loaded (/lib/systemd/system/postgresql@.service; enabled-runtime; vendor preset: enabled)
     Active: active (running) since Sat 2022-07-30 09:33:45 UTC; 7s ago
    Process: 1612 ExecStart=/usr/bin/pg_ctlcluster --skip-systemctl-redirect 14-main start (code=exited, status=0/SUCCESS)
   Main PID: 1617 (postgres)
      Tasks: 7 (limit: 2240)
     Memory: 45.7M
        CPU: 216ms
     CGroup: /system.slice/system-postgresql.slice/postgresql@14-main.service
             ├─1617 /usr/lib/postgresql/14/bin/postgres -D /var/lib/postgresql/14/main -c config_file=/etc/postgresql/14/main/postgresql.conf
             ├─1619 "postgres: 14/main: checkpointer " "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" >
             ├─1620 "postgres: 14/main: background writer " "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" ">
             ├─1621 "postgres: 14/main: walwriter " "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" >
             ├─1622 "postgres: 14/main: autovacuum launcher " "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "">
             ├─1623 "postgres: 14/main: stats collector " "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" >
             └─1624 "postgres: 14/main: logical replication launcher " "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "">
Jul 30 09:33:43 ubuntupg systemd[1]: Starting PostgreSQL Cluster 14-main...
Jul 30 09:33:45 ubuntupg systemd[1]: Started PostgreSQL Cluster 14-main.
```
3. Запустил команду `pgbench -c8 -P 60 -T 3600 -U postgres postgres`
4. Дождался результатов
```bash
progress: 3060.0 s, 584.7 tps, lat 13.685 ms stddev 13.072
progress: 3120.0 s, 721.8 tps, lat 11.082 ms stddev 8.286
progress: 3180.0 s, 706.2 tps, lat 11.327 ms stddev 8.669
progress: 3240.0 s, 717.7 tps, lat 11.147 ms stddev 7.956
progress: 3300.0 s, 631.6 tps, lat 12.665 ms stddev 10.686
progress: 3360.0 s, 682.4 tps, lat 11.720 ms stddev 8.514
progress: 3420.0 s, 739.1 tps, lat 10.826 ms stddev 7.891
progress: 3480.0 s, 678.7 tps, lat 11.786 ms stddev 7.933
progress: 3540.0 s, 643.5 tps, lat 12.432 ms stddev 8.669
progress: 3600.0 s, 723.2 tps, lat 11.060 ms stddev 7.716
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 8
number of threads: 1
duration: 3600 s
number of transactions actually processed: 2752112
latency average = 10.464 ms
latency stddev = 7.511 ms
initial connection time = 21.404 ms
tps = 764.473714 (without initial connection time)
```
5. Настроил vacuum
```bash
vacuum_cost_delay = 0                   # 0-100 milliseconds (0 disables)
vacuum_cost_page_hit = 1                # 0-10000 credits
vacuum_cost_page_miss = 2               # 0-10000 credits
vacuum_cost_page_dirty = 20             # 0-10000 credits
vacuum_cost_limit = 200         # 1-10000 credits
```
и autovacum
```bash
autovacuum = on   
autovacuum_max_workers = 6 
autovacuum_naptime = 15s 
autovacuum_vacuum_threshold = 50
autovacuum_analyze_threshold = 50
autovacuum_vacuum_scale_factor = 0.05
autovacuum_vacuum_cost_delay = 2ms
autovacuum_vacuum_cost_limit = -1
```
Запустил `pgbench -c8 -P 60 -T 600 -U postgres postgres`
Результаты
```bash
pgbench (14.4 (Ubuntu 14.4-1.pgdg22.04+1))
starting vacuum...end.
progress: 60.0 s, 730.0 tps, lat 10.952 ms stddev 8.848
progress: 120.0 s, 709.9 tps, lat 11.270 ms stddev 8.346
progress: 180.0 s, 743.2 tps, lat 10.763 ms stddev 7.483
progress: 240.0 s, 678.2 tps, lat 11.794 ms stddev 8.965
progress: 300.0 s, 744.9 tps, lat 10.740 ms stddev 7.688
progress: 360.0 s, 606.8 tps, lat 13.174 ms stddev 11.763
progress: 420.0 s, 688.6 tps, lat 11.624 ms stddev 9.689
progress: 480.0 s, 627.9 tps, lat 12.739 ms stddev 9.613
progress: 540.0 s, 529.5 tps, lat 15.107 ms stddev 11.262
progress: 600.0 s, 546.9 tps, lat 14.625 ms stddev 10.441
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 8
number of threads: 1
duration: 600 s
number of transactions actually processed: 396363
latency average = 12.109 ms
latency stddev = 9.482 ms
initial connection time = 24.717 ms
tps = 660.595378 (without initial connection time)
```
Как видно, результаты в горизонте 10 минут довольно разнятся.
6. Снова настроил autovacuum
```bash
autovacuum = on
autovacuum_max_workers = 6
autovacuum_naptime = 1s
autovacuum_vacuum_threshold = 50
autovacuum_analyze_threshold = 0
autovacuum_vacuum_scale_factor = 0.03
autovacuum_analyze_scale_factor = 0.02
autovacuum_vacuum_cost_delay = 2ms
autovacuum_vacuum_cost_limit = -1
```
Запустил `pgbench -c8 -P 60 -T 600 -U postgres postgres`
Результаты
```bash
starting vacuum...end.
progress: 60.0 s, 639.3 tps, lat 12.499 ms stddev 12.344
progress: 120.0 s, 616.7 tps, lat 12.976 ms stddev 10.680
progress: 180.0 s, 764.6 tps, lat 10.461 ms stddev 7.521
progress: 240.0 s, 413.5 tps, lat 19.334 ms stddev 22.611
progress: 300.0 s, 71.9 tps, lat 111.124 ms stddev 82.881
progress: 360.0 s, 145.6 tps, lat 55.020 ms stddev 67.127
progress: 420.0 s, 500.1 tps, lat 15.996 ms stddev 13.613
progress: 480.0 s, 705.9 tps, lat 11.332 ms stddev 8.876
progress: 540.0 s, 691.1 tps, lat 11.576 ms stddev 8.988
progress: 600.0 s, 645.7 tps, lat 12.389 ms stddev 10.477
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 8
number of threads: 1
duration: 600 s
number of transactions actually processed: 311676
latency average = 15.399 ms
latency stddev = 23.205 ms
initial connection time = 36.264 ms
tps = 519.470806 (without initial connection time)
```
Результаты стали еще хуже, есть результаты 71,9 tps.
7. Снова оптимизирую **avtovacuum**
Выполняю команду pgbench, результаты
```bash
pgbench -c8 -P 60 -T 360 -U postgres postgres
pgbench (14.4 (Ubuntu 14.4-1.pgdg22.04+1))
starting vacuum...end.
progress: 60.0 s, 807.5 tps, lat 9.900 ms stddev 7.473
progress: 120.0 s, 614.5 tps, lat 13.018 ms stddev 12.569
progress: 180.0 s, 740.2 tps, lat 10.808 ms stddev 7.391
progress: 240.0 s, 661.6 tps, lat 12.091 ms stddev 9.426
progress: 300.0 s, 720.8 tps, lat 11.098 ms stddev 8.146
progress: 360.0 s, 707.7 tps, lat 11.302 ms stddev 7.914
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 8
number of threads: 1
duration: 360 s
number of transactions actually processed: 255147
latency average = 11.287 ms
latency stddev = 8.911 ms
initial connection time = 24.286 ms
tps = 708.730709 (without initial connection time)
```
8. Пробую еще раз оптимизировать autovacuum, меняю значение параметров на 0.01
```bash
autovacuum_vacuum_scale_factor = 0.01
autovacuum_vacuum_insert_scale_factor = 0.01
```
Результат
```bash
pgbench -c8 -P 60 -T 300 -U postgres postgres
pgbench (14.4 (Ubuntu 14.4-1.pgdg22.04+1))
starting vacuum...end.
progress: 60.0 s, 794.4 tps, lat 10.062 ms stddev 7.232
progress: 120.0 s, 524.0 tps, lat 15.270 ms stddev 13.564
progress: 180.0 s, 749.8 tps, lat 10.669 ms stddev 7.416
progress: 240.0 s, 618.4 tps, lat 12.934 ms stddev 10.404
progress: 300.0 s, 590.9 tps, lat 13.538 ms stddev 12.181
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 8
number of threads: 1
duration: 300 s
number of transactions actually processed: 196662
latency average = 12.203 ms
latency stddev = 10.268 ms
initial connection time = 24.643 ms
tps = 655.521364 (without initial connection time)
```
9. Снова оптимизирую autovacuum
```bash
autovacuum_vacuum_scale_factor = 0.1
autovacuum_vacuum_insert_scale_factor = 0.1
autovacuum_analyze_scale_factor = 0.5
```
Результат
```bash
pgbench (14.4 (Ubuntu 14.4-1.pgdg22.04+1))
starting vacuum...end.
progress: 60.0 s, 782.2 tps, lat 10.218 ms stddev 8.004
progress: 120.0 s, 439.2 tps, lat 18.222 ms stddev 21.343
progress: 180.0 s, 783.8 tps, lat 10.205 ms stddev 6.884
progress: 240.0 s, 597.5 tps, lat 13.381 ms stddev 11.392
progress: 300.0 s, 592.6 tps, lat 13.507 ms stddev 10.752
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 8
number of threads: 1
duration: 300 s
number of transactions actually processed: 191724
latency average = 12.516 ms
latency stddev = 11.959 ms
initial connection time = 26.082 ms
tps = 639.094175 (without initial connection time)
```
Как видно, tps стало меньше при этих значениях.

10. В последний раз меняю значения autovacuum
```bash
autovacuum_naptime = 90s
autovacuum_vacuum_scale_factor = 0.05
autovacuum_vacuum_insert_scale_factor = 0.05
autovacuum_analyze_scale_factor = 0.03
```
Результат
```bash
pgbench -c8 -P 15 -T 300 -U postgres postgres
pgbench (14.4 (Ubuntu 14.4-1.pgdg22.04+1))
starting vacuum...end.
progress: 15.0 s, 925.6 tps, lat 8.622 ms stddev 4.252
progress: 30.0 s, 909.5 tps, lat 8.792 ms stddev 4.444
progress: 45.0 s, 927.7 tps, lat 8.627 ms stddev 4.281
progress: 60.0 s, 362.7 tps, lat 22.037 ms stddev 21.170
progress: 75.0 s, 409.8 tps, lat 19.530 ms stddev 17.918
progress: 90.0 s, 852.1 tps, lat 9.391 ms stddev 5.253
progress: 105.0 s, 681.7 tps, lat 11.734 ms stddev 8.589
progress: 120.0 s, 763.0 tps, lat 10.485 ms stddev 8.373
progress: 135.0 s, 766.9 tps, lat 10.426 ms stddev 7.496
progress: 150.0 s, 735.7 tps, lat 10.874 ms stddev 7.802
progress: 165.0 s, 640.7 tps, lat 12.489 ms stddev 9.445
progress: 180.0 s, 884.4 tps, lat 9.045 ms stddev 4.800
progress: 195.0 s, 751.7 tps, lat 10.641 ms stddev 8.143
progress: 210.0 s, 707.5 tps, lat 11.306 ms stddev 9.030
progress: 225.0 s, 706.4 tps, lat 11.326 ms stddev 7.799
progress: 240.0 s, 635.6 tps, lat 12.566 ms stddev 9.340
progress: 255.0 s, 689.8 tps, lat 11.616 ms stddev 9.098
progress: 270.0 s, 756.3 tps, lat 10.566 ms stddev 7.467
progress: 285.0 s, 809.6 tps, lat 9.888 ms stddev 6.488
progress: 300.0 s, 615.3 tps, lat 12.994 ms stddev 9.664
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 8
number of threads: 1
duration: 300 s
number of transactions actually processed: 217989
latency average = 11.009 ms
latency stddev = 8.850 ms
initial connection time = 29.023 ms
tps = 726.596581 (without initial connection time)
```
TPS выросло, как мне кажется потому, что autovacuum запускался не так часто. Считаю это приемлемые настройки
```bash
vacuum_cost_delay = 0                   # 0-100 milliseconds (0 disables)
vacuum_cost_page_hit = 1                # 0-10000 credits
vacuum_cost_page_miss = 2               # 0-10000 credits
vacuum_cost_page_dirty = 20             # 0-10000 credits
vacuum_cost_limit = 200         # 1-10000 credits
max_worker_processes = 4
autovacuum = on
autovacuum_max_workers = 3
autovacuum_naptime = 90s
autovacuum_vacuum_threshold = 500
autovacuum_vacuum_insert_threshold = 1000
autovacuum_analyze_threshold = 500
autovacuum_vacuum_scale_factor = 0.05
autovacuum_vacuum_insert_scale_factor = 0.05
autovacuum_analyze_scale_factor = 0.03
autovacuum_vacuum_cost_delay = 2ms
autovacuum_vacuum_cost_limit = -1
```
Запустил на час тест
Результат
```bash
progress: 3060.0 s, 552.5 tps, lat 14.480 ms stddev 9.954
progress: 3120.0 s, 640.4 tps, lat 12.488 ms stddev 8.791
progress: 3180.0 s, 585.4 tps, lat 13.667 ms stddev 9.922
progress: 3240.0 s, 603.3 tps, lat 13.260 ms stddev 8.968
progress: 3300.0 s, 540.7 tps, lat 14.789 ms stddev 11.424
progress: 3360.0 s, 545.7 tps, lat 14.663 ms stddev 10.602
progress: 3420.0 s, 595.6 tps, lat 13.431 ms stddev 9.243
progress: 3480.0 s, 544.6 tps, lat 14.690 ms stddev 10.616
progress: 3540.0 s, 468.4 tps, lat 17.075 ms stddev 11.791
progress: 3600.0 s, 568.7 tps, lat 14.065 ms stddev 9.765
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 8
number of threads: 1
duration: 3600 s
number of transactions actually processed: 2140238
latency average = 13.455 ms
latency stddev = 9.653 ms
initial connection time = 21.284 ms
tps = 594.508398 (without initial connection time)
```
Значения TPS стали ровнее, относительно первого теста, но количество транзакций снизилось.







