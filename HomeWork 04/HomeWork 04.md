### Домашнее задание к уроку "Работа с базами данных, пользователями и правами"

Для этого урока была использована виртуальная машина, созданная и настроенная ранее, для выполнения "домашки" к урокам 1 и 3, а именно: Ubuntu 22.04 LTS, с установленным PostgreSQL версий 14 и 15 и возможностью подключения по SSH.

1) `sudo -u postgres pg_lsclusters`
```
Ver Cluster Port Status Owner    Data directory              Log file
14  main    5432 online postgres /var/lib/postgresql/14/main /var/log/postgresql/postgresql-14-main-%Y-%m-%d_%H%M%S.log
15  main    5432 down   postgres /var/lib/postgresql/15/main /var/log/postgresql/postgresql-15-main-%Y-%m-%d_%H%M%S.log
```
2) `sudo -iu postgres psql -d postgres`
3) `CREATE DATABASE "Lesson 04";`
4) `\c "Lesson 04"`
```
psql (15.6 (Ubuntu 15.6-1.pgdg22.04+1), server 14.11 (Ubuntu 14.11-1.pgdg22.04+1))
You are now connected to database "Lesson 04" as user "postgres".
```
5) `CREATE SCHEMA testnm;`
6) `CREATE TABLE t1(c1 int);`
7) `INSERT INTO t1(c1) VALUES(1);`
8) `CREATE ROLE readonly;`
9) `GRANT CONNECT ON DATABASE "Lesson 04" TO readonly;`
10) `GRANT USAGE ON SCHEMA testnm TO readonly;`
11) `GRANT SELECT ON ALL TABLES IN SCHEMA testnm TO readonly;`
12) `CREATE ROLE testread LOGIN PASSWORD '123456q';`
13) `GRANT readonly TO testread;`
14) Так сразу подключиться к БД `"Lesson 04"` под пользователем `testread` не получится, сначала надо прописать разрешение на подключение в `pg_hba.conf`:
```
host    "Lesson 04"      testread       127.0.0.1/32            scram-sha-256
```
а затем -- перечитать конфигурационные файлы: `"SELECT pg_reload_conf();"`
Подключаемся к БД из новой сессии: `psql -h localhost -U testread -d 'Lesson 04'` 

15) Подключились:
```
Lesson 04=> \conninfo
You are connected to database "Lesson 04" as user "testread" on host "localhost" (address "127.0.0.1") at port "5432".
SSL connection (protocol: TLSv1.3, cipher: TLS_AES_256_GCM_SHA384, compression: off)
```
Выполняем `SELECT * FROM t1;`

16)  Неудачно -- `"ERROR:  permission denied for table t1"`
17)  `\dt`
```
List of relations
 Schema | Name | Type  |  Owner
--------+------+-------+----------
 public | t1   | table | postgres
(1 row)
```
`\dp`
```
 Access privileges
 Schema | Name | Type  | Access privileges | Column privileges | Policies
--------+------+-------+-------------------+-------------------+----------
 public | t1   | table |                   |                   |
(1 row)
```

Мы видим, что владельцем таблицы t1 является пользователь `postgres`, а в колонке "Права доступа" -- пусто, что фактически обозначает наличие прав на таблицу только у её владельца, поэтому `testuser` не может прочесть её содержимое.

18) (а так же 19-21) Право `SELECT` было дано на схему `testnm`, а наша таблица создалась в схеме `public`.
19) 
20) 
21) 
22) Вернёмся в первую сессию с БД, нажав комбинацию Alt-Tab.
23) `DROP TABLE t1;`
24) `CREATE TABLE testnm.t1(c1 int);`
25) `INSERT INTO testnm.t1(c1) VALUES (2);`
26) Переключаемся в сессию пользователя `testread`.
27) `SELECT * FROM testnm.t1;`
28) Неудачно -- `"ERROR:  permission denied for table t1"`.
29) Так вышло потому, что команда, которую мы выполнили в п.11, применилась лишь к таблицам, уже существовавшим в схеме на момент её запуска. В нашем случае она не применилась ни к чему, так как схема `testnm` в тот момент была пуста, а таблицу `testnm.t1` мы создали только что, и действие команды на неё не распространилось.
30) Раздача прав на "будущие" объекты БД осуществляется с помощью команды `ALTER DEFAULT PRIVILEGES`. В нашем случае мы должны выполнить от имени `postgres` вот такую команду: `ALTER DEFAULT PRIVILEGES FOR ROLE postgres IN SCHEMA testnm GRANT SELECT ON TABLES TO testread;`
```
Lesson 04=# \ddp
            Default access privileges
  Owner   | Schema | Type  |  Access privileges
----------+--------+-------+---------------------
 postgres | testnm | table | testread=r/postgres
(1 row)
```
31) `SELECT * FROM testnm.t1;`
32) Опять-таки нет: `"ERROR:  permission denied for table t1"`.
33) С помощью `ALTER DEFAULT PRIVILEGES` мы определили правила выдачи доступов только для тех объектов, которые будут созданы впоследствии, но не для тех, что уже существуют, поэтому для обеспечения доступа `testread` к `testnm.t1` нам не остаётся ничего иного, как явно дать эти права. Причём мы можем дать права как пользователю, так и роли, членом которой `testread` является. Выберем второй вариает и выполним в сессии пользователя `postgres` : `"GRANT SELECT ON testnm.t1 TO readonly;"`. Можно было бы повторить команду из п. 11, но у нас в схеме `testnm` одна-единственная таблица, поэтому выдадим права точечно.
34) `SELECT * FROM testnm.t1;` 
```
 c1
----
  2
(1 row)
```
35) (и 36) Получилось!
36) 
37) Пробуем:
```
Lesson 04=> CREATE TABLE t2(c1 int);
CREATE TABLE
Lesson 04=> INSERT INTO t2(c1) VALUES (3);
INSERT 0 1
```
38) `\dn+`
```                          List of schemas
  Name  |  Owner   |  Access privileges   |      Description
--------+----------+----------------------+------------------------
 public | postgres | postgres=UC/postgres+| standard public schema
        |          | =UC/postgres         |
 testnm | postgres | postgres=UC/postgres+|
        |          | readonly=U/postgres  |
(2 rows)
```
Мы видим, что схема `public` по умолчанию имеет права, разрешающие группе `PUBLIC` создание и использование любых объектов в этой схеме: `"=UC/postgres"`.

39) `REVOKE CREATE,USAGE ON SCHEMA public FROM PUBLIC;` 
40) Справился сам, так как отзываю подобные права по умолчанию в своих базах.
41) `CREATE TABLE t3(c1 int);`
```
ERROR:  no schema has been selected to create in
LINE 1: CREATE TABLE t3(c1 int);
```
Поскольку `testread` теперь не имеет прав на создание объектов ни в одной из схем, то при попытке создания таблицы без явного указания схемы, получаем вот такую ошибку. Впрочем, указание схемы ничего не меняет, разве что появляется более вразумительное сообщение об ошибке:
```
Lesson 04=> CREATE TABLE public.t3(c1 int);
ERROR:  permission denied for schema public
LINE 1: CREATE TABLE public.t3(c1 int);
                     ^
Lesson 04=> CREATE TABLE testnm.t3(c1 int);
ERROR:  permission denied for schema testnm
LINE 1: CREATE TABLE testnm.t3(c1 int);
                     ^
```
42) Пробуем вставить строку в существующую таблицу: `INSERT INTO t2(c1) VALUES(4);`
```
ERROR:  relation "t2" does not exist
LINE 1: INSERT INTO t2(c1) VALUES(4);
                    ^
```
На самом деле таблица `t2` никуда не делась из схемы `public`, но без права `USAGE`  пользователь `testread` ничего не видит:
```
Lesson 04=> \dt
Did not find any relations.
```
