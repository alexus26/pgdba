# Работа с индексами
1. Подключился к ВМ с Ubuntu 22.04 и PostgreSQL14.

2. Cоздал БД index_test:
```sql
create table index_test as select 
  generate_series(1,1000) as id,
  md5(random()::text)::char(10) as fio, 
  md5(random()::text)::char(10) as fio1, 
  md5(random()::text)::char(10) as fio2, 
  md5(random()::text)::char(10) as fio3;
SELECT 1000
```

3. Посмотрим, есть ли индексы в таблице
```sql
repl_test=#\d index_test 
                Table "public.index_test"
 Column |     Type      | Collation | Nullable | Default 
--------+---------------+-----------+----------+---------
 id     | integer       |           |          | 
 fio    | character(10) |           |          | 
 fio1   | character(10) |           |          | 
 fio2   | character(10) |           |          | 
 fio3   | character(10) |           |          |
```
Как видно, индексов в таблице нет.

4. Проверим через команду explain как работает выборка значений с фильтрацией
```sql
repl_test=# explain select * from index_test where id < 500;
                          QUERY PLAN                          
--------------------------------------------------------------
 Seq Scan on index_test  (cost=0.00..22.50 rows=499 width=48)
   Filter: (id < 500)
(2 rows)
```
Как видно, используется последовательное сканирование (Seq Scan on index_test)

5. Создадим индекс на таблицу index_test на колонку id
```sql
repl_test=# create index uk_index_test_id on index_test(id);
CREATE INDEX
```

6. Проверим, используются ли индексы
```sql
repl_test=# explain select * from index_test where id < 500;
QUERY PLAN                                       
---------------------------------------------------------------------------------------
 Index Scan using uk_index_test_id on index_test  (cost=0.28..17.41 rows=499 width=48)
   Index Cond: (id < 500)
(2 rows)
```
Как видно, используется сканирование индекса (Index Scan using uk_index_test_id on index_test)
 
7. Выберем из таблицы строчки, в которых fio и fio1 удовлетворяют условию
```sql
select * from index_test where fio = '6e2f836664' and fio1 = 'd59bac6c2f';
 id  |    fio     |    fio1    |    fio2    |    fio3    
-----+------------+------------+------------+------------
 500 | 6e2f836664 | d59bac6c2f | 37279e8c26 | 6516cc7aed
(1 row)
```

Проверим, как выполняется данная команда
```sql
repl_test=# explain select * from index_test where fio = '6e2f836664' and fio1 = 'd59bac6c2f';
                                 QUERY PLAN                                 
----------------------------------------------------------------------------
 Seq Scan on index_test  (cost=0.00..25.00 rows=1 width=48)
   Filter: ((fio = '6e2f836664'::bpchar) AND (fio1 = 'd59bac6c2f'::bpchar))
(2 rows)
```

8. Создадим составной индекс на колонки fio1 и fio, проверим, как теперь выполняется поиск и выборка значений
```sql
repl_test=#create index index_test_fio_fio1 on index_test(fio) include (fio1);
CREATE INDEX

repl_test=# explain select * from index_test where fio = '6e2f836664' and fio1 = 'd59bac6c2f';
                                      QUERY PLAN                                       
---------------------------------------------------------------------------------------
 Index Scan using index_test_fio_fio1 on index_test  (cost=0.28..2.50 rows=1 width=48)
   Index Cond: (fio = '6e2f836664'::bpchar)
   Filter: (fio1 = 'd59bac6c2f'::bpchar)
(3 rows)
```
Как видно, используется составной индекс.

9. Создадим индекс на часть таблицы. Проверим, как теперь выполняется поиск с условиями
```sql
repl_test=# create index partindex_test_id_400 on index_test(id) where id > 400;
CREATE INDEX
repl_test=# explain select * from index_test where id < 400;
                          QUERY PLAN                          
--------------------------------------------------------------
 Seq Scan on index_test  (cost=0.00..22.50 rows=399 width=48)
   Filter: (id < 400)
(2 rows)

repl_test=# explain select * from index_test where id > 400;
                                         QUERY PLAN                                         
--------------------------------------------------------------------------------------------
 Index Scan using partindex_test_id_400 on index_test  (cost=0.28..19.77 rows=600 width=48)
(1 row)
```

10. Модифицируем таблицу для того, чтобы проверить работу полнотекстового индекса
```sql
repl_test=# alter table index_test add column some_text_lexeme tsvector;
ALTER TABLE
repl_test=# update index_test set some_text_lexeme = to_tsvector(fio2);
UPDATE 1000
repl_test=# explain select some_text_lexeme from index_test where some_text_lexeme @@ to_tsvector('d955cbaf86');
                          QUERY PLAN                          
--------------------------------------------------------------
 Seq Scan on index_test  (cost=0.00..534.50 rows=10 width=23)
   Filter: ((fio2)::text @@ to_tsquery('d955cbaf86'::text))
(2 rows)
```
Как видно, используется последовательно сканирование колонки в поисках значения.

11. Создадим индекс для полнотекстового поиска
```sql
repl_test=# CREATE INDEX search_index_ord ON index_test USING GIN (some_text_lexeme);
CREATE INDEX
repl_test=# explain select some_text_lexeme from index_test where some_text_lexeme @@ to_tsquery('d955cbaf86');
                                  QUERY PLAN                                   
-------------------------------------------------------------------------------
 Bitmap Heap Scan on index_test  (cost=2.46..3.82 rows=1 width=23)
   Recheck Cond: (some_text_lexeme @@ to_tsquery('d955cbaf86'::text))
   ->  Bitmap Index Scan on search_index_ord  (cost=0.00..2.46 rows=1 width=0)
         Index Cond: (some_text_lexeme @@ to_tsquery('d955cbaf86'::text))
(4 rows)
```

После создания индекса при поиске начала использоваться битовая карта для поиска нужной страницы, которая содержит искомую строку.

# JOINS
1. Запросы буду выполнять к базе flights среднего размера.

2. Прямое соединение. Выберу названия самолетов и аэропортов
```sql
select a.model, ad.airport_name 
from bookings.flights f 
join bookings.aircrafts_data a 
on f.aircraft_code = a.aircraft_code
join bookings.airports_data ad 
on f.arrival_airport = ad.airport_code 
limit 10;

|model                                                     |airport_name                                                  |
|----------------------------------------------------------|--------------------------------------------------------------|
|{"en": "Airbus A321-200", "ru": "Аэробус A321-200"}       |{"en": "Pulkovo Airport", "ru": "Пулково"}                    |
|{"en": "Cessna 208 Caravan", "ru": "Сессна 208 Караван"}  |{"en": "Yoshkar-Ola Airport", "ru": "Йошкар-Ола"}             |
|{"en": "Sukhoi Superjet-100", "ru": "Сухой Суперджет-100"}|{"en": "Bryansk Airport", "ru": "Брянск"}                     |
|{"en": "Bombardier CRJ-200", "ru": "Бомбардье CRJ-200"}   |{"en": "Saratov Central Airport", "ru": "Саратов-Центральный"}|
|{"en": "Boeing 777-300", "ru": "Боинг 777-300"}           |{"en": "Tolmachevo Airport", "ru": "Толмачёво"}               |
|{"en": "Cessna 208 Caravan", "ru": "Сессна 208 Караван"}  |{"en": "Petrozavodsk Airport", "ru": "Бесовец"}               |
|{"en": "Sukhoi Superjet-100", "ru": "Сухой Суперджет-100"}|{"en": "Ulyanovsk Baratayevka Airport", "ru": "Баратаевка"}   |
|{"en": "Airbus A321-200", "ru": "Аэробус A321-200"}       |{"en": "Domodedovo International Airport", "ru": "Домодедово"}|
|{"en": "Airbus A321-200", "ru": "Аэробус A321-200"}       |{"en": "Vnukovo International Airport", "ru": "Внуково"}      |
|{"en": "Cessna 208 Caravan", "ru": "Сессна 208 Караван"}  |{"en": "Cherepovets Airport", "ru": "Череповец"}              |
```

В этом из таблицы с рейсами в запросе по коду самолета, прибывающего в аэропорт, получаем модель самолета, а по коду аэропорта - название аэропорта.

3. Левостороннее соединение. Выберу все города, в которых больше 1 аэропорта.
```sql
with small_towns as
			(
				SELECT   aa.city as small_city
	            FROM     airports aa
	            GROUP BY aa.city
	            HAVING   COUNT(*) = 1
	        )
select a.airport_code as code,
         a.airport_name,
         a.city,
         a.coordinates
FROM     airports a
left join small_towns st 
on a.city = st.small_city
where st.small_city is null
ORDER BY a.city, a.airport_code;

|code|airport_name       |city     |coordinates                           |
|----|-------------------|---------|--------------------------------------|
|DME |Домодедово         |Москва   |(37.90629959106445,55.40879821777344) |
|SVO |Шереметьево        |Москва   |(37.4146,55.972599)                   |
|VKO |Внуково            |Москва   |(37.2615013123,55.5914993286)         |
|ULV |Баратаевка         |Ульяновск|(48.226699829100006,54.26829910279999)|
|ULY |Ульяновск-Восточный|Ульяновск|(48.80270004272461,54.4010009765625)  |

```

4. Кросс-соединение. Выберу все места и все самолеты
```sql 
select s.seat_no, s.fare_conditions, ad.model 
from bookings.aircrafts_data ad
cross join bookings.seats s;
```

В результате у нас будет очень много строчек. Кросс-соединение по сути "перемножает" все строки одной таблицы со строками другой таблицы.

5. Полное соединение. Выведем все данные из таблиц с местами и самолетами, соединив таблицы по коду самолета.
```sql
select * from seats s 
full join aircrafts_data ad 
on s.aircraft_code = ad.aircraft_code 
where ad.aircraft_code = '773'
order by seat_no;

|aircraft_code|seat_no|fare_conditions|aircraft_code|model                                          |range |
|-------------|-------|---------------|-------------|-----------------------------------------------|------|
|773          |11A    |Comfort        |773          |{"en": "Boeing 777-300", "ru": "Боинг 777-300"}|11 100|
|773          |11C    |Comfort        |773          |{"en": "Boeing 777-300", "ru": "Боинг 777-300"}|11 100|
|773          |11D    |Comfort        |773          |{"en": "Boeing 777-300", "ru": "Боинг 777-300"}|11 100|
|773          |11E    |Comfort        |773          |{"en": "Boeing 777-300", "ru": "Боинг 777-300"}|11 100|
|773          |11F    |Comfort        |773          |{"en": "Boeing 777-300", "ru": "Боинг 777-300"}|11 100|
|773          |11G    |Comfort        |773          |{"en": "Boeing 777-300", "ru": "Боинг 777-300"}|11 100|
|773          |11H    |Comfort        |773          |{"en": "Boeing 777-300", "ru": "Боинг 777-300"}|11 100|
|773          |11K    |Comfort        |773          |{"en": "Boeing 777-300", "ru": "Боинг 777-300"}|11 100|
|773          |12A    |Comfort        |773          |{"en": "Boeing 777-300", "ru": "Боинг 777-300"}|11 100|
|773          |12C    |Comfort        |773          |{"en": "Boeing 777-300", "ru": "Боинг 777-300"}|11 100|
```

6. Разные типы соединений. Выберем все аэропорты, которые содержат больше 1 аэропорта и все самолеты, которые в них летают
```sql
with small_towns as
			(
				SELECT   aa.city as small_city
	            FROM     airports aa
	            GROUP BY aa.city
	            HAVING   COUNT(*) = 1
	        )
select   distinct a.airport_name,
         a2.model 
FROM     airports a
left join small_towns st 
on a.city = st.small_city
inner join flights f 
on a.airport_code = f.arrival_airport 
inner join aircrafts a2 
on f.aircraft_code = a2.aircraft_code 
where st.small_city is null
order by a.airport_name;

|airport_name       |model              |
|-------------------|-------------------|
|Баратаевка         |Боинг 767-300      |
|Баратаевка         |Сессна 208 Караван |
|Баратаевка         |Сухой Суперджет-100|
|Внуково            |Аэробус A321-200   |
|Внуково            |Сухой Суперджет-100|
|Внуково            |Бомбардье CRJ-200  |
|Внуково            |Боинг 777-300      |
|Внуково            |Аэробус A319-100   |
|Внуково            |Сессна 208 Караван |
|Внуково            |Боинг 767-300      |
|Домодедово         |Боинг 777-300      |
|Домодедово         |Сухой Суперджет-100|
|Домодедово         |Аэробус A321-200   |
|Домодедово         |Боинг 737-300      |
|Домодедово         |Сессна 208 Караван |
|Домодедово         |Бомбардье CRJ-200  |
|Домодедово         |Аэробус A319-100   |
|Домодедово         |Боинг 767-300      |
|Ульяновск-Восточный|Сессна 208 Караван |
|Ульяновск-Восточный|Бомбардье CRJ-200  |
|Шереметьево        |Боинг 737-300      |
|Шереметьево        |Сессна 208 Караван |
|Шереметьево        |Боинг 767-300      |
|Шереметьево        |Сухой Суперджет-100|
|Шереметьево        |Аэробус A319-100   |
|Шереметьево        |Аэробус A321-200   |
|Шереметьево        |Боинг 777-300      |
|Шереметьево        |Бомбардье CRJ-200  |
```
