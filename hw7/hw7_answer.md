1. Подключимся к базе тайских перевозок (small)
```
\c thai
```
2. Включим таймер
```
\timing
```
3. Сделаем сложный запрос и посмотрим время исполнения
```sql
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
Получили 1357,270 мс
4. Проиндексируем таблицу
```sql
-- Для all_place CTE (группировка по fkbus)
CREATE INDEX IF NOT EXISTS idx_seat_fkbus ON book.seat(fkbus);

-- Для order_place CTE (группировка по fkride)
CREATE INDEX IF NOT EXISTS idx_tickets_fkride ON book.tickets(fkride);

-- Для соединения ride с schedule
CREATE INDEX IF NOT EXISTS idx_ride_fkschedule ON book.ride(fkschedule);

-- Для соединения schedule с busroute
CREATE INDEX IF NOT EXISTS idx_schedule_fkroute ON book.schedule(fkroute);

-- Для соединения busroute с busstation
CREATE INDEX IF NOT EXISTS idx_busroute_fkbusstationfrom ON book.busroute(fkbusstationfrom);

-- Для соединения ride с all_place (по fkbus)
CREATE INDEX IF NOT EXISTS idx_ride_fkbus ON book.ride(fkbus);
```
5. Повторим запрос и сравним время
Запрос показал такое же время, EXPLAIN ANALYZE показал, что индексы не используются.

Вывод:
Индексы не меняют время запросы. Вероятно, при малом размере таблицы psql выбирает полное сканирование.
