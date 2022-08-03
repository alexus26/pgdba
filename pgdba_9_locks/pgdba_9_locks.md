# Блокировки
1. Создал базу accounts и наполнил ее данными
```sql
CREATE TABLE accounts(acc_no integer PRIMARY KEY, amount numeric);
INSERT INTO accounts VALUES (1, 1000.00), (2, 2000.00), (3, 3000.00) (4, 4000.00) (5, 5000.00);
```
2. Проверил pid текущего сеанса
```sql
SELECT pg_backend_pid();
pg_backend_pid
----------------
           1431
(1 row)
```
3. Запустил второй сеанс, проверил его pid
```sql
SELECT pg_backend_pid();
pg_backend_pid
----------------
           1416
(1 row)
```

4. Запустил в первом сеансе обновление колонки с acc_no = 5
```sql
begin; update accounts set amount = amount + 100 where acc_no = 5;
```

5. Запустил во втором терминале создание индекса по таблице. Транзакция подвисла. Проверяю блокировки, которые поставил сеанс с pid = 1431 (обновление таблицы)
```sql
SELECT locktype, relation::REGCLASS, virtualxid AS virtxid, transactionid AS xid, mode, granted
FROM pg_locks WHERE pid = 1431;
```
Результат
```sql
   locktype    |      relation       | virtxid |   xid   |       mode       | granted 
---------------+---------------------+---------+---------+------------------+---------
 relation      | accounts_acc_no_idx |         |         | RowExclusiveLock | t
 relation      | accounts_pkey       |         |         | RowExclusiveLock | t
 virtualxid    |                     | 3/52    |         | ExclusiveLock    | t
 transactionid |                     |         | 8974659 | ExclusiveLock    | t
 relation      | accounts            |         |         | RowExclusiveLock | t
(5 rows)
```
Здесь наблюдаем 5 блокировок
- На таблицы accounts_acc_no_idx и accounts_pkey в режиме RowExclusiveLock (на колонку с индексом).
- На строку с номером своего собственного номера сеанса virtualxid в режиме ExclusiveLock;
- На строку с транзакцией с xid 8974659 в режиме ExclusiveLock;
- На таблицу accounts в режиме режиме ExclusiveLock;

6. Проверим, какие блокировки есть во втором сеансе
```sql
SELECT locktype, relation::REGCLASS, virtualxid AS virtxid, transactionid AS xid, mode, granted
FROM pg_locks WHERE pid = 1431;
postgres=# SELECT locktype, relation::REGCLASS, virtualxid AS virtxid, transactionid AS xid, mode, granted
FROM pg_locks WHERE pid = 1416;
  locktype  | relation | virtxid | xid |     mode      | granted 
------------+----------+---------+-----+---------------+---------
 virtualxid |          | 4/18    |     | ExclusiveLock | t
 relation   | accounts |         |     | ShareLock     | f
(2 rows)
```
Тут всего 2 блокировки: одна на строчку с собственным сеансом, вторая на таблицу accounts в попытке создать индекс в режиме ShareLock. Но так как в первом сеансе мы заблокировали всю таблицу в режиме ExclusiveLock, то блокировка в режиме ShareLock во втором сеансе не удалась (флаг f).
7. Взаимоблокировку настроить не смог. 
8. Журнал настроить смог, в процессе тестировантия была такая информация
```bash
tail -n 7 /var/log/postgresql/postgresql-14-main.log
2022-08-03 15:21:04.964 UTC [1416] postgres@postgres LOG:  process 1416 still waiting for ShareLock on transaction 8974664 after 1000.908 ms
2022-08-03 15:21:04.964 UTC [1416] postgres@postgres DETAIL:  Process holding the lock: 1431. Wait queue: 1416.
2022-08-03 15:21:04.964 UTC [1416] postgres@postgres CONTEXT:  while updating tuple (0,13) in relation "accounts"
2022-08-03 15:21:04.964 UTC [1416] postgres@postgres STATEMENT:  update accounts set amount = amount + 100 where acc_no = 5;
2022-08-03 15:21:27.780 UTC [1416] postgres@postgres LOG:  process 1416 acquired ShareLock on transaction 8974664 after 23816.919 ms
2022-08-03 15:21:27.780 UTC [1416] postgres@postgres CONTEXT:  while updating tuple (0,13) in relation "accounts"
```

Т.е. в журнал занеслась информация о том, какая блокировка и на что была установлена и когда снята.
