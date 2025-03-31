1. На первом кластере создаем пользователя для репликации
```bash
psql -c "CREATE USER replicator WITH REPLICATION ENCRYPTED PASSWORD 'secret\$123';"
```
2. Разрешим репликацию в настройках
```
cat >> /etc/postgresql/17/main/pg_hba.conf << EOL
host replication replicator 127.0.0.1/32 scram-sha-256
EOL
```
3. Срздадим новый кластер для репликкации
```bash
pg_createcluster 17 replica --start
```
Теперь команда pg_lsclusters выводит
```
17  main    5432 online postgres /var/lib/postgresql/17/main    /var/log/postgresql/postgresql-17-main.log
17  replica 5434 online postgres /var/lib/postgresql/17/replica /var/log/postgresql/postgresql-17-replica.log
```
4. Удалим содержимое каталога реплики
```bash
rm -rf /var/lib/postgresql/17/replica/*
```
5. Создадим реплику (после ввода пароля дождемся чекпоинта или вызовем его из другого терминала)
```bash
pg_basebackup -h 127.0.0.1 -p 5432 -U replicator -R -S test -D /var/lib/postgresql/17/replica
```
6. Запустим реплику
```bash
pg_ctlcluster 17 main start
```
7. Убедимся, что на реплике появились данные
```bash
sudo -su postgres
psql -p 5434
\l
```
Увидим таблицу thai, с которой будем работать
8. Проверим режим реплицации
```bash
psql
SELECT application_name, sync_state FROM pg_stat_replication;
```
Увидим
```
application_name | sync_state 
------------------+------------
 17/replica       | async
```
9. Подготовим файлы для тестирования скорости
На запись
```bash
cat > ~/workload2.sql << EOL
INSERT INTO book.tickets (fkRide, fio, contact, fkSeat)
VALUES (
	ceil(random()*100)
	, (array(SELECT fam FROM book.fam))[ceil(random()*110)]::text || ' ' ||
    (array(SELECT nam FROM book.nam))[ceil(random()*110)]::text
    ,('{"phone":"+7' || (1000000000::bigint + floor(random()*9000000000)::bigint)::text || '"}')::jsonb
    , ceil(random()*100));

EOL
```
На чтение
```bash
cat > ~/workload.sql << EOL

\set r random(1, 5000000) 
SELECT id, fkRide, fio, contact, fkSeat FROM book.tickets WHERE id = :r;

EOL
```
10. Протестируем запись с включенной репликой
```bash
pgbench -c 8 -j 4 -T 10 -f ~/workload2.sql -n -U postgres -p 5432 thai
```
Получили tps = 2938.922490
11. Протестируем чтение с включенной репликой
```
pgbench -c 8 -j 4 -T 10 -f ~/workload.sql -n -U postgres thai
```
Получили tps = 21642.204511
12. Выключим репликацию
```
pg_ctlcluster 17 replica stop
```
13. Протестируем запись с выключенной репликой
```bash
pgbench -c 8 -j 4 -T 10 -f ~/workload2.sql -n -U postgres -p 5432 thai
```
Получили tps = 4906.165815
14. Протестируем чтение с выключенной репликой
```
pgbench -c 8 -j 4 -T 10 -f ~/workload.sql -n -U postgres thai
```
Получили tps = 20739.495091

Вывод:
При включенной реплике запись занимает больше времени
