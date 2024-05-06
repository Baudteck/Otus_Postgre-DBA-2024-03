### Домашнее задание к уроку "Работа с журналами"

Для выполнения этого домашнего задания воспользуемся машиной, которая использовалась в домашней работе к предыдущему уроку: 12 ядер, 32 ГБ ОЗУ и SSD диск. Настройки, соответственно, остались прежними. 

1. Для настройки заданного интервала выполнения контрольной точки внесём следующую правку в `postgresql.conf`: `"checkpoint_timeout = 30s"`. Чтобы информация о контрольных точках попадала в лог-файл, проверим настройку `log_checkpoints`. Она включена по умолчанию `"#log_checkpoints = on"`, поэтому знак комментария можно не убирать.

2. Создадим и проинициализируем БД для выполнения ДЗ:
```
$ psql -c 'CREATE DATABASE "HomeWork 06";'
$ pgbench -i 'HomeWork 06'
```


3. Чтобы получить как можно более точную информацию об объёме журнальных файлов, выполним три команды в одной связке: `psql -c "SELECT pg_current_wal_insert_lsn();" && pgbench -c 10 -P 30 -T 600 'HomeWork 06' && psql -c "SELECT pg_current_wal_insert_lsn();"`

```
$ psql -c "SELECT pg_current_wal_insert_lsn();"; pgbench -c 10 -P 30 -T 600 'HomeWork 06'; psql -c "SELECT pg_current_wal_insert_lsn();"

 pg_current_wal_insert_lsn
---------------------------
 1/C480AE68
(1 row)

pgbench (15.3 (Ubuntu 15.3-1.pgdg18.04+1))
starting vacuum...end.
progress: 30.0 s, 1153.4 tps, lat 8.649 ms stddev 7.295, 0 failed
progress: 60.0 s, 1161.2 tps, lat 8.603 ms stddev 7.707, 0 failed
progress: 90.0 s, 1167.6 tps, lat 8.556 ms stddev 7.167, 0 failed
...
progress: 540.0 s, 1148.3 tps, lat 8.699 ms stddev 11.017, 0 failed
progress: 570.0 s, 1159.2 tps, lat 8.618 ms stddev 7.111, 0 failed
progress: 600.0 s, 1167.2 tps, lat 8.559 ms stddev 7.060, 0 failed
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 10
number of threads: 1
maximum number of tries: 1
duration: 600 s
number of transactions actually processed: 697431
number of failed transactions: 0 (0.000%)
latency average = 8.594 ms
latency stddev = 7.666 ms
initial connection time = 36.425 ms
tps = 1162.437186 (without initial connection time)
 pg_current_wal_insert_lsn
---------------------------
 1/EAB430E8
(1 row)
```

4. Подсчитаем общий объём созданных журнальных файлов (в байтах):
```
SELECT '1/EAB430E8'::pg_lsn - '1/C480AE68'::pg_lsn "WAL amount";
 WAL amount
------------
  640909952
(1 row)
```
Если исходить из предположения, что первая контрольная точка случилась через 30 секунд после запуска нагрузки, а последняя - совпала с её окончанием, то у нас получается 20 контрольных точек. Отсюда выходит, что на одну контрольную точку приходится в среднем 32045497,6 ~ 32045500 байт (30,6 МиБ)

5. Проверим интервалы между выполнениями checkpoint'ов: `grep checkpoint /var/log/postgresql/postgresql-15-main-2024-05-04_205757.log`. Поскольку я не догадался засечь время начала и окончания тестовой нагрузки, будем добывать эту информацию из лога.
```
2024-05-04 20:58:27.597 MSK [172061] LOG:  checkpoint starting: time
2024-05-04 20:58:27.629 MSK [172061] LOG:  checkpoint complete: wrote 3 buffers (0.0%); 0 WAL file(s) added, 0 removed, 0 recycled; write=0.027 s, sync=0.002 s, total=0.033 s; sync files=2, longest=0.001 s, average=0.001 s; distance=0 kB, estimate=0 kB
2024-05-04 20:59:57.719 MSK [172061] LOG:  checkpoint starting: time
2024-05-04 21:00:24.028 MSK [172061] LOG:  checkpoint complete: wrote 1702 buffers (0.2%); 0 WAL file(s) added, 1 removed, 0 recycled; write=26.292 s, sync=0.009 s, total=26.309 s; sync files=47, longest=0.001 s, average=0.001 s; distance=12783 kB, estimate=12783 kB
2024-05-04 21:11:27.608 MSK [172061] LOG:  checkpoint starting: time
2024-05-04 21:11:54.036 MSK [172061] LOG:  checkpoint complete: wrote 2227 buffers (0.2%); 0 WAL file(s) added, 2 removed, 0 recycled; write=26.396 s, sync=0.009 s, total=26.429 s; sync files=22, longest=0.002 s, average=0.001 s; distance=30088 kB, estimate=30088 kB
2024-05-04 21:11:57.040 MSK [172061] LOG:  checkpoint starting: time
2024-05-04 21:12:24.050 MSK [172061] LOG:  checkpoint complete: wrote 2166 buffers (0.2%); 0 WAL file(s) added, 2 removed, 0 recycled; write=26.991 s, sync=0.007 s, total=27.011 s; sync files=10, longest=0.002 s, average=0.001 s; distance=31520 kB, estimate=31520 kB
2024-05-04 21:12:27.053 MSK [172061] LOG:  checkpoint starting: time
2024-05-04 21:12:54.343 MSK [172061] LOG:  checkpoint complete: wrote 2413 buffers (0.2%); 0 WAL file(s) added, 2 removed, 0 recycled; write=26.992 s, sync=0.280 s, total=27.291 s; sync files=22, longest=0.271 s, average=0.013 s; distance=32197 kB, estimate=32197 kB
2024-05-04 21:12:57.346 MSK [172061] LOG:  checkpoint starting: time
2024-05-04 21:13:24.056 MSK [172061] LOG:  checkpoint complete: wrote 2192 buffers (0.2%); 0 WAL file(s) added, 2 removed, 0 recycled; write=26.686 s, sync=0.006 s, total=26.710 s; sync files=10, longest=0.002 s, average=0.001 s; distance=32037 kB, estimate=32181 kB
2024-05-04 21:13:27.057 MSK [172061] LOG:  checkpoint starting: time
2024-05-04 21:13:54.398 MSK [172061] LOG:  checkpoint complete: wrote 2371 buffers (0.2%); 0 WAL file(s) added, 2 removed, 0 recycled; write=26.987 s, sync=0.342 s, total=27.342 s; sync files=19, longest=0.336 s, average=0.018 s; distance=31372 kB, estimate=32100 kB
2024-05-04 21:13:57.402 MSK [172061] LOG:  checkpoint starting: time
2024-05-04 21:14:24.113 MSK [172061] LOG:  checkpoint complete: wrote 2157 buffers (0.2%); 0 WAL file(s) added, 2 removed, 0 recycled; write=26.684 s, sync=0.006 s, total=26.712 s; sync files=9, longest=0.002 s, average=0.001 s; distance=31528 kB, estimate=32043 kB
2024-05-04 21:14:27.116 MSK [172061] LOG:  checkpoint starting: time
2024-05-04 21:14:54.553 MSK [172061] LOG:  checkpoint complete: wrote 2355 buffers (0.2%); 0 WAL file(s) added, 1 removed, 0 recycled; write=26.885 s, sync=0.533 s, total=27.437 s; sync files=20, longest=0.526 s, average=0.027 s; distance=30922 kB, estimate=31931 kB
2024-05-04 21:14:57.556 MSK [172061] LOG:  checkpoint starting: time
2024-05-04 21:15:24.075 MSK [172061] LOG:  checkpoint complete: wrote 2143 buffers (0.2%); 0 WAL file(s) added, 2 removed, 0 recycled; write=26.491 s, sync=0.006 s, total=26.520 s; sync files=10, longest=0.002 s, average=0.001 s; distance=30583 kB, estimate=31796 kB
2024-05-04 21:15:27.076 MSK [172061] LOG:  checkpoint starting: time
2024-05-04 21:15:54.622 MSK [172061] LOG:  checkpoint complete: wrote 2319 buffers (0.2%); 0 WAL file(s) added, 2 removed, 0 recycled; write=26.993 s, sync=0.544 s, total=27.547 s; sync files=20, longest=0.537 s, average=0.028 s; distance=30115 kB, estimate=31628 kB
2024-05-04 21:15:57.623 MSK [172061] LOG:  checkpoint starting: time
2024-05-04 21:16:24.043 MSK [172061] LOG:  checkpoint complete: wrote 2120 buffers (0.2%); 0 WAL file(s) added, 2 removed, 0 recycled; write=26.391 s, sync=0.008 s, total=26.420 s; sync files=11, longest=0.003 s, average=0.001 s; distance=30488 kB, estimate=31514 kB
2024-05-04 21:16:27.046 MSK [172061] LOG:  checkpoint starting: time
2024-05-04 21:16:54.344 MSK [172061] LOG:  checkpoint complete: wrote 2288 buffers (0.2%); 0 WAL file(s) added, 2 removed, 0 recycled; write=26.993 s, sync=0.294 s, total=27.299 s; sync files=19, longest=0.288 s, average=0.016 s; distance=29611 kB, estimate=31324 kB
2024-05-04 21:16:57.347 MSK [172061] LOG:  checkpoint starting: time
2024-05-04 21:17:24.061 MSK [172061] LOG:  checkpoint complete: wrote 2092 buffers (0.2%); 0 WAL file(s) added, 2 removed, 0 recycled; write=26.687 s, sync=0.015 s, total=26.714 s; sync files=9, longest=0.009 s, average=0.002 s; distance=30304 kB, estimate=31222 kB
2024-05-04 21:17:27.064 MSK [172061] LOG:  checkpoint starting: time
2024-05-04 21:17:54.084 MSK [172061] LOG:  checkpoint complete: wrote 2277 buffers (0.2%); 0 WAL file(s) added, 1 removed, 0 recycled; write=26.984 s, sync=0.009 s, total=27.020 s; sync files=20, longest=0.003 s, average=0.001 s; distance=30045 kB, estimate=31104 kB
2024-05-04 21:17:57.087 MSK [172061] LOG:  checkpoint starting: time
2024-05-04 21:18:24.566 MSK [172061] LOG:  checkpoint complete: wrote 2064 buffers (0.2%); 0 WAL file(s) added, 2 removed, 0 recycled; write=26.983 s, sync=0.484 s, total=27.480 s; sync files=12, longest=0.480 s, average=0.041 s; distance=30174 kB, estimate=31011 kB
2024-05-04 21:18:27.567 MSK [172061] LOG:  checkpoint starting: time
2024-05-04 21:18:55.124 MSK [172061] LOG:  checkpoint complete: wrote 2714 buffers (0.3%); 0 WAL file(s) added, 2 removed, 0 recycled; write=26.495 s, sync=1.042 s, total=27.557 s; sync files=19, longest=1.035 s, average=0.055 s; distance=30551 kB, estimate=30965 kB
2024-05-04 21:18:57.126 MSK [172061] LOG:  checkpoint starting: time
2024-05-04 21:19:24.290 MSK [172061] LOG:  checkpoint complete: wrote 2048 buffers (0.2%); 0 WAL file(s) added, 2 removed, 0 recycled; write=26.886 s, sync=0.256 s, total=27.165 s; sync files=10, longest=0.253 s, average=0.026 s; distance=29881 kB, estimate=30857 kB
2024-05-04 21:19:27.291 MSK [172061] LOG:  checkpoint starting: time
2024-05-04 21:19:54.829 MSK [172061] LOG:  checkpoint complete: wrote 2266 buffers (0.2%); 0 WAL file(s) added, 2 removed, 0 recycled; write=26.782 s, sync=0.745 s, total=27.539 s; sync files=16, longest=0.741 s, average=0.047 s; distance=30436 kB, estimate=30815 kB
2024-05-04 21:19:57.832 MSK [172061] LOG:  checkpoint starting: time
2024-05-04 21:20:24.035 MSK [172061] LOG:  checkpoint complete: wrote 2037 buffers (0.2%); 0 WAL file(s) added, 2 removed, 0 recycled; write=26.185 s, sync=0.006 s, total=26.204 s; sync files=14, longest=0.002 s, average=0.001 s; distance=30244 kB, estimate=30758 kB
2024-05-04 21:20:27.038 MSK [172061] LOG:  checkpoint starting: time
2024-05-04 21:20:54.586 MSK [172061] LOG:  checkpoint complete: wrote 2706 buffers (0.3%); 0 WAL file(s) added, 2 removed, 0 recycled; write=26.991 s, sync=0.545 s, total=27.548 s; sync files=20, longest=0.540 s, average=0.028 s; distance=29831 kB, estimate=30665 kB
2024-05-04 21:20:57.586 MSK [172061] LOG:  checkpoint starting: time
2024-05-04 21:21:24.093 MSK [172061] LOG:  checkpoint complete: wrote 2032 buffers (0.2%); 0 WAL file(s) added, 1 removed, 0 recycled; write=26.496 s, sync=0.003 s, total=26.507 s; sync files=9, longest=0.002 s, average=0.001 s; distance=30449 kB, estimate=30643 kB
2024-05-04 21:22:27.130 MSK [172061] LOG:  checkpoint starting: time
2024-05-04 21:22:54.025 MSK [172061] LOG:  checkpoint complete: wrote 476 buffers (0.0%); 0 WAL file(s) added, 1 removed, 0 recycled; write=26.878 s, sync=0.007 s, total=26.895 s; sync files=18, longest=0.003 s, average=0.001 s; distance=13570 kB, estimate=28936 kB
```
По этой выборке можно понять, что нагрузка была запущена примерно в 22:11. Строчки `"checkpoint starting: time"` сообщают нам о том, что контрольные точки выполнялись исключительно по причине истечения очередного 30-секундного интервала. Поскольку от предыдущего домашнего задания осталось достаточно большое значение параметра `max_wal_size`, а именно 10 GB, то рассчитывать на то, что контрольная точка будет выполнена по превышению этого параметра, не стоило. Так же в логе видно, что точки начинали выполняться в точности в одно и то же время каждой минуты: в 27 и 57 секунд. Разницей в доли секунды, полагаю, можно пренебречь. 

6. Перейдём к сравнению количества транзакций в секунду в зависимости от значения параметра `"synchronous_commit"`. Выясним текущее значение:
```
$ psql -c 'SHOW synchronous_commit;'
 synchronous_commit
--------------------
 on
(1 row)
``` 
Проинициализируем БД: `"pgbench -i 'HomeWork 06'"` и запустим тест:
```
$ pgbench -P 1 -T 20 'HomeWork 06'
pgbench (15.3 (Ubuntu 15.3-1.pgdg18.04+1))
starting vacuum...end.
progress: 1.0 s, 667.0 tps, lat 1.492 ms stddev 0.416, 0 failed
progress: 2.0 s, 694.0 tps, lat 1.437 ms stddev 0.207, 0 failed
progress: 3.0 s, 687.0 tps, lat 1.458 ms stddev 0.477, 0 failed
progress: 4.0 s, 707.1 tps, lat 1.413 ms stddev 0.338, 0 failed
progress: 5.0 s, 713.0 tps, lat 1.401 ms stddev 0.512, 0 failed
progress: 6.0 s, 735.0 tps, lat 1.360 ms stddev 0.364, 0 failed
progress: 7.0 s, 748.0 tps, lat 1.337 ms stddev 0.175, 0 failed
progress: 8.0 s, 745.0 tps, lat 1.341 ms stddev 0.182, 0 failed
progress: 9.0 s, 725.0 tps, lat 1.377 ms stddev 0.237, 0 failed
progress: 10.0 s, 715.0 tps, lat 1.399 ms stddev 0.354, 0 failed
progress: 11.0 s, 731.0 tps, lat 1.367 ms stddev 0.250, 0 failed
progress: 12.0 s, 696.0 tps, lat 1.436 ms stddev 0.519, 0 failed
progress: 13.0 s, 701.0 tps, lat 1.425 ms stddev 0.331, 0 failed
progress: 14.0 s, 697.0 tps, lat 1.435 ms stddev 0.518, 0 failed
progress: 15.0 s, 691.0 tps, lat 1.447 ms stddev 0.427, 0 failed
progress: 16.0 s, 703.0 tps, lat 1.420 ms stddev 0.553, 0 failed
progress: 17.0 s, 639.0 tps, lat 1.564 ms stddev 0.521, 0 failed
progress: 18.0 s, 716.0 tps, lat 1.396 ms stddev 0.275, 0 failed
progress: 19.0 s, 683.0 tps, lat 1.463 ms stddev 0.478, 0 failed
progress: 20.0 s, 688.0 tps, lat 1.453 ms stddev 0.319, 0 failed
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 1
number of threads: 1
maximum number of tries: 1
duration: 20 s
number of transactions actually processed: 14082
number of failed transactions: 0 (0.000%)
latency average = 1.419 ms
latency stddev = 0.393 ms
initial connection time = 4.143 ms
tps = 704.222359 (without initial connection time)
```
В среднем получаем 704 транзакции/сек. Поменяем параметр:
```
~$ psql -c 'ALTER SYSTEM SET synchronous_commit TO 0;'
ALTER SYSTEM

$ psql -c 'SELECT pg_reload_conf();'
 pg_reload_conf
----------------
 t
(1 row)

$ psql -c 'SHOW synchronous_commit;'
 synchronous_commit
--------------------
 off
(1 row)

```
Заново проинициализируем БД: `"pgbench -i 'HomeWork 06'"` и повторно запустим тест:
```
$ pgbench -P 1 -T 20 'HomeWork 06'
pgbench (15.3 (Ubuntu 15.3-1.pgdg18.04+1))
starting vacuum...end.
progress: 1.0 s, 1017.9 tps, lat 0.978 ms stddev 0.355, 0 failed
progress: 2.0 s, 1065.0 tps, lat 0.938 ms stddev 0.120, 0 failed
progress: 3.0 s, 1029.0 tps, lat 0.971 ms stddev 0.434, 0 failed
progress: 4.0 s, 1033.0 tps, lat 0.967 ms stddev 0.212, 0 failed
progress: 5.0 s, 1064.0 tps, lat 0.940 ms stddev 0.096, 0 failed
progress: 6.0 s, 1066.1 tps, lat 0.938 ms stddev 0.121, 0 failed
progress: 7.0 s, 1078.9 tps, lat 0.926 ms stddev 0.153, 0 failed
progress: 8.0 s, 1025.1 tps, lat 0.975 ms stddev 0.384, 0 failed
progress: 9.0 s, 1035.0 tps, lat 0.966 ms stddev 0.411, 0 failed
progress: 10.0 s, 1054.0 tps, lat 0.948 ms stddev 0.362, 0 failed
progress: 11.0 s, 1064.9 tps, lat 0.939 ms stddev 0.092, 0 failed
progress: 12.0 s, 1025.1 tps, lat 0.975 ms stddev 0.218, 0 failed
progress: 13.0 s, 1049.0 tps, lat 0.953 ms stddev 0.276, 0 failed
progress: 14.0 s, 1076.0 tps, lat 0.929 ms stddev 0.122, 0 failed
progress: 15.0 s, 1069.0 tps, lat 0.935 ms stddev 0.253, 0 failed
progress: 16.0 s, 1060.1 tps, lat 0.943 ms stddev 0.317, 0 failed
progress: 17.0 s, 1057.9 tps, lat 0.939 ms stddev 0.182, 0 failed
progress: 18.0 s, 1071.0 tps, lat 0.938 ms stddev 0.217, 0 failed
progress: 19.0 s, 1036.0 tps, lat 0.965 ms stddev 0.318, 0 failed
progress: 20.0 s, 1047.0 tps, lat 0.955 ms stddev 0.241, 0 failed
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 1
number of threads: 1
maximum number of tries: 1
duration: 20 s
number of transactions actually processed: 21025
number of failed transactions: 0 (0.000%)
latency average = 0.951 ms
latency stddev = 0.266 ms
initial connection time = 4.146 ms
tps = 1051.413705 (without initial connection time)
```
На сей раз мы видим 1051 tps - прирост составил почти 50%! В документации сказано, что при включённом `('on')` параметре `"synchronous_commit"` присутствует ожидание успешного сброса буферов WAL на диск. В случае, когда `"synchronous_commit"` имеет значение `'off'` - ожидания нет. PosgtreSQL перепоручает всё остальное операционной системе и не ожидает успешного сохранения данных журнала предзаписи, что позволяет увеличить скорость работы. 

7. Работа с контрольными суммами. Чтобы включить контрольные суммы страниц с данными, необходимо передать соответствующий параметр программе `initdb`. Её можно вызвать как явным образом, так и косвенно - через вызов `pg_createcluster`. Воспользуемся вторым вариантом, при котором параметры, относящиеся к `initdb` отделяются от основных двумя дефисами: `"pg_createcluster 15 chksum -- --data-checksums"`. Выполняем эту команду и видим, что кластер успешно создан. Поскольку дальнейшие действия с ним будут достаточно несложными, то никакие настройки можно не менять. Запускаем кластер: `sudo systemctl start postgresql@15-chksum`, после чего подключаемя к нему, создаём таблицу,  вставляем в нёе несколько строк и выясняем, в каком файле она находится:
```
CREATE DATABASE "HomeWork 06";
\c 'HomeWork 06'
CREATE TABLE checksum(id serial, strval text);
INSERT INTO checksum(strval) VALUES ('one'), ('two'), ('three');
SELECT * FROM pg_relation_filepath('checksum');
 pg_relation_filepath
----------------------
 base/16384/16390
(1 row)
```
Останавливаю кластер и с помощью шестнадцатиричного редактора правлю несколько байт в заголовке таблицы, после чего вновь запускаю кластер. Попытка выполнить запрос `SELECT * FROM checksum;` приводит к ошибке:
```
WARNING:  page verification failed, calculated checksum 13140 but expected 59742
ERROR:  invalid page in block 0 of relation base/16384/16390
```
Один из вариантов продолжения работы - сказать Postgres'у о том, что на это можно не обращать внимание. Останавливаю кластер и добавляю в файл конфигурации строчку: `"ignore_checksum_failure=1"`, после чего вновь его запускаю и пробую выполнить тот же запрос. Postgres ругается, но данные всё же выдаёт:
```
SELECT * FROM checksum;
WARNING:  page verification failed, calculated checksum 13140 but expected 59742
 id | strval
----+--------
  1 | one
  2 | two
  3 | three
(3 rows)
```
Кроме этого, в представление `pg_stat_database` попадает количество ошибок обнаруженных ошибок контрольных сумм и время последнего такого события:
```
checksum_failures        | 2
checksum_last_failure    | 2024-05-05 20:47:54.892346+03
```
Ошибок две, так как мы дважды пытались сделать выборку из таблицы.