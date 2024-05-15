### Домашнее задание к уроку "Нагрузочное тестирование и тюнинг PostgreSQL"

Подробности создания виртуальной машины и установки 15 версии PostgreSQL опустим, ввиду их неважности для данного урока. Единственное, что стоит уточнить - параметры виртуалки:
* ОЗУ - 8 GB;
* количество ядер процессора - 8;
* тип диска - HDD.

1. Для работы программы тестирования я создал отдельную БД - "HomeWork 08". Также, ради интереса, было включено логирование создания временных файлов: `"log_temp_files = 0"`

1. Посмотрим на показатели, выдаваемые программой `pgbench` при настройках по умолчанию:
```
$ pgbench -c 80 -j 8 -P 10 -T 60 "HomeWork 08"
pgbench (15.1 (Ubuntu 15.1-1.pgdg18.04+1))
starting vacuum...end.
progress: 10.0 s, 1706.9 tps, lat 46.246 ms stddev 58.215, 0 failed
progress: 20.0 s, 1453.2 tps, lat 55.116 ms stddev 71.281, 0 failed
progress: 30.0 s, 1663.0 tps, lat 48.082 ms stddev 57.076, 0 failed
progress: 40.0 s, 1720.9 tps, lat 46.372 ms stddev 58.547, 0 failed
progress: 50.0 s, 1681.1 tps, lat 47.675 ms stddev 57.328, 0 failed
progress: 60.0 s, 1625.3 tps, lat 49.108 ms stddev 60.039, 0 failed
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 80
number of threads: 8
maximum number of tries: 1
duration: 60 s
number of transactions actually processed: 98581
number of failed transactions: 0 (0.000%)
latency average = 48.651 ms
latency stddev = 60.474 ms
initial connection time = 75.729 ms
tps = 1643.423332 (without initial connection time)
```
2. Воспользуемся конфигуратором с сайта `https://pgtune.leopard.in.ua`, задав следующие параметры сервера:
* версия PG - 15;
* тип OS - Linux;
* тип БД - OLTP;
* объём памяти - 8 ГБ;
* количество ядер ЦПУ - 8 шт;
* максимальное число соединений - 100 шт;
* тип диска - HDD.\
В ответ получим следующие параметры настройки:
```
max_connections = 100
shared_buffers = 2GB
effective_cache_size = 6GB
maintenance_work_mem = 512MB
checkpoint_completion_target = 0.9
wal_buffers = 16MB
default_statistics_target = 100
random_page_cost = 4
effective_io_concurrency = 2
work_mem = 5242kB
huge_pages = off
min_wal_size = 2GB
max_wal_size = 8GB
max_worker_processes = 8
max_parallel_workers_per_gather = 4
max_parallel_workers = 8
max_parallel_maintenance_workers = 4
```
Поместим это всё в файл postgresql.auto.conf и выполним перезапуск СУБД, так как без рестарта некоторые параметры, например `"shared_buffers"`, не применятся. Повторяем нагрузочное тестирование: 
```
$ pgbench -c 80 -j 8 -P 10 -T 60 "HomeWork 08"
pgbench (15.1 (Ubuntu 15.1-1.pgdg18.04+1))
starting vacuum...end.
progress: 10.0 s, 1682.4 tps, lat 46.979 ms stddev 55.412, 0 failed
progress: 20.0 s, 1456.5 tps, lat 54.856 ms stddev 70.483, 0 failed
progress: 30.0 s, 1722.6 tps, lat 46.356 ms stddev 55.828, 0 failed
progress: 40.0 s, 1442.7 tps, lat 55.426 ms stddev 80.585, 0 failed
progress: 50.0 s, 1638.9 tps, lat 48.904 ms stddev 59.589, 0 failed
progress: 60.0 s, 1643.5 tps, lat 48.723 ms stddev 58.937, 0 failed
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 80
number of threads: 8
maximum number of tries: 1
duration: 60 s
number of transactions actually processed: 95946
number of failed transactions: 0 (0.000%)
latency average = 49.992 ms
latency stddev = 63.615 ms
initial connection time = 74.221 ms
tps = 1599.296580 (without initial connection time)
```
Откровенно говоря, результаты меня удивили. Впрочем, если мы заглянем в журнальный файл, то не увидим там никаких сообщений об использовании временных файлов во время тестирования. Значит, оперативной памяти хватало изначально, даже до "оптимизации", и лишних обращений к диску не потребовалось. Возможно, надо было сразу задавать гораздо большее число клиентов. 

3. Попробую протестировать базу, используя 1000 клиентов. Удаляю postgresql.auto.conf и перезапускаю СУБД и даю нагрузку. Тут же натыкаюсь на ошибку, так как по умолчанию максимальное количество одновременных соединений в 10 раз меньше. Увеличим его до 1000. Да, на сей раз число транзакций в секунду уменьшилось более, чем в 4 раза:
```
$ pgbench -c 1000 -j 8 -P 10 -T 60 "HomeWork 08"
pgbench (15.1 (Ubuntu 15.1-1.pgdg18.04+1))
starting vacuum...end.
progress: 10.0 s, 497.8 tps, lat 1337.361 ms stddev 1355.646, 0 failed
progress: 20.0 s, 364.3 tps, lat 2523.655 ms stddev 2790.241, 0 failed
progress: 30.0 s, 368.4 tps, lat 2562.118 ms stddev 3179.913, 0 failed
progress: 40.0 s, 220.6 tps, lat 3582.669 ms stddev 3893.475, 0 failed
progress: 50.0 s, 363.5 tps, lat 3421.805 ms stddev 4482.308, 0 failed
progress: 60.0 s, 365.3 tps, lat 2696.567 ms stddev 3651.652, 0 failed
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 1000
number of threads: 8
maximum number of tries: 1
duration: 60 s
number of transactions actually processed: 22799
number of failed transactions: 0 (0.000%)
latency average = 2641.970 ms
latency stddev = 3411.609 ms
initial connection time = 832.168 ms
tps = 373.064644 (without initial connection time)
```
Но и сейчас система обошлась без создания временных файлов на диске. Вновь обратимся к конфигуратору, увеличив максимальное число соединений до 1000:
```
# DB Version: 15
# OS Type: linux
# DB Type: oltp
# Total Memory (RAM): 8 GB
# CPUs num: 8
# Connections num: 1000
# Data Storage: hdd

max_connections = 1000
shared_buffers = 2GB
effective_cache_size = 6GB
maintenance_work_mem = 512MB
checkpoint_completion_target = 0.9
wal_buffers = 16MB
default_statistics_target = 100
random_page_cost = 4
effective_io_concurrency = 2
work_mem = 524kB
huge_pages = off
min_wal_size = 2GB
max_wal_size = 8GB
max_worker_processes = 8
max_parallel_workers_per_gather = 4
max_parallel_workers = 8
max_parallel_maintenance_workers = 4
```
В ответ он ожидаемо предложил нам в 10 раз уменьшить `"work_mem"`, что логично - иначе тысяче соединений не хватит места в памяти. Однако, запустив тест, я получаю ошибку о нехватке памяти: 
```
$ pgbench -c 1000 -j 8 -P 10 -T 60 "HomeWork 08"
pgbench (15.1 (Ubuntu 15.1-1.pgdg18.04+1))
starting vacuum...end.
pgbench: error: connection to server on socket "/var/run/postgresql/.s.PGSQL.5432" failed: FATAL:  Peer authentication failed for user "postgres"
pgbench: error: could not create connection for client 343
pgbench: error: connection to server on socket "/var/run/postgresql/.s.PGSQL.5432" failed: could not fork new process for connection: Cannot allocate memory
pgbench: error: could not create connection for client 219
```
Выходит, что автоматические настройки не всегда бывают верны. Уменьшим `shared_buffers` до 256 MБ и снова попытаем удачу:
```
$ pgbench -c 1000 -j 8 -P 10 -T 60 "HomeWork 08"
pgbench (15.1 (Ubuntu 15.1-1.pgdg18.04+1))
starting vacuum...end.
progress: 10.0 s, 518.4 tps, lat 1364.738 ms stddev 1443.236, 0 failed
progress: 20.0 s, 461.8 tps, lat 2071.026 ms stddev 2332.643, 0 failed
progress: 30.0 s, 420.1 tps, lat 2325.744 ms stddev 2638.035, 0 failed
progress: 40.0 s, 306.5 tps, lat 2691.046 ms stddev 3003.383, 0 failed
progress: 50.0 s, 148.3 tps, lat 5164.650 ms stddev 4645.327, 0 failed
progress: 60.0 s, 165.0 tps, lat 6343.451 ms stddev 6682.022, 0 failed
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 1000
number of threads: 8
maximum number of tries: 1
duration: 60 s
number of transactions actually processed: 21201
number of failed transactions: 0 (0.000%)
latency average = 2868.899 ms
latency stddev = 3873.932 ms
initial connection time = 712.887 ms
tps = 342.738358 (without initial connection time)
```
Начали бодро, а закончили совсем плохо: даже общее число транзакций во второй попытке оказалось меньше. У меня только одно предположение - из-за маленького размера `shared_buffers` пришлось гораздо чаще сбрасывать на диск изменённые страницы, что не позволило обрабатывать больше транзакций.
 
 4. Поскольку в тексте домашего задания есть интересная формулировка "не обращая внимание на возможные проблемы с надежностью в случае аварийной перезагрузки виртуальной машины", то попробуем отключить параметр `"synchronous_commit"` и посмотрим, что из этого выйдет:
```
$ pgbench -c 1000 -j 8 -P 10 -T 60 "HomeWork 08"
pgbench (15.1 (Ubuntu 15.1-1.pgdg18.04+1))
starting vacuum...end.
progress: 10.0 s, 583.6 tps, lat 1205.906 ms stddev 1308.551, 0 failed
progress: 20.0 s, 455.7 tps, lat 2037.458 ms stddev 2340.747, 0 failed
progress: 30.0 s, 418.4 tps, lat 2279.927 ms stddev 2968.356, 0 failed
progress: 40.0 s, 385.2 tps, lat 2417.235 ms stddev 3358.449, 0 failed
progress: 50.0 s, 337.3 tps, lat 2933.720 ms stddev 3862.537, 0 failed
progress: 60.0 s, 360.9 tps, lat 2874.360 ms stddev 4096.086, 0 failed
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 1000
number of threads: 8
maximum number of tries: 1
duration: 60 s
number of transactions actually processed: 26411
number of failed transactions: 0 (0.000%)
latency average = 2286.890 ms
latency stddev = 3156.257 ms
initial connection time = 713.154 ms
tps = 431.658947 (without initial connection time)
``` 
Ну хоть на сей раз получили какой-то прирост производительности!

5. Попробуем тестирование утилитой `sysbench-tpcc`. Её установка не вызывает особой сложности. Удаляю и заново создаю базу, в которой будут находиться нагрузочные таблицы. Удаляю файл `postgresql.auto.conf` и перезапускаю сервер БД, чтобы применились параметры из `postgresql.conf`. Проинициализируем БД: `./tpcc.lua --scale=5 --tables=2 --use_fk=0 --db-driver=pgsql --pgsql-host=localhost --pgsql-db="HomeWork 08" --pgsql-schema=public --pgsql-user=postgres --pgsql-port=5432 prepare`. Чтобы этот процесс завершился быстрее, я уменьшил параметр масштабирования со 100 до 5 и отключил использование внешних ключей, но зато увеличил до 2 количество комплектов используемых таблиц.\
Запускаем нагрузку:
```
./tpcc.lua --scale=5 --tables=2 --use_fk=0 --db-driver=pgsql --pgsql-host=localhost --pgsql-db="HomeWork 08" --pgsql-schema=public --pgsql-user=postgres --pgsql-port=5432 run
sysbench 1.0.11 (using system LuaJIT 2.1.0-beta3)

Running the test with following options:
Number of threads: 1
Initializing random number generator from current time


Initializing worker threads...

DB SCHEMA public
Threads started!

SQL statistics:
    queries performed:
        read:                            817
        write:                           829
        other:                           144
        total:                           1790
    transactions:                        71     (7.06 per sec.)
    queries:                             1790   (178.08 per sec.)
    ignored errors:                      1      (0.10 per sec.)
    reconnects:                          0      (0.00 per sec.)

General statistics:
    total time:                          10.0493s
    total number of events:              71

Latency (ms):
         min:                                  2.01
         avg:                                141.52
         max:                                576.62
         95th percentile:                    404.61
         sum:                              10048.15

Threads fairness:
    events (avg/stddev):           71.0000/0.00
    execution time (avg/stddev):   10.0481/0.00
``` 

Теперь применим к базе настройки из п.2 (вариант со 100 соединениями) и заново запустим тестирование:
```
./tpcc.lua --scale=5 --tables=2 --use_fk=0 --db-driver=pgsql --pgsql-host=localhost --pgsql-db="HomeWork 08" --pgsql-schema=public --pgsql-user=postgres --pgsql-port=5432 run
sysbench 1.0.11 (using system LuaJIT 2.1.0-beta3)

Running the test with following options:
Number of threads: 1
Initializing random number generator from current time


Initializing worker threads...

DB SCHEMA public
Threads started!

SQL statistics:
    queries performed:
        read:                            993
        write:                           1022
        other:                           166
        total:                           2181
    transactions:                        82     (8.18 per sec.)
    queries:                             2181   (217.51 per sec.)
    ignored errors:                      0      (0.00 per sec.)
    reconnects:                          0      (0.00 per sec.)

General statistics:
    total time:                          10.0252s
    total number of events:              82

Latency (ms):
         min:                                  4.65
         avg:                                122.24
         max:                                629.08
         95th percentile:                    404.61
         sum:                              10023.88

Threads fairness:
    events (avg/stddev):           82.0000/0.00
    execution time (avg/stddev):   10.0239/0.00
```
Здесь мы видим прирост около 15-20%, если учитывать количество выполненных запросов и среднее количество транзакций в секунду.