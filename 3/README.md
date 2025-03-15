## 1. Создать таблицу с текстовым полем и заполнить случайными или сгенерированными данным в размере 1 млн строк
```
CREATE TABLE table_1(value varchar(50));

INSERT INTO table_1 AS t (value)
SELECT *
FROM generate_series(1, 1000000);
```

## 2. Посмотреть размер файла с таблицей
```
SELECT pg_size_pretty(pg_total_relation_size('table_1'::regclass));
```
> 35 MB

## 3. 5 раз обновить все строчки и добавить к каждой строчке любой символ
```
UPDATE table_1 t set value = t.value || 'a';
UPDATE table_1 t set value = t.value || 'b';
UPDATE table_1 t set value = t.value || 'c';
UPDATE table_1 t set value = t.value || 'd';
UPDATE table_1 t set value = t.value || 'e';
```

## 4. Посмотреть количество мертвых строчек в таблице и когда последний раз приходил автовакуум
```
SELECT relname, n_live_tup, n_dead_tup,
trunc(100*n_dead_tup/(n_live_tup+1))::float "ratio%", last_autovacuum
FROM pg_stat_user_tables WHERE relname = 'table_1';
```
| relname | n\_live\_tup | n\_dead\_tup | ratio% | last\_autovacuum |
| :--- | :--- | :--- | :--- | :--- |
| table\_1 | 1000000 | 4999870 | 499 | 2025-03-15 11:01:52.802882 +00:00 |


## 5. Подождать некоторое время, проверяя, пришел ли автовакуум
```
SELECT relname, n_live_tup, n_dead_tup,
trunc(100*n_dead_tup/(n_live_tup+1))::float "ratio%", last_autovacuum
FROM pg_stat_user_tables WHERE relname = 'table_1';
```
| relname | n\_live\_tup | n\_dead\_tup | ratio% | last\_autovacuum |
| :--- | :--- | :--- | :--- | :--- |
| table\_1 | 999703 | 0 | 0 | 2025-03-15 11:02:57.318023 +00:00 |

Вот тут не сходится количество живых tuple с количеством строк в таблице. Данные видимо берет из статистики

## 6. 5 раз обновить все строчки и добавить к каждой строчке любой символ
```
UPDATE table_1 t set value = t.value || 'a';
UPDATE table_1 t set value = t.value || 'b';
UPDATE table_1 t set value = t.value || 'c';
UPDATE table_1 t set value = t.value || 'd';
UPDATE table_1 t set value = t.value || 'e';
```

## 7. Посмотреть размер файла с таблицей
```
SELECT pg_size_pretty(pg_total_relation_size('table_1'::regclass));
```
> 260 MB

## 8. Отключить Автовакуум на конкретной таблице
```
ALTER TABLE table_1 SET (AUTOVACUUM_ENABLED = OFF);
```

## 9. 10 раз обновить все строчки и добавить к каждой строчке любой символ
```
UPDATE table_1 t set value = t.value || 'a';
UPDATE table_1 t set value = t.value || 'b';
UPDATE table_1 t set value = t.value || 'c';
UPDATE table_1 t set value = t.value || 'd';
UPDATE table_1 t set value = t.value || 'e';
UPDATE table_1 t set value = t.value || 'a';
UPDATE table_1 t set value = t.value || 'b';
UPDATE table_1 t set value = t.value || 'c';
UPDATE table_1 t set value = t.value || 'd';
UPDATE table_1 t set value = t.value || 'e';
```

## 10. Посмотреть размер файла с таблицей
```
SELECT pg_size_pretty(pg_total_relation_size('table_1'::regclass));
```
> 569 MB


## 11. Объясните полученный результат
Автовакуум отключен, после десятикратного обновления каждой строки в таблице появилось 10 млн старых версий строк. Отсюда и размер 569 Мб.


## 12. Не забудьте включить автовакуум
```
ALTER TABLE table_1 SET (AUTOVACUUM_ENABLED = ON);
```
## 13. Написать анонимную процедуру, в которой в цикле 10 раз обновятся все строчки в искомой таблице. 
```
DO $$
    DECLARE i INT = 1;
    BEGIN
        FOR i IN 1..10
        LOOP
            UPDATE table_1 t set value = t.value || i::VARCHAR;
            RAISE NOTICE 'i = %', i;
            i = i + 1;
        END LOOP;
    END;
$$ LANGUAGE plpgsql;
```