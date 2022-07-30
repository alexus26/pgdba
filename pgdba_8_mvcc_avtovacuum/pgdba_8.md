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
