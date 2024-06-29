### Домашнее задание к уроку "Секционирование таблицы"

1. Для выполнения домашней работы была использована демонстрационная БД "Авиаперевозки" с годовым объёмом информации (https://edu.postgrespro.ru/demo-big.zip).

2. После загрузки данных был выполнен запрос, позволивший найти самую большую таблицу:
```
SELECT reltuples::bigint, pg_table_size(oid) tblsz, relname FROM pg_class WHERE relkind='r' and relnamespace::regnamespace::text='bookings' ORDER BY relname;
 reltuples |   tblsz   |      relname
-----------+-----------+-------------------
         9 |     16384 | aircrafts_data
       104 |     57344 | airports_data
   7925944 | 477577216 | boarding_passes
   2111110 | 110215168 | bookings
    214867 |  21528576 | flights
      1339 |     98304 | seats
   8391852 | 573079552 | ticket_flights
   2949716 | 404955136 | tickets
(8 rows)
```
Сразу становится ясно, что будем секционировать таблицу `ticket_flights`:
```
demo=# \d ticket_flights
                     Table "bookings.ticket_flights"
     Column      |         Type          | Collation | Nullable | Default
-----------------+-----------------------+-----------+----------+---------
 ticket_no       | character(13)         |           | not null |
 flight_id       | integer               |           | not null |
 fare_conditions | character varying(10) |           | not null |
 amount          | numeric(10,2)         |           | not null |
Indexes:
    "ticket_flights_pkey" PRIMARY KEY, btree (ticket_no, flight_id)
Check constraints:
    "ticket_flights_amount_check" CHECK (amount >= 0::numeric)
    "ticket_flights_fare_conditions_check" CHECK (fare_conditions::text = ANY (ARRAY['Economy'::character varying::text, 'Comfort'::character varying::text, 'Business'::character varying::text]))
Foreign-key constraints:
    "ticket_flights_flight_id_fkey" FOREIGN KEY (flight_id) REFERENCES flights(flight_id)
    "ticket_flights_ticket_no_fkey" FOREIGN KEY (ticket_no) REFERENCES tickets(ticket_no)
Referenced by:
    TABLE "boarding_passes" CONSTRAINT "boarding_passes_ticket_no_fkey" FOREIGN KEY (ticket_no, flight_id) REFERENCES ticket_flights(ticket_no, flight_id)
```

4. Наиболее уместно дробить таблицу по первичному ключу, но так как в нашем случае он является составным (ticket_no, flight_id), ограничимся секционированием по `ticket_no`. 
Определим максимальный и минимальный номер билета:
```
demo=# SELECT min(ticket_no), max(ticket_no) FROM bookings.ticket_flights;
      min      |      max
---------------+---------------
 0005432000284 | 0005435999873
(1 row)
```
Одним из вариантов разбивки может быть следующая схема: от 0005432000000 до 0005436000000 с шагом 5000000. В таком случае наша таблица будет содержать 8 секций:
```
demo=# CREATE TABLE ticket_flights_parted (LIKE ticket_flights INCLUDING ALL) PARTITION BY RANGE(ticket_no);
CREATE TABLE
demo=# CREATE TABLE ticket_flights_01 PARTITION OF ticket_flights_parted FOR VALUES FROM ('0005432000000') TO ('0005432500000');
CREATE TABLE
demo=# CREATE TABLE ticket_flights_02 PARTITION OF ticket_flights_parted FOR VALUES FROM ('0005432500000') TO ('0005433000000');
CREATE TABLE
demo=# CREATE TABLE ticket_flights_03 PARTITION OF ticket_flights_parted FOR VALUES FROM ('0005433000000') TO ('0005433500000');
CREATE TABLE
demo=# CREATE TABLE ticket_flights_04 PARTITION OF ticket_flights_parted FOR VALUES FROM ('0005433500000') TO ('0005434000000');
CREATE TABLE
demo=# CREATE TABLE ticket_flights_05 PARTITION OF ticket_flights_parted FOR VALUES FROM ('0005434000000') TO ('0005434500000');
CREATE TABLE
demo=# CREATE TABLE ticket_flights_06 PARTITION OF ticket_flights_parted FOR VALUES FROM ('0005434500000') TO ('0005435000000');
CREATE TABLE
demo=# CREATE TABLE ticket_flights_07 PARTITION OF ticket_flights_parted FOR VALUES FROM ('0005435000000') TO ('0005435500000');
CREATE TABLE
demo=# CREATE TABLE ticket_flights_08 PARTITION OF ticket_flights_parted FOR VALUES FROM ('0005435500000') TO ('0005436000000');
CREATE TABLE

``` 

5. Встaвим данные:
```
demo=# INSERT INTO ticket_flights_parted (ticket_no, flight_id, fare_conditions, amount) SELECT ticket_no, flight_id, fare_conditions, amount FROM ticket_flights;
INSERT 0 8391852
``` 
Как видим, всё прошло успешно. Посмотрим на количество вставленных строк по секциям:
```
demo=# SELECT reltuples::bigint, relname FROM pg_class WHERE relkind='r' and relnamespace::regnamespace::text='bookings' AND relname LIKE 'ticket_flights_0%' ORDER BY relname;
 reltuples |      relname
-----------+-------------------
    739266 | ticket_flights_01
   1092697 | ticket_flights_02
   1110931 | ticket_flights_03
   1120187 | ticket_flights_04
   1201921 | ticket_flights_05
   1063435 | ticket_flights_06
   1012556 | ticket_flights_07
   1050859 | ticket_flights_08
(8 rows)
``` 

6. После всех этих манипуляций возникает резонный вопрос: "А есть ли от них польза?" Попробуем сделать следующее: выберем из таблицы `bookings.tickets` некоторое количество пассажиров и вычислим суммарную стоимость их перелётов `sum(ticket_flights.amount)`. Этот запрос, скорее бесполезен с житейской точки зрения, но как оценка скорости работы с обычной и секционированной таблицами вполне нам подойдёт. Сам запрос выглядит так: `"WITH paxes AS (SELECT DISTINCT passenger_name FROM bookings.tickets LIMIT 50) SELECT sum(amount) FROM bookings.ticket_flights JOIN bookings.tickets USING (ticket_no) JOIN paxes USING(passenger_name);"`. Перед экспериментом выполним анализ таблиц, используемых в запросе: `"ANALYZE tickets, ticket_flights, ticket_flights_parted;"`. Для исключения влияния кеширования перед каждым запросом будем перезапускать СУБД. Поочерёдно выполним запрос для 50, 5000 и 10000 пассажиров, суммируя по каждой из таблиц: `ticket_flights` и `ticket_flights_parted`. Сведём полученные результаты:
```
            Table name      | Number of paxes | Execution time, ms
      ----------------------+-----------------+-------------------
             ticket_flights |             50  |           1285.946
      ticket_flights_parted |             50  |           1263.257
      ----------------------+-----------------+-------------------
             ticket_flights |           5000  |           7665.556
      ticket_flights_parted |           5000  |           7729.912
      ----------------------+-----------------+-------------------
             ticket_flights |          10000  |           7841.306
      ticket_flights_parted |          10000  |           7952.700
      ----------------------+-----------------+-------------------      
```
Никаких существенных различий не наблюдается. Возможно, дело в неудачном запросе или же в том, что в таблицах не так уж и много строк. В общем, можно сказать, что само секционирование прошло удачно, но оказалось бесполезным.