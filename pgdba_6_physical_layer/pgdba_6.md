# Перенос БД PostgreSQL на новый диск (папку)
1. Поставили PostgreSQL 14 на ВМ командой
````bash
sudo apt update && sudo apt upgrade -y -q && sudo sh -c 'echo "deb http://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list' && wget --quiet -O - https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo apt-key add - && sudo apt-get update && sudo apt -y install postgresql-14
````
После завершения установки проверили, что инстанс PosgreSQL поставился и работает:
`pg_lsclusters`
Результат
```bash
Ver Cluster Port Status Owner    Data directory              Log file
14  main    5432 online postgres /var/lib/postgresql/14/main /var/log/postgresql/postgresql-14-main.log
```
2. Добавили новый жесткий диск к ВМ в Virtualbox
![Alt text](https://github.com/alexus26/pgdba/blob/main/pgdba_6_physical_layer/%D0%A1%D0%BD%D0%B8%D0%BC%D0%BE%D0%BA%20%D1%8D%D0%BA%D1%80%D0%B0%D0%BD%D0%B0%20%D0%BE%D1%82%202022-07-22%2011-10-06.png "a title")

инициализировал его, разметил и добавил в **fstab** для автоматической загрузки

`/dev/disk/by-partlabel/primary	/home/alexus/pg_tablespaces	ext4	defaults 0 2`

3. Выдаем права на папку, в которую смотнирован новый жесткий диск, для пользователя postgresql
`drwxrwxrwx 3 postgres postgres 4096 июл 21 18:23 pg_tablespaces`
4. Остановил кластер PosgreSQL
```bash
Ver Cluster Port Status Owner    Data directory              Log file
14  main    5432 down   postgres /var/lib/postgresql/14/main /var/log/postgresql/postgresql-14-main.log
```
5. Перенес содержимое попытался запустить кластер:
`systemctl start postgresql@14-main`
и получил ошибку:
`Job for postgresql@14-main.service failed because the service did not take the steps required by its unit configuration.`
Посмотрел журнал
`systemctl status postgresql@14-main.service | journalctl -xe`
там расшифровка ошибки:
```bash
июл 22 07:33:27 pg1 postgresql@14-main[14690]: Error: /var/lib/postgresql/14/main is not accessible or does not exist
```
Ошибка понятна, мы же перенесли (не скопировали, а именно вырезали) все содержимое в новую папку на новый диск.

6. Идем редактировать файл */etc/postgresql/14/main/postgresql.conf,* находим в нем параметр *data_directory* и меняем на актуальную директорию
`data_directory = '/home/alexus/pg_tablespaсes/14/main'`
7. Запускаем кластер
`systemctl start postgresql@14-main`
Проверяем статус
`systemctl status postgresql@14-main.service`
Результат
```bash
 postgresql@14-main.service - PostgreSQL Cluster 14-main
     Loaded: loaded (/lib/systemd/system/postgresql@.service; enabled-runtime; vendor preset: enabled)
     Active: active (running) since Fri 2022-07-22 07:45:51 UTC; 5s ago
    Process: 14829 ExecStart=/usr/bin/pg_ctlcluster --skip-systemctl-redirect 14-main start (code=exited, status=0/SUCCESS)
   Main PID: 14844 (postgres)
      Tasks: 7 (limit: 1065)
     Memory: 23.0M
     CGroup: /system.slice/system-postgresql.slice/postgresql@14-main.service
             ├─14844 /usr/lib/postgresql/14/bin/postgres -D /home/alexus/pg_tablespaces/14/main -c config_file=/etc/postgresql/14/main/postgresql.conf
             ├─14846 postgres: 14/main: checkpointer
             ├─14847 postgres: 14/main: background writer
             ├─14848 postgres: 14/main: walwriter
             ├─14849 postgres: 14/main: autovacuum launcher
             ├─14850 postgres: 14/main: stats collector
             └─14851 postgres: 14/main: logical replication launcher
июл 22 07:45:48 pg1 systemd[1]: Starting PostgreSQL Cluster 14-main...
июл 22 07:45:51 pg1 systemd[1]: Started PostgreSQL Cluster 14-main.
```
8. Проверим содержимое кластера
```bash
\l
Список баз данных
Имя    | Владелец | Кодировка | LC_COLLATE  |  LC_CTYPE   |     Права доступа
-----------+----------+-----------+-------------+-------------+-----------------------
postgres  | postgres | UTF8      | ru_RU.UTF-8 | ru_RU.UTF-8 |
template0 | postgres | UTF8      | ru_RU.UTF-8 | ru_RU.UTF-8 | =c/postgres + postgres=CTc/postgres
template1 | postgres | UTF8      | ru_RU.UTF-8 | ru_RU.UTF-8 | =c/postgres + postgres=CTc/postgres
test      | postgres | UTF8      | ru_RU.UTF-8 | ru_RU.UTF-8 |
(4 строки)
\c test
\dt
Список отношений
Схема  | Имя  |   Тип   | Владелец
--------+------+---------+----------
public | test | таблица | postgres
(1 строка)
select * from test;
i | amount
---+--------
1 |    100
2 |    500
2 строки)
```
Все данные на месте.
