### Block 1
1. ```sql SELECT * FROM bookings.flights WHERE departure_airport='UFA' AND actual_departure-scheduled_departure > INTERVAL '2 hours ; ```
* Сначала фильтруем по аэропорту вылета на уфу, потом добавляем филтрацию по разности времени запланированного и актуального вылета
2.  ```sql SELECT * FROM bookings.flights WHERE arrival_airport='SVO' AND actual_arrival >='2016-07-22 16:00' AND actual_arrival < '2016-07-22 19:00';```
* Фильтруем(c помощью where) по аэропорту SVO и времени больше 16:00 и меньше 19:00
3. ```sql SELECT * 
FROM bookings.flights 
WHERE aircraft_code = 'SU9' AND EXTRACT(HOUR FROM actual_departure) BETWEEN 4 AND 11; ```
* ФИльтруем по коду самолета и (SU9) и по вытаскиваем именно время, так как важно тольк время, но не дата, еще время считал именно актуального вылета, так как не сказано какого
4. ```sql SELECT t.*
FROM bookings.tickets t
JOIN bookings.bookings b ON t.book_ref = b.book_ref
WHERE t.passenger_name LIKE '%IVANOV%'
  AND b.book_date < '2017-09-13'; ```
* Исопльзуем джоин, чтобы брать данные из booking.tickets и booking.bookings, которые связаны book.ref, затем фильтруем по фамилии, точнее ищем все возможные ее вариации(мужская, женская, разные имена) и по дате
### Block 2
4. ```sql UPDATE bookings.flights SET status='Cancelled' WHERE departure_airport='SVO' AND status='Delayed';```
* Обновляем в столбце status на Cancelled, но ставим фильтр, что в status Delayed и аэропорт вылета SVO
### Block 3
4. ```sql SELECT DATE(scheduled_departure) as flight_date, COUNT(*) as delayed_flights_count FROM bookings.flights WHERE status = 'Delayed' AND EXTRACT(MONTH FROM scheduled_departure) = 8 GROUP BY DATE(scheduled_departure); ```
* flight_date псевдоним для даты(сразу обрезал время, оставив только дату для удобства), создаем столбец delayed_flights_count который считает количество строк в группе(группа создаеттся по каждому дню 8 месяца с помощью group by) с помощью агрегатной функции count. Фильтром where выбираем строки где значение в столбце status равно delayed и с помощью функции extract вытаскиваем из даты только месяц, который проверяем на равенство 8 
