### Домашнее задание к уроку "Триггеры, поддержка заполнения витрин"

1. Для выполнения ДЗ была создана отдельная БД с именем `HomeWork 14`, в которую был загружен скрипт, доступный по ссылке: https://disk.yandex.ru/d/l70AvknAepIJXQ. Для удобства дальнейшей работы в таблице `goods` был переименован один из столбцов: `"ALTER TABLE pract_functions.goods RENAME goods_id TO good_id;"`. Кроме этого, был изменен путь поиска схем: `"ALTER ROLE postgres IN DATABASE "HomeWork 14" SET search_path TO pract_functions, public;"`

2. В соответcтвии с заданием была написана триггерная функция:
```
CREATE OR REPLACE FUNCTION pract_functions.sales_trigger_function()
RETURNS TRIGGER
AS
$TRIG$
DECLARE 
     article character varying(63);
     price  numeric(12,2);
     totals numeric(12,2);
BEGIN
    IF TG_OP = 'INSERT' THEN
--     ищем наименование и цену продаваемого товара	
       SELECT INTO article, price
               g.good_name, g.good_price
          FROM
	       pract_functions.goods g
	 WHERE
	       g.good_id = NEW.good_id; 
--     осуществляем поиск в агрегированной таблице. В зависимости от его результатов обновляем либо вставляем строку
       PERFORM  
               gsm.good_name 
       FROM   
               pract_functions.goods_sum_mart gsm 
       WHERE 
              gsm.good_name = article;
       IF FOUND THEN
	  UPDATE pract_functions.goods_sum_mart SET sum_sale = sum_sale + price * NEW.sales_qty WHERE good_name = article;
       ELSE
	  INSERT INTO pract_functions.goods_sum_mart (good_name, sum_sale) VALUES (article, price * NEW.sales_qty);
       END IF;
       RETURN NEW;
    ELSIF TG_OP = 'DELETE' THEN
       SELECT INTO article, price
               g.good_name, g.good_price
          FROM
               pract_functions.goods g
         WHERE
               g.good_id = OLD.good_id;	
      UPDATE pract_functions.goods_sum_mart SET sum_sale = sum_sale - price * OLD.sales_qty WHERE good_name = article;
--    в случае, если после удаления очередной записи о продаже, сумма продаж этого товара становится нулевой,
--    удаляем и саму запись из агрегированной таблицы.   
      SELECT INTO totals
            sum_sale
        FROM 
	    pract_functions.goods_sum_mart
       WHERE
            good_name = article;
      IF totals = 0 THEN
	 DELETE FROM pract_functions.goods_sum_mart WHERE good_name = article;
      END IF;	
      RETURN OLD;
    ELSEIF TG_OP = 'UPDATE' THEN
--   запрещаем смену кода товара	
	  IF NEW.good_id <> OLD.good_id THEN
	     RAISE EXCEPTION 'Changing item code is not allowed!';
             RETURN NULL;	
	  END IF;
          SELECT INTO article, price
               g.good_name, g.good_price
          FROM
               pract_functions.goods g
          WHERE
               g.good_id = OLD.good_id;
          UPDATE pract_functions.goods_sum_mart SET sum_sale = sum_sale + price * (NEW.sales_qty - OLD.sales_qty) WHERE good_name = article;
	 RETURN NEW;
    END IF;	
END
$TRIG$ 
LANGUAGE plpgsql;
```
Комментарии в коде функции поясняют основные моменты её работы. Подключение триггера к таблице выполняется следующей командой: `"CREATE OR REPLACE TRIGGER sales_trigger AFTER INSERT OR UPDATE OR DELETE ON pract_functions.sales FOR EACH ROW EXECUTE FUNCTION pract_functions.sales_trigger_function();"`

3. Проверим, что у нас получилось. Очистим данные о продажах и добавим пару товаров:
```
HomeWork 14=# DELETE FROM sales;
DELETE 4
HomeWork 14=# INSERT INTO goods (good_id, good_name, good_price) VALUES (3, 'Beer', 100);
INSERT 0 1
HomeWork 14=# INSERT INTO goods (good_id, good_name, good_price) VALUES (4, 'Road bike', 250000);
INSERT 0 1
```
Выполняем запрос и видим, что таблица пуста:
```
HomeWork 14=# SELECT * FROM goods_sum_mart;
 good_name | sum_sale 
-----------+----------
(0 rows)
```	
Внесём информацию о продажах:
```
HomeWork 14=# INSERT INTO sales (good_id, sales_qty) VALUES (1, 10);
INSERT 0 1
HomeWork 14=# INSERT INTO sales (good_id, sales_qty) VALUES (2, 1);
INSERT 0 1
HomeWork 14=# INSERT INTO sales (good_id, sales_qty) VALUES (3, 100);
INSERT 0 1
HomeWork 14=# INSERT INTO sales (good_id, sales_qty) VALUES (3, 25);
INSERT 0 1
HomeWork 14=# INSERT INTO sales (good_id, sales_qty) VALUES (4, 10);
INSERT 0 1
HomeWork 14=# INSERT INTO sales (good_id, sales_qty) VALUES (4, 1);
INSERT 0 1
```
Смотрим, что у нас находится таблицах `sales` и `goods_sum_mart`:
```
HomeWork 14=# SELECT * FROM sales ORDER BY sales_id;
 sales_id | good_id |          sales_time           | sales_qty 
----------+---------+-------------------------------+-----------
       41 |       1 | 2024-07-08 17:36:11.459649+03 |        10
       42 |       2 | 2024-07-08 17:36:11.461414+03 |         1
       43 |       3 | 2024-07-08 17:36:11.462422+03 |       100
       44 |       3 | 2024-07-08 17:36:11.463451+03 |        25
       45 |       4 | 2024-07-08 17:36:11.464638+03 |        10
       46 |       4 | 2024-07-08 17:36:11.465591+03 |         1
(6 rows)

HomeWork 14=# SELECT * FROM goods_sum_mart;
        good_name         |   sum_sale   
--------------------------+--------------
 Спички хозяйственные     |         5.00
 Автомобиль Ferrari FXX K | 185000000.01
 Beer                     |     12500.00
 Road bike                |   2750000.00
(4 rows)
```

Удалим информацию о продаже автомобиля и посмотрим на агрегированные данные:
```
HomeWork 14=# DELETE FROM sales WHERE sales_id = 42;
DELETE 1
HomeWork 14=# SELECT * FROM goods_sum_mart;
      good_name       |  sum_sale  
----------------------+------------
 Спички хозяйственные |       5.00
 Beer                 |   12500.00
 Road bike            | 2750000.00
(3 rows)
```
Видим, что строка `"Автомобиль Ferrari FXX K"` исчезла.
Обновим информацию о покупках пива и велосипедов:
```	
HomeWork 14=# UPDATE sales SET sales_qty = 50 WHERE sales_id = 44;
UPDATE 1
HomeWork 14=# UPDATE sales SET sales_qty = 1 WHERE sales_id = 45;
UPDATE 1
HomeWork 14=# SELECT * FROM goods_sum_mart;
      good_name       | sum_sale  
----------------------+-----------
 Спички хозяйственные |      5.00
 Beer                 |  15000.00
 Road bike            | 500000.00
(3 rows)
```
Информация корректно обновилась.

4. Ответ на вопрос со звёздочкой: 
- использование витрины с данными может быть более безопасным для основной БД, особенно если агрегированная информация будут находиться совсем на другом сервере. В этом случае взлом этого сервера никак не отразится на гораздо более конфиденциальных данных, которые находятся на сервере продаж;
- витрина данных хранит минимум необходимой информации, в то время как содержимое таблицы продаж может многое рассказать опытному человеку; 
- объём хранимой информации в витрине данных обычно гораздо меньше, чем в базовой таблице или таблицах, из-за чего увеличивается скорость доступа и уменьшается время анализа данных; 
- несложно будет поменять структуру выводимой информации или даже создать новую витрину в случае изменения требований пользователей;
- eсли допустить возможность изменения цен, то понадобится триггер на таблицу `goods`, чтобы это отразилось в таблице продаж и как следствие - в витрине данных. 
