## 1. Создать таблицу accounts(id integer, amount numeric);
```
DROP TABLE IF EXISTS accounts;
CREATE TABLE accounts
(
    id integer PRIMARY KEY,
    amount numeric(10, 2)
);
INSERT INTO accounts (id, amount) 
VALUES (1, 2000.00), (2, 2000.00), (3, 3000.00);
```


##2. Добавить несколько записей и подключившись через 2 терминала добиться ситуации взаимоблокировки (deadlock).

1-я консоль
```
BEGIN TRANSACTION ;
UPDATE accounts SET amount = amount + 1 WHERE id = 1;
```

2-я консоль
```
BEGIN TRANSACTION ;
UPDATE accounts SET amount = amount + 10 WHERE id = 2;
```

1-я консоль
```
UPDATE accounts SET amount = amount + 1 WHERE id = 2;
```

2-я консоль
```
UPDATE accounts SET amount = amount + 10 WHERE id = 1;
```

##3. Посмотреть логи и убедиться, что информация о дедлоке туда попала.
```
2025-03-17 16:29:53 2025-03-17 13:29:53.464 UTC [45] ERROR:  deadlock detected
2025-03-17 16:29:53 2025-03-17 13:29:53.464 UTC [45] DETAIL:  Process 45 waits for ShareLock on transaction 1021; blocked by process 37.
2025-03-17 16:29:53     Process 37 waits for ShareLock on transaction 1022; blocked by process 45.
2025-03-17 16:29:53     Process 45: UPDATE accounts SET amount = amount + 10 WHERE id = 1
2025-03-17 16:29:53     Process 37: UPDATE accounts SET amount = amount + 1 WHERE id = 2
2025-03-17 16:29:53 2025-03-17 13:29:53.464 UTC [45] HINT:  See server log for query details.
2025-03-17 16:29:53 2025-03-17 13:29:53.464 UTC [45] CONTEXT:  while updating tuple (0,1) in relation "accounts"
2025-03-17 16:29:53 2025-03-17 13:29:53.464 UTC [45] STATEMENT:  UPDATE accounts SET amount = amount + 10 WHERE id = 1
2025-03-17 16:29:53 2025-03-17 13:29:53.468 UTC [45] ERROR:  current transaction is aborted, commands ignored until end of transaction block
```

Транзакция во 2 консоли была выбрана жертвой и отменена.
Транзакция в 1 консоли благополучно завершилась
SELECT *
FROM accounts a;
| id | amount |
| :--- | :--- |
| 3 | 3000.00 |
| 1 | 2001.00 |
| 2 | 2001.00 |
