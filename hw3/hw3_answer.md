1. Создал новую базу homework_3, подключился к ней
```
create database homework_3;
\c homework_3
```
2. Создал таблицу с 1 млн случайных данных
```sql
CREATE TABLE test(i text);
INSERT INTO test SELECT s.id FROM generate_series(1,1000000) AS s(id);
```
3. Посмотрел размер таблицы
```
\dt+ test
                                  Список отношений
 Схема  | Имя  |   Тип   | Владелец |  Хранение  | Метод доступа | Размер | Описание 
--------+------+---------+----------+------------+---------------+--------+----------
 public | test | таблица | postgres | постоянное | heap          | 35 MB  | 
```
```
SELECT
    n.nspname || '.' || c.relname AS table_name,
    pg_size_pretty(pg_total_relation_size(c.oid)) AS total_size,
    pg_size_pretty(pg_total_relation_size(c.reltoastrelid)) AS toast_size
FROM pg_class c
JOIN pg_namespace n
    ON c.relnamespace = n.oid
WHERE
    relname = 'test';

 table_name  | total_size | toast_size 
-------------+------------+------------
 public.test | 35 MB      | 8192 bytes
```
4. 5 раз обновитл все строчки и добавил к каждой строчке любой символ
```sql
UPDATE test SET i = i || 'a';
UPDATE test SET i = i || 'b';
UPDATE test SET i = i || 'c';
UPDATE test SET i = i || 'd';
UPDATE test SET i = i || 'e';
```
5. Посмотрел количество мертвых строк и когда последний раз приходил автовакуум
```sql
SELECT relname, n_live_tup, n_dead_tup, trunc(100*n_dead_tup/(n_live_tup+1))::float "ratio%", last_autovacuum FROM pg_stat_user_tables WHERE relname = 'test';

 relname | n_live_tup | n_dead_tup | ratio% |        last_autovacuum        
---------+------------+------------+--------+-------------------------------
 test    |    1000000 |    4999870 |    499 | 2025-03-13 12:14:45.082366+03
```
6. Дождался, когда пришел автовакуум
```
 relname | n_live_tup | n_dead_tup | ratio% |        last_autovacuum        
---------+------------+------------+--------+-------------------------------
 test    |    1615122 |          0 |      0 | 2025-03-13 12:17:45.498833+03
```
7. Добавил еще символов 5 раз
```sql
UPDATE test SET i = i || 'a';
UPDATE test SET i = i || 'b';
UPDATE test SET i = i || 'c';
UPDATE test SET i = i || 'd';
UPDATE test SET i = i || 'e';
```
7. Посмотрел размер файла с таблицей
```
\dt+ test
                                                                                                                                            
 Схема  | Имя  |   Тип   | Владелец |  Хранение  | Метод доступа | Размер | Описание 
--------+------+---------+----------+------------+---------------+--------+----------
 public | test | таблица | postgres | постоянное | heap          | 260 MB | 
```
8. 
8. Отключил автовакуум на конкретной таблице
```sql
ALTER TABLE test SET (autovacuum_enabled = off);
```
9. 10 раз обновил таблицу, добавив к каждой строчке символ
```sql
UPDATE test SET i = i || 'f';
UPDATE test SET i = i || 'g';
UPDATE test SET i = i || 'h';
UPDATE test SET i = i || 'i';
UPDATE test SET i = i || 'j';
UPDATE test SET i = i || 'k';
UPDATE test SET i = i || 'l';
UPDATE test SET i = i || 'm';
UPDATE test SET i = i || 'n';
UPDATE test SET i = i || 'o';
```
10. Посмотрел размер файла с таблицей
```
\dt+ test;

 Схема  | Имя  |   Тип   | Владелец |  Хранение  | Метод доступа | Размер | Описание 
--------+------+---------+----------+------------+---------------+--------+----------
 public | test | таблица | postgres | постоянное | heap          | 569 MB | 
```
