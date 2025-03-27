# Тестирование производительности pg_bouncer при разных режимах работы
Тестирование производилось на виртуальной машине VirtualBox с установленной Ubuntu 24.10
Скрипт workload.sql взял из лекции 
```
\set r random(1, 5000000) 
SELECT id, fkRide, fio, contact, fkSeat FROM book.tickets WHERE id = :r;
```
При смене режима работы выполнял команды 

```
sudo systemctl stop pgbouncer
sudo systemctl start pgbouncer
```

## 1. Производительность при подключении к ПГ напрямую без pg_bouncer
### 8 потоков 4 ядра 
>pgbench -c 8 -j 4 -T 10 -f workload.sql -n -U postgres thai
tps = 29421.793824

### 100 потоков 4 ядра 
> pgbench -c 100 -j 4 -T 10 -f workload.sql -n -U postgres thai
sorry, too many clients already

## 2. Производительность при подключении к ПГ через pg_bouncer pool_mode = session
### 150 потоков 4 ядра
> pgbench -c 150 -j 4 -T 10 -f workload.sql -n -U postgres -p 6432 -h 127.0.0.1 thai
number of transactions actually processed: 96059
number of failed transactions: 0 (0.000%)
latency average = 14.808 ms
initial connection time = 578.009 ms
tps = 10129.536247 (without initial connection time)

## 3. Производительность при подключении к ПГ через pg_bouncer pool_mode = transaction
### 150 потоков 4 ядра
> pgbench -c 150 -j 4 -T 10 -f workload.sql -n -U postgres -p 6432 -h 127.0.0.1 thai
number of transactions actually processed: 86588
number of failed transactions: 0 (0.000%)
latency average = 16.412 ms
initial connection time = 546.283 ms
tps = 9139.513746 (without initial connection time)

## 4. Производительность при подключении к ПГ через pg_bouncer pool_mode = statement
### 150 потоков 4 ядра
> pgbench -c 150 -j 4 -T 10 -f workload.sql -n -U postgres -p 6432 -h 127.0.0.1 thai
tps = 9073.412472
number of transactions actually processed: 89502
number of failed transactions: 0 (0.000%)
latency average = 15.868 ms
initial connection time = 554.289 ms
tps = 9452.756758 (without initial connection time)

# Особой разницы между режимами не заметил. Возможно не были соблюдены все необходимые условия.