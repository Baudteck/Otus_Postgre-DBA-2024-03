### Домашнее задание к уроку "Работа с JOIN, статистикой"


1. Для выполнения домашней работы использовалась демонстрационная база "DVD Rental": https://www.postgresqltutorial.com/postgresql-getting-started/postgresql-sample-database/. Описание создания и настройки кластера, загрузки данных в базу и тому подобные процедуры опустим.

2. Прямое соединение двух или более таблиц. Создав запрос по таблицам `film`, `film_category`  и `category`, можно узнать жанр той или иной кинокартины. Из-за того, что в таблице `film` много строк, ограничим получаемый результат запроса:
```
SELECT f.title, f.description, c.name category FROM film f JOIN film_category fc USING (film_id) JOIN category c USING (category_id) ORDER BY f.title LIMIT 10;
      title       |                                                      description                                                      |   category
------------------+-----------------------------------------------------------------------------------------------------------------------+-------------
 Academy Dinosaur | A Epic Drama of a Feminist And a Mad Scientist who must Battle a Teacher in The Canadian Rockies                      | Documentary
 Ace Goldfinger   | A Astounding Epistle of a Database Administrator And a Explorer who must Find a Car in Ancient China                  | Horror
 Adaptation Holes | A Astounding Reflection of a Lumberjack And a Car who must Sink a Lumberjack in A Baloon Factory                      | Documentary
 Affair Prejudice | A Fanciful Documentary of a Frisbee And a Lumberjack who must Chase a Monkey in A Shark Tank                          | Horror
 African Egg      | A Fast-Paced Documentary of a Pastry Chef And a Dentist who must Pursue a Forensic Psychologist in The Gulf of Mexico | Family
 Agent Truman     | A Intrepid Panorama of a Robot And a Boy who must Escape a Sumo Wrestler in Ancient China                             | Foreign
 Airplane Sierra  | A Touching Saga of a Hunter And a Butler who must Discover a Butler in A Jet Boat                                     | Comedy
 Airport Pollock  | A Epic Tale of a Moose And a Girl who must Confront a Monkey in Ancient India                                         | Horror
 Alabama Devil    | A Thoughtful Panorama of a Database Administrator And a Mad Scientist who must Outgun a Mad Scientist in A Jet Boat   | Horror
 Aladdin Calendar | A Action-Packed Tale of a Man And a Lumberjack who must Reach a Feminist in Ancient China                             | Sports
(10 rows)
```
3. Лево- или правосторонее соединение двух и более таблиц. Поскольку информация в базе отличается полнотой, пришлось поискать пару таблиц, для которых данные, выводимые INNER JOIN, отличаются от данных, получаемых LEFT JOIN'ом:
```
 SELECT c.city_id, c.city, a.address FROM city c LEFT JOIN address a ON c.city_id=a.city_id WHERE c.city = 'London' ORDER BY city, address;
 city_id |  city  |      address
---------+--------+--------------------
     312 | London | 1497 Yuzhou Drive
     312 | London | 548 Uruapan Street
     313 | London |
(3 rows)
```
Оказывается, в таблице `city` Лондон представлен 2 кодами, а вот в таблице `address` нет строк для кода, равного 313.

4. CROSS JOIN. Так как данный вариант соединения обычно выдаёт "на гора" очень много строк, я очень обрадовался, узнав, что пара таблиц, для которых имеется возможность продемонстрировать подобный вариант запроса, содержит всего по две строки:
```
SELECT * FROM staff CROSS JOIN store;
 staff_id | first_name | last_name | address_id |            email             | store_id | active | username |                 password                 |        last_update        |      picture       | store_id | manager_staff_id | address_id |     last_update
----------+------------+-----------+------------+------------------------------+----------+--------+----------+------------------------------------------+---------------------------+--------------------+----------+------------------+------------+---------------------
        1 | Mike       | Hillyer   |          3 | Mike.Hillyer@sakilastaff.com |        1 | t      | Mike     | 8cb2237d0679ca88db6464eac60da96345513964 | 2006-05-16 16:13:11.79328 | \x89504e470d0a5a0a |        1 |                1 |          1 | 2006-02-15 09:57:12
        1 | Mike       | Hillyer   |          3 | Mike.Hillyer@sakilastaff.com |        1 | t      | Mike     | 8cb2237d0679ca88db6464eac60da96345513964 | 2006-05-16 16:13:11.79328 | \x89504e470d0a5a0a |        2 |                2 |          2 | 2006-02-15 09:57:12
        2 | Jon        | Stephens  |          4 | Jon.Stephens@sakilastaff.com |        2 | t      | Jon      | 8cb2237d0679ca88db6464eac60da96345513964 | 2006-05-16 16:13:11.79328 |                    |        1 |                1 |          1 | 2006-02-15 09:57:12
        2 | Jon        | Stephens  |          4 | Jon.Stephens@sakilastaff.com |        2 | t      | Jon      | 8cb2237d0679ca88db6464eac60da96345513964 | 2006-05-16 16:13:11.79328 |                    |        2 |                2 |          2 | 2006-02-15 09:57:12
(4 rows)
```
Запрос выводит все возможные комбинации строк первой и второй таблиц.

5. FULL OUTER JOIN. Поскольку, как я уже говорил ранее, информация в БД отличается полнотой, а так же, благодаря внешним ключам, логической целостностью, то для демострации работы этого варианта соединения придётся удалить внешние ключи и ограничения NOT NULL, чтобы в некоторые таблицы можно было добавить строки с фиктивной информацией. Поработаем над таблицами `county` и `city`: удалим вышеперчисленные ограничения и добавим в них соответсвенно пару стран, у которых не будет городов и несколько городов, которые не находятся ни в одной стране:
```
INSERT INTO country(country) VALUES ('Швамбрания'), ('Атлантида');
INSERT INTO city(city) VALUES ('Черноморск'), ('Арбатов'), ('Старгород'), ('Васюки');
```
Полное соединение таблиц нам выдаст результат, в котором будут объединены 600 строк, возвращаемых запросом `INNER JOIN`, а так же 2 дополнительные строки от `LEFT JOIN` и 4 -- от `RIGHT JOIN`. Целиком приводить результат запроса я не буду, а покажу лишь некоторое количество последних строк:
```
SELECT country, city FROM country FULL OUTER JOIN city USING (country_id) ORDER BY 1,2;
 Vietnam                               | Cam Ranh
 Vietnam                               | Haiphong
 Vietnam                               | Hanoi
 Vietnam                               | Nam Dinh
 Vietnam                               | Nha Trang
 Vietnam                               | Vinh
 Virgin Islands, U.S.                  | Charlotte Amalie
 Yemen                                 | Aden
 Yemen                                 | Hodeida
 Yemen                                 | Sanaa
 Yemen                                 | Taizz
 Yugoslavia                            | Kragujevac
 Yugoslavia                            | Novi Sad
 Zambia                                | Kitwe
 Атлантида                             |
 Швамбрания                            |
                                       | Арбатов
                                       | Васюки
                                       | Старгород
                                       | Черноморск
(606 rows)
```

6. Структура таблиц, использовавшихся в запросах:
```
HomeWork 12=# \d film
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

HomeWork 12=# \d film_category
                        Table "public.film_category"
   Column    |            Type             | Collation | Nullable | Default
-------------+-----------------------------+-----------+----------+---------
 film_id     | smallint                    |           | not null |
 category_id | smallint                    |           | not null |
 last_update | timestamp without time zone |           | not null | now()
Indexes:
    "film_category_pkey" PRIMARY KEY, btree (film_id, category_id)
Foreign-key constraints:
    "film_category_category_id_fkey" FOREIGN KEY (category_id) REFERENCES category(category_id) ON UPDATE CASCADE ON DELETE RESTRICT
    "film_category_film_id_fkey" FOREIGN KEY (film_id) REFERENCES film(film_id) ON UPDATE CASCADE ON DELETE RESTRICT
Triggers:
    last_updated BEFORE UPDATE ON film_category FOR EACH ROW EXECUTE FUNCTION last_updated()

HomeWork 12=# \d category
                                             Table "public.category"
   Column    |            Type             | Collation | Nullable |                    Default
-------------+-----------------------------+-----------+----------+-----------------------------------------------
 category_id | integer                     |           | not null | nextval('category_category_id_seq'::regclass)
 name        | character varying(25)       |           | not null |
 last_update | timestamp without time zone |           | not null | now()
Indexes:
    "category_pkey" PRIMARY KEY, btree (category_id)
Referenced by:
    TABLE "film_category" CONSTRAINT "film_category_category_id_fkey" FOREIGN KEY (category_id) REFERENCES category(category_id) ON UPDATE CASCADE ON DELETE RESTRICT
Triggers:
    last_updated BEFORE UPDATE ON category FOR EACH ROW EXECUTE FUNCTION last_updated()

HomeWork 12=# \d city
                                           Table "public.city"
   Column    |            Type             | Collation | Nullable |                Default
-------------+-----------------------------+-----------+----------+---------------------------------------
 city_id     | integer                     |           | not null | nextval('city_city_id_seq'::regclass)
 city        | character varying(50)       |           | not null |
 country_id  | smallint                    |           |          |
 last_update | timestamp without time zone |           | not null | now()
Indexes:
    "city_pkey" PRIMARY KEY, btree (city_id)
    "idx_fk_country_id" btree (country_id)
Referenced by:
    TABLE "address" CONSTRAINT "fk_address_city" FOREIGN KEY (city_id) REFERENCES city(city_id)
Triggers:
    last_updated BEFORE UPDATE ON city FOR EACH ROW EXECUTE FUNCTION last_updated()

HomeWork 12=# \d address
                                             Table "public.address"
   Column    |            Type             | Collation | Nullable |                   Default
-------------+-----------------------------+-----------+----------+---------------------------------------------
 address_id  | integer                     |           | not null | nextval('address_address_id_seq'::regclass)
 address     | character varying(50)       |           | not null |
 address2    | character varying(50)       |           |          |
 district    | character varying(20)       |           | not null |
 city_id     | smallint                    |           | not null |
 postal_code | character varying(10)       |           |          |
 phone       | character varying(20)       |           | not null |
 last_update | timestamp without time zone |           | not null | now()
Indexes:
    "address_pkey" PRIMARY KEY, btree (address_id)
    "idx_fk_city_id" btree (city_id)
Foreign-key constraints:
    "fk_address_city" FOREIGN KEY (city_id) REFERENCES city(city_id)
Referenced by:
    TABLE "customer" CONSTRAINT "customer_address_id_fkey" FOREIGN KEY (address_id) REFERENCES address(address_id) ON UPDATE CASCADE ON DELETE RESTRICT
    TABLE "staff" CONSTRAINT "staff_address_id_fkey" FOREIGN KEY (address_id) REFERENCES address(address_id) ON UPDATE CASCADE ON DELETE RESTRICT
    TABLE "store" CONSTRAINT "store_address_id_fkey" FOREIGN KEY (address_id) REFERENCES address(address_id) ON UPDATE CASCADE ON DELETE RESTRICT
Triggers:
    last_updated BEFORE UPDATE ON address FOR EACH ROW EXECUTE FUNCTION last_updated()

HomeWork 12=# \d staff
                                            Table "public.staff"
   Column    |            Type             | Collation | Nullable |                 Default
-------------+-----------------------------+-----------+----------+-----------------------------------------
 staff_id    | integer                     |           | not null | nextval('staff_staff_id_seq'::regclass)
 first_name  | character varying(45)       |           | not null |
 last_name   | character varying(45)       |           | not null |
 address_id  | smallint                    |           | not null |
 email       | character varying(50)       |           |          |
 store_id    | smallint                    |           | not null |
 active      | boolean                     |           | not null | true
 username    | character varying(16)       |           | not null |
 password    | character varying(40)       |           |          |
 last_update | timestamp without time zone |           | not null | now()
 picture     | bytea                       |           |          |
Indexes:
    "staff_pkey" PRIMARY KEY, btree (staff_id)
Foreign-key constraints:
    "staff_address_id_fkey" FOREIGN KEY (address_id) REFERENCES address(address_id) ON UPDATE CASCADE ON DELETE RESTRICT
Referenced by:
    TABLE "payment" CONSTRAINT "payment_staff_id_fkey" FOREIGN KEY (staff_id) REFERENCES staff(staff_id) ON UPDATE CASCADE ON DELETE RESTRICT
    TABLE "rental" CONSTRAINT "rental_staff_id_key" FOREIGN KEY (staff_id) REFERENCES staff(staff_id)
    TABLE "store" CONSTRAINT "store_manager_staff_id_fkey" FOREIGN KEY (manager_staff_id) REFERENCES staff(staff_id) ON UPDATE CASCADE ON DELETE RESTRICT
Triggers:
    last_updated BEFORE UPDATE ON staff FOR EACH ROW EXECUTE FUNCTION last_updated()

HomeWork 12=# \d store
                                              Table "public.store"
      Column      |            Type             | Collation | Nullable |                 Default
------------------+-----------------------------+-----------+----------+-----------------------------------------
 store_id         | integer                     |           | not null | nextval('store_store_id_seq'::regclass)
 manager_staff_id | smallint                    |           | not null |
 address_id       | smallint                    |           | not null |
 last_update      | timestamp without time zone |           | not null | now()
Indexes:
    "store_pkey" PRIMARY KEY, btree (store_id)
    "idx_unq_manager_staff_id" UNIQUE, btree (manager_staff_id)
Foreign-key constraints:
    "store_address_id_fkey" FOREIGN KEY (address_id) REFERENCES address(address_id) ON UPDATE CASCADE ON DELETE RESTRICT
    "store_manager_staff_id_fkey" FOREIGN KEY (manager_staff_id) REFERENCES staff(staff_id) ON UPDATE CASCADE ON DELETE RESTRICT
Triggers:
    last_updated BEFORE UPDATE ON store FOR EACH ROW EXECUTE FUNCTION last_updated()

HomeWork 12=# \d country
                                             Table "public.country"
   Column    |            Type             | Collation | Nullable |                   Default
-------------+-----------------------------+-----------+----------+---------------------------------------------
 country_id  | integer                     |           | not null | nextval('country_country_id_seq'::regclass)
 country     | character varying(50)       |           | not null |
 last_update | timestamp without time zone |           | not null | now()
Indexes:
    "country_pkey" PRIMARY KEY, btree (country_id)
Triggers:
    last_updated BEFORE UPDATE ON country FOR EACH ROW EXECUTE FUNCTION last_updated()
```

7. Полезные метрики на основе статистических представлений:
* таблицы, которые не мешало бы проиндексировать:\
`SELECT s.relname, c.reltuples::bigint reltuples, s.seq_scan, s.idx_scan FROM pg_stat_user_tables s JOIN pg_class c ON s.relid=c.oid WHERE s.schemaname='public' ORDER BY s.seq_scan DESC;`
* таблицы, которым, возможно, требуются специфические настройки автовакуума:\
`SELECT s.relname, c.reltuples::bigint reltuples, s.n_dead_tup FROM pg_stat_user_tables s JOIN pg_class c ON s.relid=c.oid WHERE s.schemaname='public' ORDER BY s.n_dead_tup  DESC;`
* запросы, которые дольше всего бездействуют:\
`SELECT pid, usename, client_addr, state, now()-state_change time_pass, wait_event, wait_event_type, regexp_replace(query,'\s+',' ','g') query,backend_type from pg_stat_activity where state != 'active' ORDER BY time_pass DESC;`

8. Добавим запрос, комбинирующий различные виды соединений. Используя таблицу `staff` в качестве основной, узнаем полные адреса проживания сотрудников. Для начала посмотрим, что выведет нормальный запрос: 
```
SELECT s.first_name, s.last_name, a.address, c.city, st.country FROM staff s JOIN address a USING (address_id) JOIN city c USING (city_id) JOIN country st USING (country_id);
 first_name | last_name |       address        |    city    |  country
------------+-----------+----------------------+------------+-----------
 Mike       | Hillyer   | 23 Workhaven Lane    | Lethbridge | Canada
 Jon        | Stephens  | 1411 Lillydale Drive | Woodridge  | Australia
(2 rows)
```
Если же мы теперь добавим модификаторы `LEFT` и `RIGHT`, то в итоге получим список стран c редкими вкраплениями основной информации:
```
SELECT s.first_name, s.last_name, a.address, c.city, st.country FROM staff s INNER JOIN address a USING (address_id) LEFT JOIN city c USING (city_id) RIGHT JOIN country st USING (country_id);
 first_name | last_name |       address        |    city    |                country
------------+-----------+----------------------+------------+---------------------------------------
            |           |                      |            | Afghanistan
            |           |                      |            | Algeria
            |           |                      |            | American Samoa
            |           |                      |            | Angola
            |           |                      |            | Anguilla
            |           |                      |            | Argentina
            |           |                      |            | Armenia
 Jon        | Stephens  | 1411 Lillydale Drive | Woodridge  | Australia
            |           |                      |            | Austria
            |           |                      |            | Azerbaijan
            |           |                      |            | Bahrain
            |           |                      |            | Bangladesh
            |           |                      |            | Belarus
            |           |                      |            | Bolivia
            |           |                      |            | Brazil
            |           |                      |            | Brunei
            |           |                      |            | Bulgaria
            |           |                      |            | Cambodia
            |           |                      |            | Cameroon
 Mike       | Hillyer   | 23 Workhaven Lane    | Lethbridge | Canada
            |           |                      |            | Chad
            |           |                      |            | Chile
            |           |                      |            | China
            |           |                      |            | Colombia
            |           |                      |            | Congo, The Democratic Republic of the
            |           |                      |            | Czech Republic
            |           |                      |            | Dominican Republic
...
            |           |                      |            | Vietnam
            |           |                      |            | Virgin Islands, U.S.
            |           |                      |            | Yemen
            |           |                      |            | Yugoslavia
            |           |                      |            | Zambia
            |           |                      |            | Швамбрания
            |           |                      |            | Атлантида
(111 rows)
```