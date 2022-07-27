# Логический уровень PostgreSQL
1. Создал ВМ в VirtualBox, поставил PG14.
`14  main    5432 online postgres /home/alexus/pg_tablespaces/14/main /var/log/postgresql/postgresql-14-main.log`
2. зайдите в созданный кластер под пользователем postgres
```bash
sudo su postgres
[sudo] password for alexus: 
postgres@pg1:/home/alexus$ pg_lsclusters
```
3. создайте новую базу данных testdb
```sql
create database testdb;
CREATE DATABASE
```
4. зайдите в созданную базу данных под пользователем postgres
```sql
\c testdb;
Вы подключены к базе данных "testdb" как пользователь "postgres".
```
5. создайте новую схему testnm
```sql
testdb=# create schema testnm;
CREATE SCHEMA
```
6. создайте новую таблицу t1 с одной колонкой c1 типа integer
```sql
testdb=# CREATE TABLE t1(c1 int);
CREATE TABLE
```
7. вставьте строку со значением c1=1
```sql
testdb=# insert into t1(c1) values(1);
INSERT 0 1
```
8. создайте новую роль readonly
```sql
create role role1;
CREATE ROLE
```
9. дайте новой роли право на подключение к базе данных testdb
```sql
testdb=# grant connect on database testdb to role1;
GRANT
```
10. дайте новой роли право на использование схемы testnm
```sql
grant usage on schema testnm to role1;
GRANT
```
11. дайте новой роли право на select для всех таблиц схемы testnm
```sql
grant select on all tables in schema testnm to role1;
GRANT
```
12. создайте пользователя testread с паролем test123
```sql
postgres@pg1:/home/alexus$ createuser --interactive
Введите имя новой роли: testread
Должна ли новая роль иметь полномочия суперпользователя? (y - да/n - нет) n
Новая роль должна иметь право создавать базы данных? (y - да/n - нет) n
Новая роль должна иметь право создавать другие роли? (y - да/n - нет) n
postgres=# alter user testread with password 'test123';
ALTER ROLE
```
13. дайте роль readonly пользователю testread
```sql
grant role1 to testread;
GRANT ROLE
postgres=# \du
                                          Список ролей
 Имя роли |                                Атрибуты                                 | Член ролей
 ----------+-------------------------------------------------------------------------+------------
 postgres | Суперпользователь, Создаёт роли, Создаёт БД, Репликация, Пропускать RLS | {}
 role1       | Вход запрещён                                                            | {}
 testread  |                                                                         | {role1}

```
14. зайдите под пользователем testread в базу данных testdb
```sql
sudo -u postgres psql -U testread -h localhost -W -d postgres
Пароль: 
psql (14.4 (Ubuntu 14.4-1.pgdg20.04+1))
SSL-соединение (протокол: TLSv1.3, шифр: TLS_AES_256_GCM_SHA384, бит: 256, сжатие: выкл.)
Введите "help", чтобы получить справку.

postgres=> \c testdb
Пароль: 
SSL-соединение (протокол: TLSv1.3, шифр: TLS_AES_256_GCM_SHA384, бит: 256, сжатие: выкл.)
Вы подключены к базе данных "testdb" как пользователь "testread".
testdb=>
```
15. сделайте select * from t1;
Ошибка
16. получилось? (могло если вы делали сами не по шпаргалке и не упустили один существенный момент про который позже)
Не получилось.
17. напишите что именно произошло в тексте домашнего задания
Не получилось, написал, что нет такой таблицы. 
18. у вас есть идеи почему? ведь права то дали?
Таблица создана в схеме public, а не в testnm;
19. посмотрите на список таблиц
Ничего не выводит
20. подсказка в шпаргалке под пунктом 20
21. а почему так получилось с таблицей (если делали сами и без шпаргалки то может у вас все нормально)
22. вернитесь в базу данных testdb под пользователем postgres
Удалил вообще все данные, все роли.
23. удалите таблицу t1 - удалил
24. создайте ее заново но уже с явным указанием имени схемы testnm - создал
```sql
create table testnm.t1
```
26. вставьте строку со значением c1=1
```sql
insert into testnm.t1 values(1)
```
28. зайдите под пользователем testread в базу данных testdb
29. сделайте select * from testnm.t1;
Вот тут все заработало нормально.
28. получилось?
Да
29. есть идеи почему? если нет - смотрите шпаргалку
Потому что обратились к таблице через указание схемы.
30. как сделать так чтобы такое больше не повторялось? если нет идей - смотрите шпаргалку
31. сделайте select * from testnm.t1;
Ошибка
32. получилось?
Нет
33. есть идеи почему? если нет - смотрите шпаргалку
Смотрел в шпаргалку
31. сделайте select * from testnm.t1;
32. получилось?
Да
33. ура!
34. теперь попробуйте выполнить команду create table t2(c1 integer); insert into t2 values (2);
```bash
testdb=> create table t2(c1 integer); insert into t2 values (2);
CREATE TABLE
INSERT 0 1
```
35. а как так? нам же никто прав на создание таблиц и insert в них под ролью readonly?
Смотрел шпаргалку.
36. есть идеи как убрать эти права? если нет - смотрите шпаргалку
Смотрел шпаргалку
37. если вы справились сами то расскажите что сделали и почему, если смотрели шпаргалку - объясните что сделали и почему выполнив указанные в ней команды
38. теперь попробуйте выполнить команду create table t3(c1 integer); insert into t2 values (2);
```bash
testdb=> create table t3(c1 integer); insert into t4 values (2);
ОШИБКА:  нет доступа к схеме public
СТРОКА 1: create table t4(c1 integer);
                       ^
ОШИБКА:  отношение "t4" не существует
СТРОКА 1: insert into t4 values (2);
```
39. расскажите что получилось и почему 
Получилось все.
