# Настройка PostgreSQL
1. Склонировал ВМ с установленным PostgreSQL 14 в Ubuntu Server 22.04. Первая машина называется ubuntu1, вторая ubuntu2.

2. На ubuntu1 создал БД repl_test, в базе создал таблицу repl_test1:
```sql
create database repl_test;
create table repl_test1 as select 
  generate_series(1,10) as id,
  md5(random()::text)::char(10) as fio;
```

3. Проверим, что вставилось в таблицу
```sql
 id |    fio     
----+------------
  1 | 600a862ed8
  2 | c7feca635c
  3 | 1f19b1b494
  4 | a97af3ec6f
  5 | 07d733141c
  6 | 8f647da77e
  7 | a651828530
  8 | 6159c1fc3f
  9 | 06cdc34d50
 10 | 834f9b511a
(10 rows) 
```

4. Установим уровень репликации logical
```sql
ALTER SYSTEM SET wal_level = logical;
```
Перезагружу кластер
```bash
sudo systemctl restart postgresql@14-main.service
```
5. Создам публикацию таблицы
```sql
CREATE PUBLICATION test_pub FOR TABLE repl_test1;
```
Проверю статус публикации
```sql
repl_test=# \dRp+
                            Publication test_pub
  Owner   | All tables | Inserts | Updates | Deletes | Truncates | Via root 
----------+------------+---------+---------+---------+-----------+----------
 postgres | f          | t       | t       | t       | t         | f
Tables:
    "public.repl_test1"
```
6. На ВМ2 создам подписку на первую таблицу на ВМ1 с копированием данных
```sql
CREATE SUBSCRIPTION test_sub 
CONNECTION 'host=192.168.1.36 port=5432 user=postgres password=postgres dbname=repl_test' 
PUBLICATION test_pub WITH (copy_data = true);
```
Выдала ошибку, что на источнике не настроено подключение с ВМ2. Иду в pg_hba.conf и добавляю строки
```sql
host    repl_test       postgres        192.168.1.71/32         password
```
Рестартую кластер. Проверю публикацию, выбрав данные из таблицы
```sql
id |    fio     
----+------------
(0 rows)
```
Публикация не сработала. Иду читать логи кластера на ВМ2. В них вижу, что для репликации не хватает процессов max_worker_processes. В файле у меня max_worker_processes = 2, поставил 8, перезагрузил кластер. Проверяю репликацию, она работает.
```sql
select * from repl_test1;
 id |    fio     
----+------------
  1 | 600a862ed8
  2 | c7feca635c
  3 | 1f19b1b494
  4 | a97af3ec6f
  5 | 07d733141c
  6 | 8f647da77e
  7 | a651828530
  8 | 6159c1fc3f
  9 | 06cdc34d50
 10 | 834f9b511a
 (10 rows)
```
 
7. На ВМ1 вставляю в таблицу еще одну строку
```sql
insert into repl_test1 values (11, 'asdqwed');
```

На ВМ2 проверяю содержимое таблицы
```sql
select * from repl_test1;
 id |    fio     
----+------------
  1 | 600a862ed8
  2 | c7feca635c
  3 | 1f19b1b494
  4 | a97af3ec6f
  5 | 07d733141c
  6 | 8f647da77e
  7 | a651828530
  8 | 6159c1fc3f
  9 | 06cdc34d50
 10 | 834f9b511a
 11 | asdqwed   
(11 rows)
```

Репликация работает.

8. Аналогично настраивается репликация с ВМ2 на ВМ1 и на ВМ3 для подписок на таблицы на ВМ1.
