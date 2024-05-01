### Домашнее задание к уроку "Настройка autovacuum с учетом особеностей производительности"

1. В точности такую же ВМ, как указано в задании, мне создать не удалось, посему я воспользовался готовым компьютером со следущими характеристиками: 12 ядер, 32 ГБ ОЗУ и SSD диск.
2. Установил на него 15 версию PostgreSQL, настройки оставив без изменений.
3. Создал БД: `CREATE DATABASE "Lesson 05";` и подготовил её для запуска нагрузочных испытаний: `pgbench -i 'Lesson 05'`
4. Запустил тест: `pgbench -c 8 -P 6 -T 60 'Lesson 05'`. Вот, что он выдал:
```
starting vacuum...end.
progress: 6.0 s, 1256.0 tps, lat 6.326 ms stddev 4.648, 0 failed
progress: 12.0 s, 1242.0 tps, lat 6.433 ms stddev 4.936, 0 failed
progress: 18.0 s, 1273.5 tps, lat 6.273 ms stddev 4.663, 0 failed
progress: 24.0 s, 1291.2 tps, lat 6.189 ms stddev 4.545, 0 failed
progress: 30.0 s, 1264.7 tps, lat 6.320 ms stddev 4.662, 0 failed
progress: 36.0 s, 1268.0 tps, lat 6.302 ms stddev 4.709, 0 failed
progress: 42.0 s, 1229.5 tps, lat 6.496 ms stddev 4.880, 0 failed
progress: 48.0 s, 1268.5 tps, lat 6.297 ms stddev 4.813, 0 failed
progress: 54.0 s, 1240.5 tps, lat 6.444 ms stddev 4.771, 0 failed
progress: 60.0 s, 1259.7 tps, lat 6.341 ms stddev 4.557, 0 failed
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 8
number of threads: 1
maximum number of tries: 1
duration: 60 s
number of transactions actually processed: 75568
number of failed transactions: 0 (0.000%)
latency average = 6.341 ms
latency stddev = 4.720 ms
initial connection time = 28.210 ms
tps = 1259.919713 (without initial connection time)
```
5. Поскольку найти таинственный файл с настройками PostgreSQL, "прикреплённый к материалам занятия", нигде так и не удалось обнаружить, я воспользовался теми настройками, которые выложил в Telegram-канале нашей группы один из обучающихся на курсе:
```
max_connections = 40
shared_buffers = 1GB
effective_cache_size = 3GB
maintenance_work_mem = 512MB
checkpoint_completion_target = 0.9
max_wal_size = 16GB
min_wal_size = 4GB
wal_buffers = 16MB
default_statistics_target = 500
effective_io_concurrency = 2
work_mem = 64MB
random_page_cost = 4
```
Чтобы данные параметры применились, я перезапустил Postgres.

6. Вторая попытка: `pgbench -c 8 -P 6 -T 60 'Lesson 05'` и её результаты:
```
starting vacuum...end.
progress: 6.0 s, 1257.3 tps, lat 6.315 ms stddev 4.613, 0 failed
progress: 12.0 s, 1274.2 tps, lat 6.275 ms stddev 4.597, 0 failed
progress: 18.0 s, 1273.8 tps, lat 6.272 ms stddev 4.647, 0 failed
progress: 24.0 s, 1272.7 tps, lat 6.277 ms stddev 4.795, 0 failed
progress: 30.0 s, 1262.8 tps, lat 6.326 ms stddev 4.703, 0 failed
progress: 36.0 s, 1232.5 tps, lat 6.484 ms stddev 4.764, 0 failed
progress: 42.0 s, 1223.8 tps, lat 6.526 ms stddev 4.895, 0 failed
progress: 48.0 s, 1258.7 tps, lat 6.349 ms stddev 4.657, 0 failed
progress: 54.0 s, 1250.7 tps, lat 6.388 ms stddev 4.594, 0 failed
progress: 60.0 s, 1231.5 tps, lat 6.487 ms stddev 4.624, 0 failed
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 8
number of threads: 1
maximum number of tries: 1
duration: 60 s
number of transactions actually processed: 75236
number of failed transactions: 0 (0.000%)
latency average = 6.369 ms
latency stddev = 4.690 ms
initial connection time = 29.947 ms
tps = 1254.407374 (without initial connection time)
```
Откровенно говоря, существенных различий не наблюдается. Возможно, не те настройки.

7. Ради интереса я решил ещё больше увеличить некоторые параметры:
```
shared_buffers = 8GB
work_mem = 256MB
maintenance_work_mem = 1024MB
max_worker_processes = 16
max_parallel_maintenance_workers = 8
```
Перезапускаю PG, повторно запускаю тест и вот что получаю:
```
starting vacuum...end.
progress: 6.0 s, 1246.5 tps, lat 6.375 ms stddev 4.684, 0 failed
progress: 12.0 s, 1261.8 tps, lat 6.331 ms stddev 4.584, 0 failed
progress: 18.0 s, 1254.3 tps, lat 6.369 ms stddev 4.782, 0 failed
progress: 24.0 s, 1250.0 tps, lat 6.390 ms stddev 4.552, 0 failed
progress: 30.0 s, 1233.2 tps, lat 6.479 ms stddev 4.804, 0 failed
progress: 36.0 s, 1243.9 tps, lat 6.422 ms stddev 4.725, 0 failed
progress: 42.0 s, 1276.6 tps, lat 6.257 ms stddev 4.669, 0 failed
progress: 48.0 s, 1284.0 tps, lat 6.226 ms stddev 4.655, 0 failed
progress: 54.0 s, 1293.2 tps, lat 6.177 ms stddev 4.789, 0 failed
progress: 60.0 s, 1290.0 tps, lat 6.195 ms stddev 4.633, 0 failed
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 8
number of threads: 1
maximum number of tries: 1
duration: 60 s
number of transactions actually processed: 75809
number of failed transactions: 0 (0.000%)
latency average = 6.321 ms
latency stddev = 4.689 ms
initial connection time = 28.835 ms
tps = 1263.941491 (without initial connection time)
```
Опять-таки прирост менее одного процента: (1263 транзакций/сек против 1254 в прошлый раз). Очевидно, в итоге должно быть более существенное увеличение скорости, обусловленное правильным подбором конфигурационных параметров. 

8. Перейдём, однако, к AUTOVACUUM'у.
9. Создадим таблицу: `CREATE TABLE vacuum_test(id serial, text_field character varying(60));`
10. Для заполнения таблицы случайными значениями воспользуемся функцией `gen_random_uuid()` из расширения `pgcrypto`, предварительно установив его:
`CREATE EXTENSION pgcrypto;`
11. Заполним таблицу данными: `INSERT INTO vacuum_test(text_field) SELECT gen_random_uuid() FROM generate_series(1,1000000);`
12. Выясним размер таблицы:
```
SELECT pg_size_pretty(pg_relation_size('vacuum_test'));
 pg_size_pretty
----------------
 73 MB
(1 row)
```
13. 5 раз обновим каждую строку:
```
Lesson 05=# UPDATE vacuum_test SET text_field = text_field || 'A';
UPDATE 1000000
Lesson 05=# UPDATE vacuum_test SET text_field = text_field || 'B';
UPDATE 1000000
Lesson 05=# UPDATE vacuum_test SET text_field = text_field || 'C';
UPDATE 1000000
Lesson 05=# UPDATE vacuum_test SET text_field = text_field || 'D';
UPDATE 1000000
Lesson 05=# UPDATE vacuum_test SET text_field = text_field || 'E';
UPDATE 1000000
```
Автовакуум у нас шустрый -- он успел отработать, пока я набирал следующую команду SQL:
```
 SELECT n_dead_tup, last_autovacuum FROM pg_stat_user_tables WHERE relname='vacuum_test';
 n_dead_tup |       last_autovacuum
------------+------------------------------
          0 | 2024-05-01 11:15:51.94131+03
(1 row)
```
14. Попробуем ещё раз, но сейчас в другой сессии я создам файл с набором нужных команд, чтобы обогнать процедуру автоочистки:
```
cat << ZE_END > hw5.sql
UPDATE vacuum_test SET text_field = 'a' || text_field;
UPDATE vacuum_test SET text_field = 'b' || text_field;
UPDATE vacuum_test SET text_field = 'c' || text_field;
UPDATE vacuum_test SET text_field = 'd' || text_field;
UPDATE vacuum_test SET text_field = 'e' || text_field;
SELECT n_dead_tup, last_autovacuum FROM pg_stat_user_tables WHERE relname='vacuum_test';
SELECT pg_size_pretty(pg_relation_size('vacuum_test'));
ZE_END
``` 
15. Выполняем: `\i hw5.sql`. Кажется, получилось:
```
UPDATE 1000000
UPDATE 1000000
UPDATE 1000000
UPDATE 1000000
UPDATE 1000000
 n_dead_tup |       last_autovacuum
------------+------------------------------
    4999474 | 2024-05-01 11:15:51.94131+03
(1 row)

 pg_size_pretty
----------------
 461 MB
(1 row)
```
Теперь мы видим почти 5 млн. мёртвых строк и более чем шестикратное увеличение размера таблицы.

16. Пока суд да дело, автовакуум сделал своё дело:
```
SELECT n_dead_tup, last_autovacuum FROM pg_stat_user_tables WHERE relname='vacuum_test';
 n_dead_tup |        last_autovacuum
------------+-------------------------------
          0 | 2024-05-01 11:32:53.173632+03
(1 row)
```
однако, размер таблицы не уменьшился:
```
SELECT pg_size_pretty(pg_relation_size('vacuum_test'));
 pg_size_pretty
----------------
 461 MB
(1 row)
```
17. Ещё помучаем таблицу, предварительно отключив автоочистку и 10 раз обновим каждую из строк: 
```
ALTER TABLE vacuum_test SET (autovacuum_enabled = 0);
ALTER TABLE
UPDATE vacuum_test SET text_field = 'F' || text_field;
UPDATE 1000000
UPDATE vacuum_test SET text_field = 'G' || text_field;
UPDATE 1000000
UPDATE vacuum_test SET text_field = 'H' || text_field;
UPDATE 1000000
UPDATE vacuum_test SET text_field = 'I' || text_field;
UPDATE 1000000
UPDATE vacuum_test SET text_field = 'J' || text_field;
UPDATE 1000000
UPDATE vacuum_test SET text_field = text_field || 'k';
UPDATE 1000000
UPDATE vacuum_test SET text_field = text_field || 'l';
UPDATE 1000000
UPDATE vacuum_test SET text_field = text_field || 'm';
UPDATE 1000000
UPDATE vacuum_test SET text_field = text_field || 'n';
UPDATE 1000000
UPDATE vacuum_test SET text_field = text_field || 'o';
UPDATE 1000000
```
Смотрим на размер таблицы и количество мёртвых строк:
```
SELECT pg_size_pretty(pg_relation_size('vacuum_test'));
 pg_size_pretty
----------------
 927 MB
(1 row)
SELECT n_dead_tup, last_autovacuum FROM pg_stat_user_tables WHERE relname='vacuum_test';
 n_dead_tup |        last_autovacuum
------------+-------------------------------
    9997876 | 2024-05-01 11:32:53.173632+03
(1 row)
```
Значение `n_dead_tup` в целом соответствует общему количеству обновлённых строк - 10 млн. Размер таблицы вырос практически в два раза. Двукратный рост можно объяснить тем, что первые 5 обновлений размещали новые версии строк в неиспользованном пространстве внутри таблицы, которое освободил последний запуск автоочистки, случившийся 2024-05-01 в 11:32:53. Вторые же 5 обновлений уже не имели возможности работать в неиспользованном пространстве, поэтому добавление строк в процессе UPDATE привело к увеличению размера таблицы.

Процедура, обновляющая в цикле 10 раз таблицу:
```
DO $$
   DECLARE nStep int;
BEGIN
   nStep = 0;
   LOOP
      nStep = nStep + 1;
      UPDATE vacuum_test SET text_field = text_field || chr(nStep + 64);
      RAISE NOTICE 'Step No: %', nStep;
      EXIT WHEN nStep = 10;
   END LOOP;
   RAISE NOTICE 'Completed.';
END
$$;
```
