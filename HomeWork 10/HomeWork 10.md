### Домашнее задание к уроку "Репликация"

Ввиду отсутствия возможности использования нескольких виртуальных машин, это ДЗ было выполнено на одном сервере. Имитация применения виртуальных машин обеспечивалась путём создания нескольких кластеров и использованием различающихся имён баз данных на них.

1. Создаём первый кластер: `pg_createcluster 15 host_01`. Поскольку все ранее созданные на этом сервере базы были предварительно удалены, то доступ к вновь созданному кластеру осуществляется по стандартному порту 5432. Подключаемся, создаём базу `"HomeWork 10 Host 01"`, а в ней - две таблицы `test1` и `test2` схожей структуры:
```
CREATE TABLE test1(id int, str text);
CREATE TABLE test2(LIKE test1);
```
По условию ДЗ первая таблица у нас предназначена для записи, поэтому вставим в неё десяток строк: `"INSERT INTO test1(id, str) SELECT val, concat('String ', val, ' on host 01') FROM generate_series (1,10) val;"`. Создадим публикацию: `
"CREATE PUBLICATION publ_test1 FOR TABLE test1;"`. Система тут же сообщает нам о том, что не все параметры СУБД соответствуют выполняемой задаче:
```
WARNING:  wal_level is insufficient to publish logical changes
HINT:  Set wal_level to "logical" before creating subscriptions.
```
Меняем параметр `wal_level` на требуемый и перезапускаем кластер. Подписаться на публикацию таблицы `test2` со второй машины мы не можем, ввиду отсутствия таковой. Приступим же к её созданию.

2. Выполняем команду`"pg_createcluster 15 host_02"`. Сразу же меняем значение `wal_level` в конфиге и запускам кластер. Поскольку порт 5432 уже занят, на вторую "виртуалку" мы будем ходить по порту 5433. Подключаемся, создаём базу, таблицы и наполняем одну из них:
```
CREATE DATABASE "HomeWork 10 Host 02";
CREATE TABLE test1(id int, str text);
CREATE TABLE test2(LIKE test1);
INSERT INTO test2(id, str) SELECT val, concat('String ', val, ' on host 02') FROM generate_series (1,10) val;
```
Создаём публикацию: `CREATE PUBLICATION publ_test2 FOR TABLE test2;`. Пытаясь подписаться на публикацию таблицы `test1` на первой виртуалке, осознаём, что для учётки `postgres` не задан пароль. В связи с этим пойдём на беспрецедентное нарушение правил безопасности: в `pg_hba.conf`прописываем пользователю `postgres` разрешение подключаться без пароля в пределах локальной сети, добавив вот такую строчку: `"host    all             postgres         127.0.0.1/32           trust"`. Делаем это для обоих кластеров, после чего оба же и перезапускаем для применения изменений.

3. Вновь подключаемся к базе на второй машине и создаём подписку на `test1`:
```
HomeWork 10 Host 02=# CREATE SUBSCRIPTION subs_test1 CONNECTION 'host=127.0.0.1 port=5432 user=postgres dbname=''HomeWork 10 Host 01''' PUBLICATION publ_test1;
NOTICE:  created replication slot "subs_test1" on publisher
CREATE SUBSCRIPTION
HomeWork 10 Host 02=# SELECT * FROM test1;
 id |         str
----+----------------------
  1 | String 1 on host 01
  2 | String 2 on host 01
  3 | String 3 on host 01
  4 | String 4 on host 01
  5 | String 5 on host 01
  6 | String 6 on host 01
  7 | String 7 on host 01
  8 | String 8 on host 01
  9 | String 9 on host 01
 10 | String 10 on host 01
(10 rows)

```
Видно, что в базу на второй "виртуалке" прилетели данные из таблицы на первой "виртуалке". 

4. Теперь подключимся к первому кластеру и подпишемся на таблицу из второго, но для начала проверим содержимое таблицы test2:
```
HomeWork 10 Host 01=# SELECT * FROM test2;
 id | str
----+-----
(0 rows)
HomeWork 10 Host 01=# CREATE SUBSCRIPTION subs_test2 CONNECTION 'host=127.0.0.1 port=5433 user=postgres dbname=''HomeWork 10 Host 02''' PUBLICATION publ_test2;
NOTICE:  created replication slot "subs_test2" on publisher
CREATE SUBSCRIPTION
HomeWork 10 Host 01=# SELECT * FROM test2;
 id |         str
----+----------------------
  1 | String 1 on host 02
  2 | String 2 on host 02
  3 | String 3 on host 02
  4 | String 4 on host 02
  5 | String 5 on host 02
  6 | String 6 on host 02
  7 | String 7 on host 02
  8 | String 8 on host 02
  9 | String 9 on host 02
 10 | String 10 on host 02
(10 rows)
```
Подписка работает, в таблице появились данные. Добавим строчку в таблицу `test1` и потом проверим, видна ли эта срока во втором кластере: `INSERT INTO test1 (id, str) VALUES (11, 'New string on host 01');`

5. Проверяем на втором кластере:
```
$ psql -p 5433 -d 'HomeWork 10 Host 02' -c 'SELECT * FROM test1 ORDER BY id;'
 id |          str
----+-----------------------
  1 | String 1 on host 01
  2 | String 2 on host 01
  3 | String 3 on host 01
  4 | String 4 on host 01
  5 | String 5 on host 01
  6 | String 6 on host 01
  7 | String 7 on host 01
  8 | String 8 on host 01
  9 | String 9 on host 01
 10 | String 10 on host 01
 11 | New string on host 01
(11 rows)
```
Всё в порядке.

6. Создаём ещё один кластер, на сей раз это будет реплика для чтения и бэкапов: `"pg_createcluster 15 host_03"`. Сразу меняю `wal_level` и добавляю возможность пользователю `postgres` подключаться по локальной сети без пароля. Создаю базу, таблицы и подписки:
```
postgres=# CREATE DATABASE "HomeWork 10 Host 03";
CREATE DATABASE
postgres=# \c "HomeWork 10 Host 03"
You are now connected to database "HomeWork 10 Host 03" as user "postgres".
HomeWork 10 Host 03=# CREATE TABLE test1(id int, str text);
CREATE TABLE
HomeWork 10 Host 03=# CREATE SUBSCRIPTION subs_test1 CONNECTION 'host=127.0.0.1 port=5432 user=postgres dbname=''HomeWork 10 Host 01''' PUBLICATION publ_test1;
ERROR:  could not create replication slot "subs_test1": ERROR:  replication slot "subs_test1" already exists
```
Ну раз `subs_test1` уже существует, создадим подписку с другим именем:
```
HomeWork 10 Host 03=# CREATE SUBSCRIPTION subs_test1_from03 CONNECTION 'host=127.0.0.1 port=5432 user=postgres dbname=''HomeWork 10 Host 01''' PUBLICATION publ_test1;
NOTICE:  created replication slot "subs_test1_from03" on publisher
CREATE SUBSCRIPTION
HomeWork 10 Host 03=# SELECT * FROM test1;
 id |          str
----+-----------------------
  1 | String 1 on host 01
  2 | String 2 on host 01
  3 | String 3 on host 01
  4 | String 4 on host 01
  5 | String 5 on host 01
  6 | String 6 on host 01
  7 | String 7 on host 01
  8 | String 8 on host 01
  9 | String 9 on host 01
 10 | String 10 on host 01
 11 | New string on host 01
(11 rows)
HomeWork 10 Host 03=# CREATE TABLE test2(id int, str text);
CREATE TABLE
HomeWork 10 Host 03=# CREATE SUBSCRIPTION subs_test2_from03 CONNECTION 'host=127.0.0.1 port=5433 user=postgres dbname=''HomeWork 10 Host 02''' PUBLICATION publ_test2;
NOTICE:  created replication slot "subs_test2_from03" on publisher
CREATE SUBSCRIPTION
HomeWork 10 Host 03=# SELECT * FROM test2;
 id |         str
----+----------------------
  1 | String 1 on host 02
  2 | String 2 on host 02
  3 | String 3 on host 02
  4 | String 4 on host 02
  5 | String 5 on host 02
  6 | String 6 on host 02
  7 | String 7 on host 02
  8 | String 8 on host 02
  9 | String 9 on host 02
 10 | String 10 on host 02
(10 rows)
```
Все данные на месте. Выйдем в оболочку операционной системы и создадим бэкап этой БД:
```
$ pg_dump -Fd -f HW10_H03 -p 5434 -v "HomeWork 10 Host 03"
pg_dump: last built-in OID is 16383
pg_dump: reading extensions
pg_dump: identifying extension members
pg_dump: reading schemas
pg_dump: reading user-defined tables
pg_dump: reading user-defined functions
pg_dump: reading user-defined types
pg_dump: reading procedural languages
pg_dump: reading user-defined aggregate functions
pg_dump: reading user-defined operators
pg_dump: reading user-defined access methods
pg_dump: reading user-defined operator classes
pg_dump: reading user-defined operator families
pg_dump: reading user-defined text search parsers
pg_dump: reading user-defined text search templates
pg_dump: reading user-defined text search dictionaries
pg_dump: reading user-defined text search configurations
pg_dump: reading user-defined foreign-data wrappers
pg_dump: reading user-defined foreign servers
pg_dump: reading default privileges
pg_dump: reading user-defined collations
pg_dump: reading user-defined conversions
pg_dump: reading type casts
pg_dump: reading transforms
pg_dump: reading table inheritance information
pg_dump: reading event triggers
pg_dump: finding extension tables
pg_dump: finding inheritance relationships
pg_dump: reading column info for interesting tables
pg_dump: flagging inherited columns in subtables
pg_dump: reading partitioning data
pg_dump: reading indexes
pg_dump: flagging indexes in partitioned tables
pg_dump: reading extended statistics
pg_dump: reading constraints
pg_dump: reading triggers
pg_dump: reading rewrite rules
pg_dump: reading policies
pg_dump: reading row-level security policies
pg_dump: reading publications
pg_dump: reading publication membership of tables
pg_dump: reading publication membership of schemas
pg_dump: reading subscriptions
pg_dump: reading large objects
pg_dump: reading dependency data
pg_dump: saving encoding = UTF8
pg_dump: saving standard_conforming_strings = on
pg_dump: saving search_path =
pg_dump: saving database definition
pg_dump: dumping contents of table "public.test1"
pg_dump: dumping contents of table "public.test2"
postgres@voxp-db-1:~$ ls -la HW10_H03
total 20
drwx------  2 postgres postgres 4096 May 25 16:56 ./
drwxr-x--- 11 postgres postgres 4096 May 25 16:56 ../
-rw-rw----  1 postgres postgres  108 May 25 16:56 3297.dat.gz
-rw-rw----  1 postgres postgres   96 May 25 16:56 3298.dat.gz
-rw-rw----  1 postgres postgres 2442 May 25 16:56 toc.dat
```
Резервная копия базы создана успешно.

7. Займёмся горячим реплицированием. Создадим очередной кластер (à la виртуальная машина): `"pg_createcluster 15 host_04"`. Параметры этого кластера менять не надо и запускать сейчас тоже, а вот в `pg_hba.conf` третьей машины надо указать возможность подключения к псевдобазе `replication` без пароля: `"host    replication     postgres             127.0.0.1/32       trust"`, после чего следует перезапустить кластер. Теперь удалим директорию с базой 4 кластера: `rm -rf 15/host_04` и с помощью `pg_basebackup` создадим копию третьей ВМ: 
```
$ pg_basebackup -p 5434 -X stream -c fast -P -v -R -D 15/host_04
pg_basebackup: initiating base backup, waiting for checkpoint to complete
pg_basebackup: checkpoint completed
pg_basebackup: write-ahead log start point: 0/2000028 on timeline 1
pg_basebackup: starting background WAL receiver
pg_basebackup: created temporary replication slot "pg_basebackup_217966"
30597/30597 kB (100%), 1/1 tablespace
pg_basebackup: write-ahead log end point: 0/2000100
pg_basebackup: waiting for background process to finish streaming ...
pg_basebackup: syncing data to disk ...
pg_basebackup: renaming backup_manifest.tmp to backup_manifest
pg_basebackup: base backup completed
```
Бэкап создан, пробуем запустить 4 ВМ: `pg_ctlcluster start 15 host_04`. Ошибок нет, что не может не радовать. Подключаемся к третьему кластеру и проверяем состояние репликации:
```
HomeWork 10 Host 03=# SELECT * FROM pg_stat_replication\gx
-[ RECORD 1 ]----+------------------------------
pid              | 218195
usesysid         | 10
usename          | postgres
application_name | 15/host_04
client_addr      |
client_hostname  |
client_port      | -1
backend_start    | 2024-05-25 17:19:51.065866+03
backend_xmin     |
state            | streaming
sent_lsn         | 0/3000060
write_lsn        | 0/3000060
flush_lsn        | 0/3000060
replay_lsn       | 0/3000060
write_lag        |
flush_lag        |
replay_lag       |
sync_priority    | 0
sync_state       | async
reply_time       | 2024-05-25 17:21:31.296486+03
```
Видно, что четвёртый кластер получает от третьего данные в режиме потоковой репликации.

8. Попробуем добавить по строке в таблицы первого и второго кластеров и посмотреть, какая информация окажется в четвёртом:
```
$ psql -p 5432 -d 'HomeWork 10 Host 01' -c "INSERT INTO test1(id, str) VALUES (12, 'String for replication test');"
INSERT 0 1
$ psql -p 5433 -d 'HomeWork 10 Host 02' -c "INSERT INTO test2(id, str) VALUES (13, 'String For Replication Test');"
INSERT 0 1
$ psql -p 5435 -d 'HomeWork 10 Host 03' -c "SELECT * FROM test1 ORDER BY id;"
 id |             str
----+-----------------------------
  1 | String 1 on host 01
  2 | String 2 on host 01
  3 | String 3 on host 01
  4 | String 4 on host 01
  5 | String 5 on host 01
  6 | String 6 on host 01
  7 | String 7 on host 01
  8 | String 8 on host 01
  9 | String 9 on host 01
 10 | String 10 on host 01
 11 | New string on host 01
 12 | String for replication test
(12 rows)
$ psql -p 5435 -d 'HomeWork 10 Host 03' -c "SELECT * FROM test2 ORDER BY id;"                                       id |             str
----+-----------------------------
  1 | String 1 on host 02
  2 | String 2 on host 02
  3 | String 3 on host 02
  4 | String 4 on host 02
  5 | String 5 on host 02
  6 | String 6 on host 02
  7 | String 7 on host 02
  8 | String 8 on host 02
  9 | String 9 on host 02
 10 | String 10 on host 02
 13 | String For Replication Test
(11 rows)
```
Посредством логической репликации добавленные строки сначала попали в БД третьего кластера, а оттуда физическая репликация перенесла их в четвёртый. Также следует обратить внимание на то, что название базы в четвёртом кластере такое же, как в третьем, поскольку мы создали его побайтную копию.