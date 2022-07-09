# Первое домашнее задание

Зашел под первой сессией, ВМ pg1:
ssh alexus@192.168.1.34

результат:
alexus@pg1:~$ 

Зашел под второй сессией
ssh alexus@192.168.1.34

результат:
alexus@pg1

Запустил в обоих сессиях postgresql
sudo su postgres
[sudo] password for alexus: 
postgres@pg1:/home/alexus$ psql
psql (12.11 (Ubuntu 12.11-0ubuntu0.20.04.1))
Type "help" for help.

postgres=#

Проверил значени параметра autocommit
postgres=# \echo :AUTOCOMMIT
on
postgres=#

Выключил AUTOCOMMIT

Создал БД
postgres=# create database pgdba_1;
CREATE DATABASE
postgres=# \c pgdba_1;
You are now connected to database "pgdba_1" as user "postgres".

Создал таблицу
postgres=# create table persons(id serial, first_name text, second_name text);
CREATE TABLE
postgres=# commit;
COMMIT
postgres=#

Вставил данные в таблицу
insert into persons(first_name, second_name) values('ivan', 'ivanov'); insert into persons(first_name, second_name) values('petr', 'petrov'); commit;
INSERT 0 1
INSERT 0 1
COMMIT

Проверил уровень изоляции
show transaction isolation level;
 transaction_isolation 
-----------------------
 read committed
(1 row)

В первой сессии вставил данные в таблицу persons
insert into persons(first_name, second_name) values('sergey', 'sergeev');
INSERT 0 1

во второй сессии делаю выборку из таблицы persons
select * from persons
;
 id | first_name | second_name 
----+------------+-------------
  1 | ivan       | ivanov
  2 | petr       | petrov
(2 rows)

Я не вижу новой строчки во второй сессии, так как в первой сессии транзакция еще не закоммитилась, изменения не зафиксированы. Это потому, что уровень изоляции read commited, то есть чтение только тех данных, транзакция которых закоммитилась.

Закоммитил изменения в первой сессии (commit;), сделал выборку во второй:
select * from persons;
 id | first_name | second_name 
----+------------+-------------
  1 | ivan       | ivanov
  2 | petr       | petrov
  3 | sergey     | sergeev
(3 rows)

Новые данные увидел, так как в первой сессии транзакция завершилась.

Начал новые но уже repeatable read транзации 

В первой сессии 
BEGIN TRANSACTION ISOLATION LEVEL REPEATABLE READ; insert into persons(first_name, second_name) values('ira', 'irova');
BEGIN
INSERT 0 1

Сделал select * from persons во второй сессии
BEGIN TRANSACTION ISOLATION LEVEL REPEATABLE READ;
select * from persons;
 id | first_name | second_name 
----+------------+-------------
  1 | ivan       | ivanov
  2 | petr       | petrov
  3 | sergey     | sergeev
  4 | sveta      | svetova
(4 rows)

Видите ли вы новую запись и если да то почему?
Не вижу, так как уровень изоляций repeatable read держит запись дольше, чем read commited. До завершения всех транзакций (в первой и во второй сессии).

Завершил транзакцию в первой сессии - commit;
pgdba_1=# commit;
COMMIT

Выбрал данные во второй сессии
select * from persons;
 id | first_name | second_name 
----+------------+-------------
  1 | ivan       | ivanov
  2 | petr       | petrov
  3 | sergey     | sergeev
  4 | sveta      | svetova
(4 rows)

Новые строчки не увидел, так как не завершена транзакция во второй сессии.

Завершил транзакцию во второй сессии - commit;
pgdba_1=# commit;
COMMIT

Сделал select * from persons во второй сессии

pgdba_1=# select * from persons;
 id | first_name | second_name 
----+------------+-------------
  1 | ivan       | ivanov
  2 | petr       | petrov
  3 | sergey     | sergeev
  4 | sveta      | svetova
  5 | ira        | irova
(5 rows)

Видите ли вы новую запись и если да то почему?
Запись увидел, так как завершились обе транзакции и данные стали доступны для чтениия.
