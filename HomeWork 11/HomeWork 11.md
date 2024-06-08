### Домашнее задание к уроку "Работа с индексами"


1. Поскольку подходящей БД для опытов у меня не было, а изобретать велосипед не хотелось, я нашёл в интернете демонстрационную базу "DVD Rental": https://www.postgresqltutorial.com/postgresql-getting-started/postgresql-sample-database/. В ней, конечно же, сразу присутствовали некоторые индексы, но это не помешало выполнению ДЗ. Все пункты "домашки" отработаны на таблице `film`:
```
HomeWork 11=# \d film
                                              Table "public.film"
      Column      |            Type             | Collation | Nullable |                Default
------------------+-----------------------------+-----------+----------+---------------------------------------
 film_id          | integer                     |           | not null | nextval('film_film_id_seq'::regclass)
 title            | character varying(255)      |           | not null |
 description      | text                        |           |          |
 release_year     | year                        |           |          |
 language_id      | smallint                    |           | not null |
 rental_duration  | smallint                    |           | not null | 3
 rental_rate      | numeric(4,2)                |           | not null | 4.99
 length           | smallint                    |           |          |
 replacement_cost | numeric(5,2)                |           | not null | 19.99
 rating           | mpaa_rating                 |           |          | 'G'::mpaa_rating
 last_update      | timestamp without time zone |           | not null | now()
 special_features | text[]                      |           |          |
 fulltext         | tsvector                    |           | not null |
Indexes:
    "film_pkey" PRIMARY KEY, btree (film_id)
    "film_fulltext_idx" gist (fulltext)
    "idx_fk_language_id" btree (language_id)
    "idx_title" btree (title)
Foreign-key constraints:
    "film_language_id_fkey" FOREIGN KEY (language_id) REFERENCES language(language_id) ON UPDATE CASCADE ON DELETE RESTRICT
Referenced by:
    TABLE "film_actor" CONSTRAINT "film_actor_film_id_fkey" FOREIGN KEY (film_id) REFERENCES film(film_id) ON UPDATE CASCADE ON DELETE RESTRICT
    TABLE "film_category" CONSTRAINT "film_category_film_id_fkey" FOREIGN KEY (film_id) REFERENCES film(film_id) ON UPDATE CASCADE ON DELETE RESTRICT
    TABLE "inventory" CONSTRAINT "inventory_film_id_fkey" FOREIGN KEY (film_id) REFERENCES film(film_id) ON UPDATE CASCADE ON DELETE RESTRICT
Triggers:
    film_fulltext_trigger BEFORE INSERT OR UPDATE ON film FOR EACH ROW EXECUTE FUNCTION tsvector_update_trigger('fulltext', 'pg_catalog.english', 'title', 'description')
    last_updated BEFORE UPDATE ON film FOR EACH ROW EXECUTE FUNCTION last_updated()
```
2. Создадим простой индекс. Мы видим, что поле `length` не проиндексировано. Исправим этот недостаток: 
```
HomeWork 11=# CREATE INDEX idx_film__length ON film(length);
CREATE INDEX
```
Выполним пару-тройку запросов, использующих этот индекс. Для начала посмотрим, как будет работать запрос, ищущий фильм минимальной продолжительности:

```
HomeWork 11=# EXPLAIN ANALYZE SELECT min(length) FROM film;
 Result  (cost=0.52..0.53 rows=1 width=2) (actual time=0.049..0.050 rows=1 loops=1)
   InitPlan 1 (returns $0)
     ->  Limit  (cost=0.28..0.52 rows=1 width=2) (actual time=0.046..0.047 rows=1 loops=1)
           ->  Index Only Scan using idx_film__length on film  (cost=0.28..249.23 rows=1000 width=2) (actual time=0.045..0.045 rows=1 loops=1)
                 Index Cond: (length IS NOT NULL)
                 Heap Fetches: 1
 Planning Time: 0.288 ms
 Execution Time: 0.068 ms
(8 rows)

```
Тут всё понятно: для поиска единичного значения задействуется индекс. Для дальнейших опытов найдём наименьшую и наибольшую продолжительность среди всех фильмов:
```
HomeWork 11=# SELECT min(length), max(length) FROM film;
 min | max
-----+-----
  46 | 185
(1 row)
```
Посмотрим, как будет работать запрос, вычисляющий количество фильмов с продолжительностью меньше средней. Здесь нами предполагается, что длительность кинокартин распределена более-менее равномерно, а следовательно таких фильмов будет примерно 50% от общего количества, а значит, запрос не должен использовать индекс:
```
HomeWork 11=# EXPLAIN ANALYZE SELECT count(film_id) FROM film WHERE length < 115;
                                               QUERY PLAN
---------------------------------------------------------------------------------------------------------
 Aggregate  (cost=67.76..67.77 rows=1 width=8) (actual time=0.288..0.289 rows=1 loops=1)
   ->  Seq Scan on film  (cost=0.00..66.50 rows=503 width=4) (actual time=0.011..0.259 rows=504 loops=1)
         Filter: (length < 115)
         Rows Removed by Filter: 496
 Planning Time: 0.134 ms
 Execution Time: 0.311 ms
(6 rows)
```
Так и есть. Видно, что был выполнен последовательный просмотр всей таблицы. К тому же, исходное предположение оказалось верным: фильмов с такой продолжительностью 504 из 1000, имеющихся в таблице.
А теперь полюбопытствуем, как будет работать запрос, выбирающй фильмы длиной менее 50 минут:
```
HomeWork 11=# EXPLAIN ANALYZE SELECT count(film_id) FROM film WHERE length < 50;
                                                           QUERY PLAN
---------------------------------------------------------------------------------------------------------------------------------
 Aggregate  (cost=46.99..47.00 rows=1 width=8) (actual time=0.061..0.062 rows=1 loops=1)
   ->  Bitmap Heap Scan on film  (cost=4.45..46.93 rows=23 width=4) (actual time=0.023..0.051 rows=28 loops=1)
         Recheck Cond: (length < 50)
         Heap Blocks: exact=17
         ->  Bitmap Index Scan on idx_film__length  (cost=0.00..4.45 rows=23 width=0) (actual time=0.015..0.015 rows=28 loops=1)
               Index Cond: (length < 50)
 Planning Time: 0.100 ms
 Execution Time: 0.095 ms
(8 rows)
```
Поскольку количество коротких фильмов чуть более 2%, для их поиска применяется более быстрый способ - индексное сканирование по битовой карте.

3. Реализация индекса для полнотекстового поиска. В таблице `film` присутствует поле `description` которое по этому поводу просится-таки быть проидексированным: `"CREATE INDEX idx_film__description ON film USING GIN ((to_tsvector('english', description)));"`\
Теперь поищем что-нибудь интересненькое:
```
HomeWork 11=# EXPLAIN ANALYZE SELECT film_id, title FROM film WHERE description @@ to_tsquery('beautiful & woman');
                                            QUERY PLAN
--------------------------------------------------------------------------------------------------
 Seq Scan on film  (cost=0.00..564.00 rows=1 width=19) (actual time=4.451..22.813 rows=5 loops=1)
   Filter: (description @@ to_tsquery('beautiful & woman'::text))
   Rows Removed by Filter: 995
 Planning Time: 2.011 ms
 Execution Time: 22.825 ms
(5 rows)

```
Как видим, запрос не торопится порекомендовать нам какой-нибуь подходящий фильм и перебирает таблицу построчно. Попробуем его ускорить:
```
HomeWork 11=# ALTER SYSTEM SET enable_seqscan = OFF;
ALTER SYSTEM
HomeWork 11=# SELECT pg_reload_conf();
 pg_reload_conf
----------------
 t
(1 row)

HomeWork 11=# EXPLAIN ANALYZE SELECT film_id, title FROM film WHERE description @@ to_tsquery('beautiful & woman');
                                                     QUERY PLAN
---------------------------------------------------------------------------------------------------------------------
 Seq Scan on film  (cost=10000000000.00..10000000564.00 rows=1 width=19) (actual time=47.109..64.191 rows=5 loops=1)
   Filter: (description @@ to_tsquery('beautiful & woman'::text))
   Rows Removed by Filter: 995
 Planning Time: 1.785 ms
 JIT:
   Functions: 4
   Options: Inlining true, Optimization true, Expressions true, Deforming true
   Timing: Generation 0.523 ms, Inlining 6.526 ms, Optimization 26.230 ms, Emission 9.728 ms, Total 43.008 ms
 Execution Time: 64.781 ms
(9 rows)
```
Результат обескураживает: несмотря на запрет последовательного сканирования, запрос по-прежнему выполняется с помощью перебора всей таблицы. Кроме этого, задействуется JIT, что приводит почти к трёхкратному увеличению времени его выполнения. Да, видимо не судьба посмотреть сегодня что-нибудь...\
Вернём настройку `enable_seqscan` к значению по умолчанию:
```
HomeWork 11=# ALTER SYSTEM RESET enable_seqscan;
ALTER SYSTEM
HomeWork 11=# SELECT pg_reload_conf();
 pg_reload_conf
----------------
 t
(1 row)
```

4. Индекс на часть таблицы. Для таблицы `film` затруднительно придумать ещё один полезный индекс, который помогал бы в поиске на ограниченном наборе строк, поэтому придумаем бесполезный индекс по полю `rental_duration` (скорее всего, это продолжительность проката). Значения в данном поле распределяются следующим образом:
```
HomeWork 11=# SELECT rental_duration, count(rental_duration) FROM film GROUP BY rental_duration ORDER BY rental_duration;
 rental_duration | count
-----------------+-------
               3 |   203
               4 |   203
               5 |   191
               6 |   212
               7 |   191
(5 rows)
```
Индекс будет выглядеть так: `"CREATE INDEX idx_film__rental_duration_partial ON film(rental_duration) WHERE rental_duration < 6;"`\
Что-нибудь вычислим с его использованием:
```
HomeWork 11=# EXPLAIN ANALYZE SELECT count(film_id) FROM film WHERE rental_duration < 4;
                                                                     QUERY PLAN                                                        
----------------------------------------------------------------------------------------------------------------------------------------------------
 Aggregate  (cost=62.77..62.78 rows=1 width=8) (actual time=0.129..0.130 rows=1 loops=1)
   ->  Bitmap Heap Scan on film  (cost=5.72..62.26 rows=203 width=4) (actual time=0.030..0.110 rows=203 loops=1)
         Recheck Cond: (rental_duration < 4)
         Heap Blocks: exact=54
         ->  Bitmap Index Scan on idx_film__rental_duration_partial  (cost=0.00..5.67 rows=203 width=0) (actual time=0.018..0.019 rows=203 loops=1)
               Index Cond: (rental_duration < 4)
 Planning Time: 0.115 ms
 Execution Time: 0.158 ms
(8 rows)
```
Ясно видно, что индекс вполне может гордиться своим участием в выполнении запроса.

4. Создадим индекс по двум полям `release_year` (год выхода) и `rating` (рейтинг Американской киноассоциации): `"CREATE INDEX idx_film__release_year_rating ON film(release_year,rating);"` \
Теперь найдём, сколько фильмов категории NC-17 ("Лицам до 18 лет просмотр запрещен"), выпущенных после 1980 года есть в прокате:

```
EXPLAIN ANALYZE SELECT count(film_id) FROM film WHERE release_year >= 1980 AND rating = 'NC-17';
QUERY PLAN                                                          
-----------------------------------------------------------------------------------------------------------------------------------------------
 Aggregate  (cost=114.83..114.84 rows=1 width=8) (actual time=0.097..0.098 rows=1 loops=1)
   ->  Bitmap Heap Scan on film  (cost=11.83..114.65 rows=74 width=4) (actual time=0.034..0.085 rows=76 loops=1)
         Recheck Cond: (((release_year)::integer >= 1980) AND (rating = 'NC-17'::mpaa_rating))
         Heap Blocks: exact=44
         ->  Bitmap Index Scan on idx_film__release_year_rating  (cost=0.00..11.81 rows=74 width=0) (actual time=0.024..0.024 rows=76 loops=1)
               Index Cond: (((release_year)::integer >= 1980) AND (rating = 'NC-17'::mpaa_rating))
 Planning Time: 0.120 ms
 Execution Time: 0.129 ms
```
Прекрасно видно, что новоиспечённый индекс помог нам в этом деле. А вот если мы немного поменяем условие на "сколько фильмов категории NC-17, а так же кинокартин, выпущенных после 1980 года есть в прокате", то увидим что запрос использует последовательный просмотр таблицы:
```
EXPLAIN ANALYZE SELECT count(film_id) FROM film WHERE release_year >= 1980 OR rating = 'NC-17';
                                                QUERY PLAN
----------------------------------------------------------------------------------------------------------
 Aggregate  (cost=123.22..123.23 rows=1 width=8) (actual time=0.406..0.406 rows=1 loops=1)
   ->  Seq Scan on film  (cost=0.00..122.00 rows=490 width=4) (actual time=0.041..0.379 rows=488 loops=1)
         Filter: (((release_year)::integer >= 1980) OR (rating = 'NC-17'::mpaa_rating))
         Rows Removed by Filter: 512
 Planning Time: 0.118 ms
 Execution Time: 0.432 ms
(6 rows)
```
Это происходит потому, что теперь таблица просматривается по двум независимым друг от друга условиям и тут нам наш индекс по двум полям не помощник. Причём такая ситуация будет наблюдаться, даже если мы создадим два отдельных индекса по полям `release_year` и `rating`. И только после того, как мы запретим последовательное сканирование, можно увидеть, что в запросе стали использоваться оба индекса:
```
HomeWork 11=# ALTER SYSTEM SET enable_seqscan=0;
ALTER SYSTEM
HomeWork 11=# SELECT pg_reload_conf();
 pg_reload_conf
----------------
 t
(1 row)

HomeWork 11=# EXPLAIN ANALYZE SELECT count(film_id) FROM film WHERE release_year >= 1980 OR rating = 'NC-17';
                                                                      QUERY PLAN                                                      
-------------------------------------------------------------------------------------------------------------------------------------------------------
 Aggregate  (cost=133.58..133.59 rows=1 width=8) (actual time=0.251..0.252 rows=1 loops=1)
   ->  Bitmap Heap Scan on film  (cost=16.90..132.36 rows=490 width=4) (actual time=0.077..0.214 rows=488 loops=1)
         Recheck Cond: (((release_year)::integer >= 1980) OR (rating = 'NC-17'::mpaa_rating))
         Heap Blocks: exact=55
         ->  BitmapOr  (cost=16.90..16.90 rows=564 width=0) (actual time=0.067..0.067 rows=0 loops=1)
               ->  Bitmap Index Scan on idx_film__release_year_rating  (cost=0.00..10.93 rows=354 width=0) (actual time=0.030..0.030 rows=354 loops=1)
                     Index Cond: ((release_year)::integer >= 1980)
               ->  Bitmap Index Scan on idx_film__rating  (cost=0.00..5.73 rows=210 width=0) (actual time=0.036..0.036 rows=210 loops=1)
                     Index Cond: (rating = 'NC-17'::mpaa_rating)
 Planning Time: 0.127 ms
 Execution Time: 0.295 ms
(11 rows)
```

5. Единственная сложность, с которой я столкнулся - это отсутствие реакции оптимизатора на запрет последовательного сканирования в случае запроса с использованием полнотекстового индекса. Предполагаю, что такое поведение вызвано размером таблицы - всего тысяча строк. Была бы она на 3-4 порядка больше, возможно индекс был бы задействован.