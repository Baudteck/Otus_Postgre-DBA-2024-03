### Домашнее задание к уроку "Бэкапы"

1. Для выполнения этого домашнего задания использовалась виртуальная машина с установленным на ней PostgreSQL 15 версии.
2. Создаём базу, подключаемся к ней, создаём и наполняем таблицу:
```
postgres=# CREATE DATABASE "HomeWork 09";
postgres=# \c "HomeWork 09"
You are now connected to database "HomeWork 09" as user "postgres".
HomeWork 09=# CREATE TABLE backup_01(id int, str text);
CREATE TABLE
HomeWork 09=# INSERT INTO backup_01(id, str) SELECT id, gen_random_uuid() FROM generate_series(1,100) id;
INSERT 0 100
```
3. Не покидая БД, создадим каталог для размещения в нём бэкапа таблицы: `\! mkdir tbl.backup`.
4. Выгрузим табличные данные: `COPY backup_01 TO '/var/lib/postgresql/tbl.backup/backup_01.data';`.
5. Восстановим даные в другую таблицу аналогичной структуры:
```
HomeWork 09=# CREATE TABLE backup_02 (LIKE backup_01);
CREATE TABLE
HomeWork 09=# COPY backup_02 FROM '/var/lib/postgresql/tbl.backup/backup_01.data';
COPY 100
```
6. Сравним содержимое таблиц:
```
HomeWork 09=# SELECT * FROM backup_01 ORDER BY id LIMIT 10;
 id |                 str
----+--------------------------------------
  1 | ca6f3c7a-aa32-43d5-89e8-0a39543e8b20
  2 | 2b591ae2-7073-413f-bcee-04df2e5b86fe
  3 | ef20bab9-06ec-403c-a284-050fa6d11a20
  4 | 91dcef2f-9b96-4c84-8260-e6a2be4615e2
  5 | cce02e6f-752d-4aad-b69f-9e6b9a444f51
  6 | 0d0c79a5-b50d-479a-8465-ba3f26d874b5
  7 | c38935cd-6126-4d4f-ac08-254cbdf4ff8c
  8 | 747d6ffe-1187-4aa1-8ce8-4c6c849462eb
  9 | 7e126e57-eeda-4718-987a-b0e4abba44b8
 10 | 7aff6b0b-8cd8-4493-ac52-c2dcdee281df
(10 rows)

HomeWork 09=# SELECT * FROM backup_02 ORDER BY id LIMIT 10;
 id |                 str
----+--------------------------------------
  1 | ca6f3c7a-aa32-43d5-89e8-0a39543e8b20
  2 | 2b591ae2-7073-413f-bcee-04df2e5b86fe
  3 | ef20bab9-06ec-403c-a284-050fa6d11a20
  4 | 91dcef2f-9b96-4c84-8260-e6a2be4615e2
  5 | cce02e6f-752d-4aad-b69f-9e6b9a444f51
  6 | 0d0c79a5-b50d-479a-8465-ba3f26d874b5
  7 | c38935cd-6126-4d4f-ac08-254cbdf4ff8c
  8 | 747d6ffe-1187-4aa1-8ce8-4c6c849462eb
  9 | 7e126e57-eeda-4718-987a-b0e4abba44b8
 10 | 7aff6b0b-8cd8-4493-ac52-c2dcdee281df
(10 rows)

``` 
7. Действия с утилитой `pg_dump` будем выполнять непосредственно в оболочке операционной системы под пользователем `postgres`. Создадим бэкап исходной базы, содержащей 2 таблицы: `pg_dump -Fd -f 'HomeWork 09' "HomeWork 09"`. При этом, по умолчанию, будет произведено сжатие файлов бэкапа. Затем создадим базу, в которую произведём частичное восстановление данных: `psql -c 'CREATE DATABASE "HomeWork 09 Restored";'`. 
8. Восстановим только одну из таблиц: `pg_restore -d "HomeWork 09 Restored" -t backup_02 'HomeWork 09'`
9. Посмотрим, что у нас получилось:
```
$ psql -d 'HomeWork 09 Restored' -c 'SELECT * FROM backup_02 ORDER BY id LIMIT 10;'
 id |                 str
----+--------------------------------------
  1 | ca6f3c7a-aa32-43d5-89e8-0a39543e8b20
  2 | 2b591ae2-7073-413f-bcee-04df2e5b86fe
  3 | ef20bab9-06ec-403c-a284-050fa6d11a20
  4 | 91dcef2f-9b96-4c84-8260-e6a2be4615e2
  5 | cce02e6f-752d-4aad-b69f-9e6b9a444f51
  6 | 0d0c79a5-b50d-479a-8465-ba3f26d874b5
  7 | c38935cd-6126-4d4f-ac08-254cbdf4ff8c
  8 | 747d6ffe-1187-4aa1-8ce8-4c6c849462eb
  9 | 7e126e57-eeda-4718-987a-b0e4abba44b8
 10 | 7aff6b0b-8cd8-4493-ac52-c2dcdee281df
(10 rows)

```