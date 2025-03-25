# Тестирование быстродействия pgbounser
1. Создадим файл для тестирования нагрузки (SELECT случайной строки)
```bash
cat > ~/workload.sql << EOL
\set r random(1, 5000000) 
SELECT id, fkRide, fio, contact, fkSeat FROM book.tickets WHERE id = :r;
EOL
```
2. Сменим пользователя на postgres с root правами
```bash
sudo -su postgres
```
3. Протестируем производительность c 8 пользователями
```bash
pgbench -c 8 -j 4 -T 10 -f ./workload.sql -n -U postgres thai
```
```
pgbench (17.4 (Debian 17.4-1.pgdg120+2))
transaction type: ./workload.sql
scaling factor: 1
query mode: simple
number of clients: 8
number of threads: 4
maximum number of tries: 1
duration: 10 s
number of transactions actually processed: 215545
number of failed transactions: 0 (0.000%)
latency average = 0.371 ms
initial connection time = 13.317 ms
tps = 21570.839911 (without initial connection time)
```
получили 21.5к танзакций в секунду
4. Протестируем производительность cо 100 пользователями
```bash
pgbench -c 100 -j 4 -T 10 -f ./workload.sql -n -U postgres thai
```
```
pgbench (17.4 (Debian 17.4-1.pgdg120+2))
transaction type: ./workload.sql
scaling factor: 1
query mode: simple
number of clients: 100
number of threads: 4
maximum number of tries: 1
duration: 10 s
number of transactions actually processed: 171589
number of failed transactions: 0 (0.000%)
latency average = 5.725 ms
initial connection time = 255.321 ms
tps = 17468.454997 (without initial connection time)
```
Получили 17.5к транзакций в секунду (меньше, потому что часть коннектов ждет своей очереди)
5. Установим pgbouncer
```bash
sudo apt install -y pgbouncer
```
6. Убедимся, что pgbouncer установлен и запущен
```bash
sudo systemctl status pgbouncer
```
```
Active: active (running) since Tue 2025-03-25 17:16:49 MSK; 4min 17s ago
```
7. Остановим pgbouncer, изменим его настройки
```bash
sudo systemctl stop pgbouncer

cat > temp.cfg << EOF 
[databases]
thai = host=127.0.0.1 port=5432 dbname=thai
[pgbouncer]
logfile = /var/log/postgresql/pgbouncer.log
pidfile = /var/run/postgresql/pgbouncer.pid
listen_addr = *
listen_port = 6432
auth_type = scram-sha-256
auth_file = /etc/pgbouncer/userlist.txt
admin_users = admindb
EOF
cat temp.cfg | sudo tee -a /etc/pgbouncer/pgbouncer.ini
```
8. Опишем пользоветелей и запустим pgbouncer
```bash
cat > temp2.cfg << EOF 
"admindb" "admin123#"
"postgres" "admin123#"
EOF
cat temp2.cfg | sudo tee -a /etc/pgbouncer/userlist.txt

sudo systemctl start pgbouncer
```
9. Поменяем пароль пользователю postgres, чтобы он соответствовал указанному в pgbouncer
```bash
sudo -u postgres psql -c "ALTER USER postgres WITH PASSWORD 'admin123#';";
```
10. Добавим пользователя в pgpass, чтобы заходить без пароля
```bash
echo "localhost:5432:thai:postgres:admin123#" | sudo tee -a /var/lib/postgresql/.pgpass && sudo chmod 600 /var/lib/postgresql/.pgpass 
sudo chown postgres:postgres /var/lib/postgresql/.pgpass
```
11. Проверим, что pgbouncer работает на порте 6432
```bash
psql -p 6432 -h 127.0.0.1 -d thai -U postgres
```
(вводим пароль и подключаемся к базе thai)
12. Протестируем производительность для 8 пользователей через Linux socket
```bash
pgbench -c 8 -j 4 -T 10 -f ./workload.sql -n -U postgres thai
```
```
pgbench (17.4 (Debian 17.4-1.pgdg120+2))
transaction type: ./workload.sql
scaling factor: 1
query mode: simple
number of clients: 8
number of threads: 4
maximum number of tries: 1
duration: 10 s
number of transactions actually processed: 212859
number of failed transactions: 0 (0.000%)
latency average = 0.375 ms
initial connection time = 16.748 ms
tps = 21313.831777 (without initial connection time)
```
Получили 21.3к транзакций в секунду
13. Протестируем производительность для 8 пользователей через TCP
```bash
pgbench -c 8 -j 4 -T 10 -f ./workload.sql -n -U postgres -p 5432 -h localhost thai
```
```

```
Получили 13.7 транзакций в секунду (меньше), потому что получали доступ по сети
14. Протестируем производительность для 8 пользователей через pgbouncer
```bash
pgbench -c 8 -j 4 -T 10 -f ./workload.sql -n -U postgres -p 6432 -h 127.0.0.1 thai
```
```
pgbench (17.4 (Debian 17.4-1.pgdg120+2))
transaction type: ./workload.sql
scaling factor: 1
query mode: simple
number of clients: 8
number of threads: 4
maximum number of tries: 1
duration: 10 s
number of transactions actually processed: 114870
number of failed transactions: 0 (0.000%)
latency average = 0.692 ms
initial connection time = 69.693 ms
tps = 11564.947748 (without initial connection time)
```
Получили 11.5к транзакций в секунду (еще меньше), потому что pgbouncer потребляет дополнительные ресурсы системы
15. Протестируем производительность для 125 пользователей через TCP
```bash
pgbench -c 125 -j 4 -T 10 -f ./workload.sql -n -U postgres -p 5432 -h localhost thai
```
```
pgbench: error: connection to server at "localhost" (::1), port 5432 failed: FATAL:  sorry, too many clients already
connection to server at "localhost" (::1), port 5432 failed: FATAL:  sorry, too many clients already
pgbench: error: could not create connection for client 55
```
Получаем ошибку (превышение количества пользователей)
16. Остановим pgbouncer, изменим настройки, запустим pgbouncer
```
[pgbouncer]
pool_mode = session
max_client_conn = 1000
```
17. Проверим производительность с pool_mode = session
```bash
pgbench -c 200 -j 4 -T 10 -f ./workload.sql -n -U postgres -p 6432 -h localhost thai
```
```
pgbench (17.4 (Debian 17.4-1.pgdg120+2))
transaction type: ./workload.sql
scaling factor: 1
query mode: simple
number of clients: 200
number of threads: 4
maximum number of tries: 1
duration: 10 s
number of transactions actually processed: 87568
number of failed transactions: 0 (0.000%)
latency average = 18.927 ms
initial connection time = 1760.464 ms
tps = 10566.965586 (without initial connection time)
```
Получили 10.5к транзакций в секунду
18. Остановим pgbouncer, изменим настройки, запустим pgbouncer
```
[pgbouncer]
pool_mode = transaction
```
19. Проверим производительность с pool_mode = transaction
```bash
pgbench -c 200 -j 4 -T 10 -f ./workload.sql -n -U postgres -p 6432 -h localhost thai
```
```
pgbench (17.4 (Debian 17.4-1.pgdg120+2))
transaction type: ./workload.sql
scaling factor: 1
query mode: simple
number of clients: 200
number of threads: 4
maximum number of tries: 1
duration: 10 s
number of transactions actually processed: 81152
number of failed transactions: 0 (0.000%)
latency average = 20.053 ms
initial connection time = 1898.141 ms
tps = 9973.415940 (without initial connection time)
```
Получили 9.9к транзакций в секунду
20. Остановим pgbouncer, изменим настройки, запустим pgbouncer
```
[pgbouncer]
pool_mode = statement
```
21. Проверим производительность с pool_mode = statement
```bash
pgbench -c 200 -j 4 -T 10 -f ./workload.sql -n -U postgres -p 6432 -h localhost thai
```
```
pgbench (17.4 (Debian 17.4-1.pgdg120+2))
transaction type: ./workload.sql
scaling factor: 1
query mode: simple
number of clients: 200
number of threads: 4
maximum number of tries: 1
duration: 10 s
number of transactions actually processed: 79174
number of failed transactions: 0 (0.000%)
latency average = 20.631 ms
initial connection time = 1867.057 ms
tps = 9694.326406 (without initial connection time)
```
Получили 9.6к транзакций в секунду

Итог:

|pool_mode | Как работает | tps |
|-|-|-|
| session | Клиент получает соединение из пула и удерживает его до тех пор, пока не завершит работу (даже между транзакциями) | 10.5k | 
| transaction | Клиент получает соединение только на время транзакции | 9.9k | 
| statement | Соединение привязывается к клиенту только на время выполнения одного SQL-запроса | 9.6k | 

В нашем случае лучшим оказлся тим session, однако к его минусам можно отнести возможномть исчерпания пула запросов. Такой тип соединений рекомендуют для длинных транзакций.
