### Домашнее задание к уроку "Механизм блокировок"


Это домашнее задание будет выполняться на 15 версии PostgreSQL.

1. Создадим отдельную БД: `CREATE DATABASE "HomeWork 07";` , в ней - таблицу для опытов: `CREATE TABLE accounts(id serial, amount int);` и заполним её данными: `INSERT INTO accounts(amount) SELECT * FROM generate_series(1000,10000,1000);`.

2. Для того, чтобы в журнале вообще начала появляться информация о блокировках, мы должны включить параметр `log_lock_waits`. В описании параметра говорится: "Регулирует, будет ли создаваться журнальное сообщение в случае, если сессия ожидает получения блокировки более, чем `deadlock_timeout`". Значит, нам надо задать оба эти параметра:
```
ALTER SYSTEM SET log_lock_waits TO 1;
ALTER SYSTEM SET deadlock_timeout TO '200ms';
SELECT pg_reload_conf();
``` 

3. Простейший способ организовать блокировку - попытаться обновить одну и ту же строку в разных сессиях. \
Сессия №1:
```
HomeWork 07=# SELECT pg_backend_pid();
 pg_backend_pid
----------------
         166714
(1 row)

HomeWork 07=# BEGIN;
BEGIN
HomeWork 07=*# UPDATE accounts SET amount=amount+100 WHERE id=1;
UPDATE 1
```
Транзакцию пока не завершаем.\
Сессия №2.\
Тут можно обойтись и без явного создания транзакции:
```
HomeWork 07=# SELECT pg_backend_pid();
 pg_backend_pid
----------------
         167757
(1 row)

HomeWork 07=# UPDATE accounts SET amount=amount + 200 WHERE id=1;
...
```
Второй UPDATE зависает в ожидании завершения блокировки, наложенной более ранним UPDATE'ом:
```
2024-05-07 23:06:01.164 MSK [167757] postgres@HomeWork 07 LOG:  process 167757 still waiting for ShareLock on transaction 1178 after 200.129 ms
2024-05-07 23:06:01.164 MSK [167757] postgres@HomeWork 07 DETAIL:  Process holding the lock: 166714. Wait queue: 167757.
2024-05-07 23:06:01.164 MSK [167757] postgres@HomeWork 07 CONTEXT:  while updating tuple (0,1) in relation "accounts"
2024-05-07 23:06:01.164 MSK [167757] postgres@HomeWork 07 STATEMENT:  UPDATE accounts SET amount=amount + 200 WHERE id=1;
```
В первой же строке фрагмента видно, что не прошло и 200 мс, как была обнаружена и зафиксирована безуспешная попытка процесса 167757 получить блокировку.
После фиксации первой транзакции мы видим в журнале, что процесс 167757 добился-таки своего:
```
2024-05-07 23:14:03.441 MSK [167757] postgres@HomeWork 07 LOG:  process 167757 acquired ShareLock on transaction 1178 after 482476.894 ms
2024-05-07 23:14:03.441 MSK [167757] postgres@HomeWork 07 CONTEXT:  while updating tuple (0,1) in relation "accounts"
2024-05-07 23:14:03.441 MSK [167757] postgres@HomeWork 07 STATEMENT:  UPDATE accounts SET amount=amount + 200 WHERE id=1;
```
 
4. Попробуем тройное обновление.\
Сессия №1:
```
HomeWork 07=# SELECT pg_backend_pid();
 pg_backend_pid
----------------
         166714
(1 row)

HomeWork 07=# BEGIN;
BEGIN
HomeWork 07=*# UPDATE accounts SET amount=amount + 100 WHERE id = 2;
UPDATE 1
```
Транзакцию не завершаем.\
Сессия №2:
```
HomeWork 07=# SELECT pg_backend_pid();
 pg_backend_pid
----------------
         167757
(1 row)

HomeWork 07=# UPDATE accounts SET amount=amount + 200 WHERE id = 2;
...
```
UPDATE в этой сессии ждёт свою блокировку.\
Сессия №3:
```
HomeWork 07=# SELECT pg_backend_pid();
 pg_backend_pid
----------------
         169213
(1 row)

HomeWork 07=# UPDATE accounts SET amount = amount + 300 WHERE id = 2;
...
```
Здесь у нас тоже возникает ожидание.\
Смотрим в `pg_locks`:
```
HomeWork 07=# SELECT locktype, relation::regclass::text "TabName", tuple, virtualxid, transactionid, pid, mode, granted FROM pg_locks ORDER BY pid, locktype;
   locktype    | TabName  | tuple | virtualxid | transactionid |  pid   |       mode       | granted
---------------+----------+-------+------------+---------------+--------+------------------+---------
 relation      | accounts |       |            |               | 166714 | RowExclusiveLock | t
 transactionid |          |       |            |          1180 | 166714 | ExclusiveLock    | t
 virtualxid    |          |       | 4/67       |               | 166714 | ExclusiveLock    | t
 relation      | accounts |       |            |               | 167757 | RowExclusiveLock | t
 transactionid |          |       |            |          1181 | 167757 | ExclusiveLock    | t
 transactionid |          |       |            |          1180 | 167757 | ShareLock        | f
 tuple         | accounts |     2 |            |               | 167757 | ExclusiveLock    | t
 virtualxid    |          |       | 5/9        |               | 167757 | ExclusiveLock    | t
 relation      | accounts |       |            |               | 169213 | RowExclusiveLock | t
 transactionid |          |       |            |          1182 | 169213 | ExclusiveLock    | t
 tuple         | accounts |     2 |            |               | 169213 | ExclusiveLock    | f
 virtualxid    |          |       | 6/5        |               | 169213 | ExclusiveLock    | t
 relation      | pg_locks |       |            |               | 169613 | AccessShareLock  | t
 virtualxid    |          |       | 7/48       |               | 169613 | ExclusiveLock    | t
(14 rows)

```
Для начала мы видим, что процесс 166714 устанавливает блокировку на таблицу `accounts` в режиме `RowExclusiveLock`. Это видно по признаку `granted=true`. Кроме этого, он блокирует обычный и виртуальный идентификатор транзакции, являющейся целью блокировки. 

Далее идут блокировки процесса 167757. Он успешно блокирует свою строку в таблице (RowExclusiveLock). Хотя, как мне кажется, правильнее тут сказать - свою версию обновляемой строки. Также успешно блокируется собственный идентификатор транзакции (1181). Далее у него не получается (`granted=false`) сделать ShareLock транзакции 1180 процесса 166714, так как последний сам блокирует её в режиме ExclusiveLock. Затем процесс 167757 успешно блокирует кортеж №2 (такие блокировки ещё называют блокировками версии строки. Они нужны для установки приоритета среди нескольких транзакций, ожидающих блокировку одной и той же строки) и виртуальный идентификатор собственной транзакции. 

Ну и наконец, третий процесс - 169213. Он успешно блокирует свою версию обновляемой строки и собственную транзакцию с номером 1182. Кортеж №2 не получается заблокировать, так как это уже сделал процесс 167757. Также успешна блокировка виртуального идентификатора собственной транзакции (6/5). 

В последних двух строках мы видим, как в той сессии, в которой выполняется запрос к `pg_locks` (pid=169613) возникает типичная для `SELECT` блокировка вида "ACCESS SHARE". Ну и какой же процесс откажется от удовольствия сделать `ExclusiveLock` виртуальному идентификатору собственной транзакции (7/48)!

5. Изобретём ситуацию с тройной взаимоблокировкой. Пожалуй, самой простой реализацией подобного случая будет моделирование ситуации с переводом средств:
- транзакция `"А"` уменьшает счёт №1 и пополняет счёт №2;
- транзакция `"Б"` уменьшает счёт №2 и пополняет счёт №3;
- транзакция `"В"` уменьшает счёт №3 и пополняет счёт №1. \
  Ну что же, воспользуемся всё той же таблицей `accounts` и посмотрим, что из этого выйдет. Начнём транзакцию в каждой сессии и сделаем списание средств с соответствующих счётов.\
Сессия №1:
```
HomeWork 07=# SELECT pg_backend_pid();
 pg_backend_pid
----------------
          90869
(1 row)
HomeWork 07=# BEGIN;
BEGIN
HomeWork 07=*# UPDATE accounts SET amount = amount - 100 WHERE id = 1;
UPDATE 1
```
Сессия №2:
```
HomeWork 07=# SELECT pg_backend_pid();
 pg_backend_pid
----------------
          90911
(1 row)
HomeWork 07=# BEGIN;
BEGIN
HomeWork 07=*# UPDATE accounts SET amount = amount - 200 WHERE id = 2;
UPDATE 1
```
Сессия №3:
```
HomeWork 07=# SELECT pg_backend_pid();
 pg_backend_pid
----------------
          90937
(1 row)
HomeWork 07=# BEGIN;
BEGIN
HomeWork 07=*# UPDATE accounts SET amount = amount - 300 WHERE id = 3;
UPDATE 1
```
Теперь перейдём к пополнению счетов. \
Сессия №1:
```
HomeWork 07=*# UPDATE accounts SET amount = amount + 100 WHERE id = 2;
...
```
UPDATE зависает. \
Сессия №2:
```
HomeWork 07=*# UPDATE accounts SET amount = amount + 200 WHERE id = 3;
...
```
UPDATE зависает. \
Сессия №3:
```
HomeWork 07=*# UPDATE accounts SET amount = amount + 300 WHERE id = 1;
```
Через секунду, в соответствии со значением `"deadlock_timeout"`, PostgreSQL определяет возникновение взаимоблокировки и разрубает гордиев узел. \
Сессия №3:
```
ERROR:  deadlock detected
DETAIL:  Process 90937 waits for ShareLock on transaction 1193; blocked by process 90869.
Process 90869 waits for ShareLock on transaction 1194; blocked by process 90911.
Process 90911 waits for ShareLock on transaction 1195; blocked by process 90937.
HINT:  See server log for query details.
CONTEXT:  while updating tuple (0,12) in relation "accounts"
HomeWork 07=!#
```
Индикация начавшейся транзакции `"*"` сменяется индикацией ошибочной транзакции: `"!"`.
Поскольку PostgreSQL прервал транзакцию в сессии №3, то у сессии №2 появилась возможность успешно завершить операцию перевода 200 тугриков со счёта №2 на счёт №3. \
Сессия №2:
```
HomeWork 07=*# UPDATE accounts SET amount = amount + 200 WHERE id = 3;
UPDATE 1
HomeWork 07=*#
```
А сессия №1 будет находиться в состоянии ожидания, пока сессия №2 не зафиксирует у себя изменения. Сделаем же это: `"COMMIT;"` тут же и увидим, что первая сессия также справилась с переводом средств, после чего завершим транзакцию и в ней.\
Сессия №1:
```
HomeWork 07=*# UPDATE accounts SET amount = amount + 100 WHERE id = 2;
UPDATE 1
HomeWork 07=*# COMMIT;
COMMIT
HomeWork 07=#
```
6. Насчёт взаимной блокировки двух `UPDATE`: да, такая ситуация может случиться. Полагаю, что это может произойти, когда два обновления перемещаются от строки к строке, двигаясь навстречу друг другу: скажем первый `UPDATE` использует физический порядок строк, а второй  - использует индекс, построенный по убыванию. Но так как в команде `UPDATE` не предусмотрено явного указания порядка перебора строк, это может произойти неявно при установке параметра `"enable_seqscan=false"`. Однако, смоделировать подобную ситуацию у меня не получилось.