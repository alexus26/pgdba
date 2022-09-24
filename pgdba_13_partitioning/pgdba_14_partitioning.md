# Секционирование
1. Подключился к ВМ с Ubuntu 22.04 и PostgreSQL14.

2. Проведем секционирование таблицы `ticket_flights` по колонке `fare_conditions`
```sql
create table fare_condition_business(like ticket_flights including all) inherits (ticket_flights);
alter table fare_condition_business add check (fare_conditions = 'Business');
```

3. Создадим триггерную функцию для вставки значений в секцию таблицы
```sql
СREATE OR REPLACE FUNCTION ticket_flights_insert_trigger()
RETURNS TRIGGER AS $$
BEGIN
    IF ( NEW.fare_conditions = 'Business') THEN
        INSERT INTO fare_condition_business VALUES (NEW.*);
    END IF;
    RETURN NULL;
END;
$$
LANGUAGE plpgsql;
```

4. Проверим вставку значений в таблицу `ticket_flights`
```sql
INSERT INTO bookings.ticket_flights
(ticket_no, flight_id, fare_conditions, amount)
VALUES('0005432000860', 57376, 'Business', 18500);
```
Результат
```sql
|ticket_no    |flight_id|fare_conditions|amount|
|-------------|---------|---------------|------|
|0005432000860|57 376   |Business       |18 500|
```

5. Проверим, как выполняется поиск по таблице `ticket_flights`
```sql
explain analyze
select * from ticket_flights tf
where fare_conditions = 'Business'

Append  (cost=0.00..50404.93 rows=242724 width=32) (actual time=5.508..240.029 rows=242204 loops=1)
  ->  Seq Scan on ticket_flights tf_1  (cost=0.00..49174.19 rows=242721 width=32) (actual time=5.507..226.761 rows=242204 loops=1)
        Filter: ((fare_conditions)::text = 'Business'::text)
        Rows Removed by Filter: 2118131
  ->  Seq Scan on fare_condition_business tf_2  (cost=0.00..17.12 rows=3 width=114) (actual time=0.007..0.007 rows=0 loops=1)
        Filter: ((fare_conditions)::text = 'Business'::text)
Planning Time: 0.732 ms
Execution Time: 246.106 ms
```
Как видно, у нас добавился поиск по секции `fare_condition_business tf_2`

6. Проведем секционирование декларативно. Для этого возьмем эту же таблицу `ticket_flights`. Но, при попытке создать секцию будет выдано предупреждение, что таблица не имеет секционирования. Поэтому создадим новую таблицу с включенным секционированием по колонке `fare_conditions`.
```sql
create table bookings.ticket_flights_new (
	ticket_no bpchar(13) NOT NULL,
	flight_id int4 NOT NULL,
	fare_conditions varchar(10) NOT NULL,
	amount numeric(10, 2) NOT NULL)
partition by list (fare_conditions);

create table fc_economy partition of bookings.ticket_flights_new for values in ('Economy');
create table fc_comfort partition of bookings.ticket_flights_new for values in ('Comfort');
create table fc_business partition of bookings.ticket_flights_new for values in ('Business');

insert into bookings.ticket_flights_new (select * from ticket_flights);
```

 
7. Проверим план запроса
```sql
explain analyze
select * from ticket_flights_new tf
where fare_conditions = 'Business'

Seq Scan on fc_business tf  (cost=0.00..5046.55 rows=242204 width=33) (actual time=0.017..50.972 rows=242204 loops=1)
  Filter: ((fare_conditions)::text = 'Business'::text)
Planning Time: 0.138 ms
Execution Time: 63.284 ms
```
Как видно, план запроса строится не по исходной таблице, а по ее секции, для которой `fare_conditions` = `'Business'`
