1. Установил на сервер Postgres (по умолчанию на Debian-12 установился postgresql 15.10)
2. Загрузил тайские перевозки
```bash
wget https://storage.googleapis.com/thaibus/thai_small.tar.gz && tar -xf thai_small.tar.gz
sudo -u postgres psql < thai.sql
```
3. Запустил утилиту psql
```
sudo -u postgres psql
```
4. подключился к базе
```
\c thai
```
5. Посчитал количество поездок
```sql
select count(*) from book.tickets;
```

Ответ: 5185505
