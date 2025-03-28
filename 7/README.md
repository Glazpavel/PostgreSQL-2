# Тестирование индексов на внешние ключи
Сам запрос
```
WITH all_place AS (
    SELECT count(s.id) as all_place, s.fkbus as fkbus
    FROM book.seat s
    group by s.fkbus
),
order_place AS (
    SELECT count(t.id) as order_place, t.fkride
    FROM book.tickets t
    group by t.fkride
)
SELECT r.id, r.startdate as depart_date, bs.city || ', ' || bs.name as busstation,
      t.order_place, st.all_place
FROM book.ride r
JOIN book.schedule as s
      on r.fkschedule = s.id
JOIN book.busroute br
      on s.fkroute = br.id
JOIN book.busstation bs
      on br.fkbusstationfrom = bs.id
JOIN order_place t
      on t.fkride = r.id
JOIN all_place st
      on r.fkbus = st.fkbus
GROUP BY r.id, r.startdate, bs.city || ', ' || bs.name, t.order_place,st.all_place
ORDER BY r.startdate
limit 10;
```
## 1. План при выполнении без индексов
```
Limit  (cost=329833.80..329833.82 rows=10 width=56) (actual time=862.085..871.188 rows=10 loops=1)
  ->  Sort  (cost=329833.80..330187.94 rows=141656 width=56) (actual time=848.253..857.355 rows=10 loops=1)
        Sort Key: r.startdate
        Sort Method: top-N heapsort  Memory: 25kB
        ->  Group  (cost=277308.49..326772.66 rows=141656 width=56) (actual time=652.788..844.632 rows=144000 loops=1)
"              Group Key: r.id, (((bs.city || ', '::text) || bs.name)), (count(t.id)), (count(s_1.id))"
              ->  Incremental Sort  (cost=277308.49..324647.82 rows=141656 width=56) (actual time=652.785..824.943 rows=144000 loops=1)
"                    Sort Key: r.id, (((bs.city || ', '::text) || bs.name)), (count(t.id)), (count(s_1.id))"
                    Presorted Key: r.id
                    Full-sort Groups: 4500  Sort Method: quicksort  Average Memory: 27kB  Peak Memory: 27kB
                    ->  Merge Join  (cost=277079.53..316167.00 rows=141656 width=56) (actual time=652.732..804.333 rows=144000 loops=1)
                          Merge Cond: (r.id = t.fkride)
                          ->  Sort  (cost=20076.71..20436.71 rows=144000 width=32) (actual time=87.600..97.614 rows=144000 loops=1)
                                Sort Key: r.id
                                Sort Method: external merge  Disk: 6488kB
                                ->  Hash Join  (cost=52.09..4291.50 rows=144000 width=32) (actual time=0.278..64.447 rows=144000 loops=1)
                                      Hash Cond: (r.fkbus = s_1.fkbus)
                                      ->  Hash Join  (cost=46.98..3587.99 rows=144000 width=28) (actual time=0.215..50.905 rows=144000 loops=1)
                                            Hash Cond: (br.fkbusstationfrom = bs.id)
                                            ->  Hash Join  (cost=45.75..3048.56 rows=144000 width=16) (actual time=0.202..36.639 rows=144000 loops=1)
                                                  Hash Cond: (s.fkroute = br.id)
                                                  ->  Hash Join  (cost=43.40..2641.51 rows=144000 width=16) (actual time=0.184..22.680 rows=144000 loops=1)
                                                        Hash Cond: (r.fkschedule = s.id)
                                                        ->  Seq Scan on ride r  (cost=0.00..2219.00 rows=144000 width=16) (actual time=0.003..5.668 rows=144000 loops=1)
                                                        ->  Hash  (cost=25.40..25.40 rows=1440 width=8) (actual time=0.175..0.176 rows=1440 loops=1)
                                                              Buckets: 2048  Batches: 1  Memory Usage: 73kB
                                                              ->  Seq Scan on schedule s  (cost=0.00..25.40 rows=1440 width=8) (actual time=0.004..0.078 rows=1440 loops=1)
                                                  ->  Hash  (cost=1.60..1.60 rows=60 width=8) (actual time=0.013..0.013 rows=60 loops=1)
                                                        Buckets: 1024  Batches: 1  Memory Usage: 11kB
                                                        ->  Seq Scan on busroute br  (cost=0.00..1.60 rows=60 width=8) (actual time=0.004..0.007 rows=60 loops=1)
                                            ->  Hash  (cost=1.10..1.10 rows=10 width=20) (actual time=0.008..0.008 rows=10 loops=1)
                                                  Buckets: 1024  Batches: 1  Memory Usage: 9kB
                                                  ->  Seq Scan on busstation bs  (cost=0.00..1.10 rows=10 width=20) (actual time=0.005..0.005 rows=10 loops=1)
                                      ->  Hash  (cost=5.05..5.05 rows=5 width=12) (actual time=0.055..0.055 rows=5 loops=1)
                                            Buckets: 1024  Batches: 1  Memory Usage: 9kB
                                            ->  HashAggregate  (cost=5.00..5.05 rows=5 width=12) (actual time=0.051..0.052 rows=5 loops=1)
                                                  Group Key: s_1.fkbus
                                                  Batches: 1  Memory Usage: 24kB
                                                  ->  Seq Scan on seat s_1  (cost=0.00..4.00 rows=200 width=8) (actual time=0.012..0.021 rows=200 loops=1)
                          ->  Finalize GroupAggregate  (cost=257002.82..292891.31 rows=141656 width=12) (actual time=565.101..673.576 rows=144000 loops=1)
                                Group Key: t.fkride
                                ->  Gather Merge  (cost=257002.82..290058.19 rows=283312 width=12) (actual time=565.070..639.237 rows=432000 loops=1)
                                      Workers Planned: 2
                                      Workers Launched: 2
                                      ->  Sort  (cost=256002.79..256356.93 rows=141656 width=12) (actual time=549.781..559.543 rows=144000 loops=3)
                                            Sort Key: t.fkride
                                            Sort Method: external merge  Disk: 3672kB
                                            Worker 0:  Sort Method: external merge  Disk: 3672kB
                                            Worker 1:  Sort Method: external merge  Disk: 3672kB
                                            ->  Partial HashAggregate  (cost=218944.62..241460.68 rows=141656 width=12) (actual time=420.679..515.624 rows=144000 loops=3)
                                                  Group Key: t.fkride
                                                  Planned Partitions: 4  Batches: 5  Memory Usage: 8241kB  Disk Usage: 24104kB
                                                  Worker 0:  Batches: 5  Memory Usage: 8241kB  Disk Usage: 27648kB
                                                  Worker 1:  Batches: 5  Memory Usage: 8241kB  Disk Usage: 24056kB
                                                  ->  Parallel Seq Scan on tickets t  (cost=0.00..80531.89 rows=2160589 width=12) (actual time=0.034..111.380 rows=1728502 loops=3)
Planning Time: 0.615 ms
JIT:
  Functions: 81
"  Options: Inlining false, Optimization false, Expressions true, Deforming true"
"  Timing: Generation 2.197 ms, Inlining 0.000 ms, Optimization 1.561 ms, Emission 25.410 ms, Total 29.168 ms"
Execution Time: 882.042 ms

## 2. Создал индексы на внешние ключи
```
CREATE INDEX CONCURRENTLY IF NOT EXISTS ix_ride_fkschedule ON book.ride (fkschedule);
VACUUM ANALYZE book.ride;

CREATE INDEX CONCURRENTLY IF NOT EXISTS  ix_schedule_fkroute ON book.schedule (fkroute);
VACUUM ANALYZE book.schedule;

CREATE INDEX CONCURRENTLY IF NOT EXISTS ix_busroute_fkbusstationfrom ON book.busroute (fkbusstationfrom);
VACUUM ANALYZE book.busroute;
--Следующие два явно использоваться не будут, но ради эксперимента сделаем
CREATE INDEX CONCURRENTLY IF NOT EXISTS ix_tickets_fkride ON book.tickets (fkride);
VACUUM ANALYZE book.tickets;

CREATE INDEX CONCURRENTLY IF NOT EXISTS ix_seat_fkbus ON book.seat (fkbus);
VACUUM ANALYZE book.seat;
```
## 3. План при выполнении с индексами
```
Limit  (cost=330421.26..330421.29 rows=10 width=56) (actual time=808.075..816.559 rows=10 loops=1)
  ->  Sort  (cost=330421.26..330778.33 rows=142829 width=56) (actual time=795.583..804.066 rows=10 loops=1)
        Sort Key: r.startdate
        Sort Method: top-N heapsort  Memory: 25kB
        ->  Group  (cost=277458.53..327334.78 rows=142829 width=56) (actual time=620.346..791.968 rows=144000 loops=1)
"              Group Key: r.id, (((bs.city || ', '::text) || bs.name)), (count(t.id)), (count(s_1.id))"
              ->  Incremental Sort  (cost=277458.53..325192.34 rows=142829 width=56) (actual time=620.343..775.279 rows=144000 loops=1)
"                    Sort Key: r.id, (((bs.city || ', '::text) || bs.name)), (count(t.id)), (count(s_1.id))"
                    Presorted Key: r.id
                    Full-sort Groups: 4500  Sort Method: quicksort  Average Memory: 27kB  Peak Memory: 27kB
                    ->  Merge Join  (cost=277227.65..316632.83 rows=142829 width=56) (actual time=620.286..757.425 rows=144000 loops=1)
                          Merge Cond: (r.id = t.fkride)
                          ->  Sort  (cost=20076.71..20436.71 rows=144000 width=32) (actual time=83.698..92.514 rows=144000 loops=1)
                                Sort Key: r.id
                                Sort Method: external merge  Disk: 6488kB
                                ->  Hash Join  (cost=52.09..4291.50 rows=144000 width=32) (actual time=0.275..64.095 rows=144000 loops=1)
                                      Hash Cond: (r.fkbus = s_1.fkbus)
                                      ->  Hash Join  (cost=46.98..3587.99 rows=144000 width=28) (actual time=0.215..50.566 rows=144000 loops=1)
                                            Hash Cond: (br.fkbusstationfrom = bs.id)
                                            ->  Hash Join  (cost=45.75..3048.56 rows=144000 width=16) (actual time=0.202..36.547 rows=144000 loops=1)
                                                  Hash Cond: (s.fkroute = br.id)
                                                  ->  Hash Join  (cost=43.40..2641.51 rows=144000 width=16) (actual time=0.183..22.491 rows=144000 loops=1)
                                                        Hash Cond: (r.fkschedule = s.id)
                                                        ->  Seq Scan on ride r  (cost=0.00..2219.00 rows=144000 width=16) (actual time=0.002..5.716 rows=144000 loops=1)
                                                        ->  Hash  (cost=25.40..25.40 rows=1440 width=8) (actual time=0.175..0.176 rows=1440 loops=1)
                                                              Buckets: 2048  Batches: 1  Memory Usage: 73kB
                                                              ->  Seq Scan on schedule s  (cost=0.00..25.40 rows=1440 width=8) (actual time=0.004..0.079 rows=1440 loops=1)
                                                  ->  Hash  (cost=1.60..1.60 rows=60 width=8) (actual time=0.013..0.013 rows=60 loops=1)
                                                        Buckets: 1024  Batches: 1  Memory Usage: 11kB
                                                        ->  Seq Scan on busroute br  (cost=0.00..1.60 rows=60 width=8) (actual time=0.004..0.007 rows=60 loops=1)
                                            ->  Hash  (cost=1.10..1.10 rows=10 width=20) (actual time=0.008..0.008 rows=10 loops=1)
                                                  Buckets: 1024  Batches: 1  Memory Usage: 9kB
                                                  ->  Seq Scan on busstation bs  (cost=0.00..1.10 rows=10 width=20) (actual time=0.004..0.005 rows=10 loops=1)
                                      ->  Hash  (cost=5.05..5.05 rows=5 width=12) (actual time=0.053..0.053 rows=5 loops=1)
                                            Buckets: 1024  Batches: 1  Memory Usage: 9kB
                                            ->  HashAggregate  (cost=5.00..5.05 rows=5 width=12) (actual time=0.049..0.050 rows=5 loops=1)
                                                  Group Key: s_1.fkbus
                                                  Batches: 1  Memory Usage: 24kB
                                                  ->  Seq Scan on seat s_1  (cost=0.00..4.00 rows=200 width=8) (actual time=0.012..0.020 rows=200 loops=1)
                          ->  Finalize GroupAggregate  (cost=257150.94..293336.62 rows=142829 width=12) (actual time=536.554..634.721 rows=144000 loops=1)
                                Group Key: t.fkride
                                ->  Gather Merge  (cost=257150.94..290480.04 rows=285658 width=12) (actual time=536.517..603.502 rows=432000 loops=1)
                                      Workers Planned: 2
                                      Workers Launched: 2
                                      ->  Sort  (cost=256150.92..256507.99 rows=142829 width=12) (actual time=513.046..522.043 rows=144000 loops=3)
                                            Sort Key: t.fkride
                                            Sort Method: external merge  Disk: 3672kB
                                            Worker 0:  Sort Method: external merge  Disk: 3672kB
                                            Worker 1:  Sort Method: external merge  Disk: 3672kB
                                            ->  Partial HashAggregate  (cost=218950.40..241478.95 rows=142829 width=12) (actual time=388.772..484.792 rows=144000 loops=3)
                                                  Group Key: t.fkride
                                                  Planned Partitions: 4  Batches: 5  Memory Usage: 8241kB  Disk Usage: 27536kB
                                                  Worker 0:  Batches: 5  Memory Usage: 8241kB  Disk Usage: 24088kB
                                                  Worker 1:  Batches: 5  Memory Usage: 8241kB  Disk Usage: 27408kB
                                                  ->  Parallel Seq Scan on tickets t  (cost=0.00..80532.67 rows=2160667 width=12) (actual time=0.042..105.640 rows=1728502 loops=3)
Planning Time: 0.706 ms
JIT:
  Functions: 81
"  Options: Inlining false, Optimization false, Expressions true, Deforming true"
"  Timing: Generation 2.257 ms, Inlining 0.000 ms, Optimization 1.450 ms, Emission 24.855 ms, Total 28.562 ms"
Execution Time: 824.901 ms
```
# Вывод. В данном конкретном примере индексы на внешних ключах не используются.