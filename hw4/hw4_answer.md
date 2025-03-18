1. Создал новую базу homework_4, подключился к ней.
```
create database homework_4;
\c homework_4
```
2. Зашел со второго терминала, подключился к той же базе.
3. В первом терминале создал новую таблицу. Заполнил ее данными.
```sql
CREATE TABLE accounts(id integer, amount numeric);
INSERT INTO accounts VALUES (1,2000.00), (2,2000.00), (3,2000.00);
```
4. В первом терминале начал транзакцию, внес изменение в строку c id=1
```sql
BEGIN;
UPDATE accounts SET amount = amount + 1 WHERE id = 1;
```
5. Во втором терминале начал транзакцию, внес изменение в строку с id=2
```sql
BEGIN;
UPDATE accounts SET amount = amount + 1 WHERE id = 2;
```
6. Пока что ничего не произошло, попробуем получить из первого терминала доступ к той же строке, что сейчас изменяет второй терминал
```sql
UPDATE accounts SET amount = amount + 1 WHERE id = 2;
```
В первоем терминале получили зависшую транзакцию, которая ожидает завершения транзакции, которая была начата во втором терминале. Как только транзакция во втором терминале завершится, первая транзакция получит доступ к строке с id=2

7. Попытаемся записать из второго терминала в строку с id=1
```sql
UPDATE accounts SET amount = amount + 1 WHERE id = 1;
```
Во втором терминале получили ошибку.
```
ERROR:  deadlock detected
ПОДРОБНОСТИ:  Process 198124 waits for ShareLock on transaction 1338; blocked by process 198085.
Process 198085 waits for ShareLock on transaction 1339; blocked by process 198124.
ПОДСКАЗКА:  See server log for query details.
КОНТЕКСТ:  while updating tuple (0,1) in relation "accounts"
```
В первом терминале запись прошла успешно

8. В итоге в базе наблюдаем 
```
 id | amount  
----+---------
  3 | 2000.00
  1 | 2001.00
  2 | 2001.00
```

9. Посмотрим логи
```bash
\! tail -n 50 /var/log/postgresql/postgresql-17-main.log
```
В них есть информация о deadlock
```
2025-03-18 17:45:19.997 MSK [198124] postgres@homework_4 ERROR:  deadlock detected
2025-03-18 17:45:19.997 MSK [198124] postgres@homework_4 DETAIL:  Process 198124 waits for ShareLock on transaction 1338; blocked by process 198085.
	Process 198085 waits for ShareLock on transaction 1339; blocked by process 198124.
	Process 198124: UPDATE accounts SET amount = amount + 1 WHERE id = 1;
	Process 198085: UPDATE accounts SET amount = amount + 1 WHERE id = 2;
2025-03-18 17:45:19.997 MSK [198124] postgres@homework_4 HINT:  See server log for query details.
2025-03-18 17:45:19.997 MSK [198124] postgres@homework_4 CONTEXT:  while updating tuple (0,1) in relation "accounts"
2025-03-18 17:45:19.997 MSK [198124] postgres@homework_4 STATEMENT:  UPDATE accounts SET amount = amount + 1 WHERE id = 1;
```

