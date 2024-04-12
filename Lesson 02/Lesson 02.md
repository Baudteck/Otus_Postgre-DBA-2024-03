### Домашнее задание к уроку "Установка и настройка PostgreSQL в контейнере Docker"


В качестве полигона для выполнения этой работы я использовал платформу Cloud.ru ("Сбербанк").
 
Создание виртуалки, назачение ей внешнего IP-адреса и установка Docker особой сложности не вызвали. 
 
После того, как была создана директория для будущей БД: `sudo mkdir -p /var/lib/postgres/data`, я вспомнил, что внутри контейнера Docker пользователь postgres имеет числовой идентификатор 999, поэтому залогом успешного выполнения ДЗ стало выполнение команды: `sudo chown 999:999 -R /var/lib/postgres`. Если бы я не поменял владельца, то после удаления контейнера в БД не сохранились бы изменения. Ну и с целью безопасности были изменены права доступа на папку: `sudo chmod 700 -R /var/lib/postgres`.
 
Перед тем, как локально запустить наши контейнеры, создадим отдельную сеть, с помощью которой они будуть видеть друг друга: `sudo docker network create pg_network`.

Далее мы запускаем серверный контейнер:
```
sudo docker run -d --hostname pg_docker_srv --name pg_docker_srv --network pg_network -v /var/lib/postgres/data:/var/lib/postgresql/data -e POSTGRES_PASSWORD=a5r8z3 -p 0.0.0.0:5432:5432 postgres:15
``` 
Из обязательного: 
* мы указываем пароль пользователя `postgres`, иначе контейнер не запустится; 
* cогласно требований к ДЗ я монтирую папку файловой системы виртуальной машины к файловой системе контейнера: `-v /var/lib/postgres/data:/var/lib/postgresql/data`;
* делаю проброс порта `-p 0.0.0.0:5432:5432`, чтобы на третьем шаге можно было подключиться к нашем контейнеру извне.

Для удобства подключения в сети Docker указываю имя хоста `--hostname pg_docker_srv`, иначе придётся с помощью `docker inspect` выяснять IP-адрес контейнера внутри сети `pg_network`. 

Поскольку на данной виртуальной машине не установлен PostgreSQL, для подключения к нашей БД мы запустим другой контейнер, клиентский:
```
sudo docker run --rm -it --network=pg_network postgres:15 psql -h pg_docker_srv -U postgres
```

Увидев приглашение, вводим пароль и подключаемся к БД.

Оставим же в кластере свой след:
```
CREATE DATABASE "Lesson 02";
\c "Lesson 02"
Lesson 02=# CREATE TABLE conninfo(id serial, conninfo text);
CREATE TABLE
Lesson 02=# \conninfo
You are connected to database "Lesson 02" as user "postgres" on host "pg_docker_srv" (address "172.19.0.2") at port "5432".
Lesson 02=# INSERT INTO conninfo(conninfo) VALUES('You are connected to database "Lesson 02" as user "postgres" on host "pg_docker_srv" (address "172.19.0.2") at port "5432"');
INSERT 0 1
``` 
Самое время отключиться и попробовать зайти в БД через другую дверь. "СберОблако" любезно выделило мне статический реальный IP, по которому я попробую подключиться, предварительно настроив проброс порта 5432: `psql -h 176.123.160.31 -U postgres -d "Lesson 02"`. Вижу приглашение, ввожу пароль и успешно подключаюсь к вновь созданной базе:
```
psql -h 176.123.160.31 -U postgres -d "Lesson 02"
Password for user postgres:
psql (15.6 (Ubuntu 15.6-1.pgdg20.04+1))
Type "help" for help.

Lesson 02=#
```
Добавим ещё одну строку в таблицу:
```
Lesson 02=# \conninfo
You are connected to database "Lesson 02" as user "postgres" on host "176.123.160.31" at port "5432".
Lesson 02=# INSERT INTO conninfo(conninfo) VALUES('You are connected to database "Lesson 02" as user "postgres" on host "176.123.160.31" at port "5432"');
INSERT 0 1

```
На этом закончим и вернёмся в виртуальную машину.
Остановим и удалим серверный контейнер:
```
hadrian@pgdba-sc-01:~ $ sudo docker ps
CONTAINER ID   IMAGE         COMMAND                  CREATED          STATUS          PORTS                    NAMES
c4cb2ada2883   postgres:15   "docker-entrypoint.s…"   44 minutes ago   Up 44 minutes   0.0.0.0:5432->5432/tcp   pg_docker_srv
hadrian@pgdba-sc-01:~ $ sudo docker stop c4cb2ada2883
c4cb2ada2883
hadrian@pgdba-sc-01:~ $ sudo docker rm c4cb2ada2883
c4cb2ada2883

``` 
Теперь запускаем контейнер в БД повторно:
```
sudo docker run -d -h pg_docker_srv --name pg_docker_srv --network pg_network -v /var/lib/postgres/data:/var/lib/postgresql/data -e POSTGRES_PASSWORD=a5r8z3 -p 0.0.0.0:5432:5432 postgres:15
```
Подключаемся с целью проверки сохранности данных:\
`sudo docker run --rm -it --network=pg_network postgres:15 psql -h pg_docker_srv -U postgres -d "Lesson 02" -c 'SELECT * FROM conninfo;'`\
Вводим пароль и облегчённо выдыхаем:
```
 id |                                                          conninfo
----+----------------------------------------------------------------------------------------------------------------------------
  1 | You are connected to database "Lesson 02" as user "postgres" on host "pg_docker_srv" (address "172.19.0.2") at port "5432"
  2 | You are connected to database "Lesson 02" as user "postgres" on host "176.123.160.31" at port "5432"
(2 rows)
```
### Quod erat demonstrandum.
