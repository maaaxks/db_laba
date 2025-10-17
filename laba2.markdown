### Variant 4
'WITH passenger_journeys AS (
    SELECT tf.ticket_no, t.passenger_name,
        f.departure_airport,
        f.arrival_airport,
        f.scheduled_departure,
        f.scheduled_arrival,
        LEAD(f.departure_airport) OVER (PARTITION BY tf.ticket_no ORDER BY f.scheduled_departure) as next_departure_airport,
        LEAD(f.scheduled_departure) OVER (PARTITION BY tf.ticket_no ORDER BY f.scheduled_departure) as next_departure_time
    FROM bookings.ticket_flights tf
    JOIN bookings.flights f ON tf.flight_id = f.flight_id
    JOIN bookings.tickets t ON tf.ticket_no = t.ticket_no
),
transit_passengers AS (
    SELECT
        arrival_airport as transit_airport,
        COUNT(*) as transit_passengers_count,
        AVG(next_departure_time - scheduled_arrival) as avg_connection_time
    FROM passenger_journeys
    WHERE next_departure_airport IS NOT NULL
      AND arrival_airport = next_departure_airport
    GROUP BY arrival_airport
),
total_passengers AS (
    SELECT
        departure_airport as airport,
        COUNT(*) as total_passengers
    FROM bookings.ticket_flights tf
    JOIN bookings.flights f ON tf.flight_id = f.flight_id
    GROUP BY departure_airport
)
SELECT
    a.airport_code,
    a.airport_name,
    COALESCE(tp.transit_passengers_count, 0) as transit_passengers,
    tp2.total_passengers as total_passengers, 
    ROUND(
        COALESCE(tp.transit_passengers_count::decimal / NULLIF(tp2.total_passengers, 0) * 100, 0),
        2
    ) as transit_percentage,
    COALESCE(EXTRACT(EPOCH FROM tp.avg_connection_time)/3600, 0) as avg_connection_hours
FROM bookings.airports_data a
LEFT JOIN transit_passengers tp ON a.airport_code = tp.transit_airport
LEFT JOIN total_passengers tp2 ON a.airport_code = tp2.airport 
WHERE COALESCE(tp.transit_passengers_count, 0) > 0
ORDER BY transit_passengers DESC
LIMIT 20;'
* Если очень коротко, то сначала составляем маршруты пассажиров, ищем ессть ли следующие аэропорты в пути, по этой информации считаем количество тразитных пассажиров(тех у кого есть следующие остановки), обычных пассажиров и среднее время стыковки, затем выводим.
* Если же подробней, то: 
* использую with .... as, чтобы создать временные результаты, временными результатми иемю ввиду здесь passenger_journeys, transit_passengers, total_passengers. Последние по названию понятно вычисляются количество тразитных и всех пассажиров для последующего соотношения и подсчета, их я потом тоже вывожу для простоты. passenger_journeys же это маршрут пассажиров, он нужен чтобы смотреть на следующий аэропорт в пути пассажира(next_departure_airport, если он конечно есть).
* теперь по каждому отдельно
# passenger_journeys
* нам нужны данные столбцы: аэропорты вылета/прилета и время прилета/вылета для расчета времени стыковки 
'tf.ticket_no, t.passenger_name,
        f.departure_airport,
        f.arrival_airport,
        f.scheduled_departure,
        f.scheduled_arrival'
* их вытаскиваем с помощью join нескольких таблиц, делаем это так
    'FROM bookings.ticket_flights tf
    JOIN bookings.flights f ON tf.flight_id = f.flight_id
    JOIN bookings.tickets t ON tf.ticket_no = t.ticket_no'
* далее используем lead для определения следующего аэропорта и времени следующего вылета(после текущего), так как нужны именно они, чтобы понимать является данная точка тразитной или нет, используем тут еще partition для разделения данных(каждого пассажира отдельно) и order by для упорядочивания по времени вылета. Этим (следующий аэропорт в пути пассажира и время слледующего вылета) даем псевдонимы next_departure_airport, next_departure_time соответственно.
# transit_passengers
* считаем количество тразитных пассажиров по каждому аэропорту. Для этого сначала фильтруем where, чтобы следующий аэропорт существовал (не null) и чтобы аэропорт прилета и вылета совпадали(пересел он там значит), затем группируем по аэропортам прилета и затем для каждой этой группы с помощью count считаем количество строк в группе(это и есть количество транзитных пассажиров для данного аэропорта). Еще высчитываем тут же среднее время стыковки с помощью функции avg от разности времени прилета и вылета
# total_passengers
* почти аналогично предыдущему блоку считаем количество всех пассажиров, только не используем филтров, потому что нужны любые люди и также из-за этого выбираем не из passengers_journeys, а из начальных таблиц, тут мы тоже кстати исопльзуем join, так как в таблице с перелетами нет айдишников пассажиров, что не дает их посчитать
# Финальный select
*  используем airports_data для вывода финального результата. Оттуда берем код самолета (airport_code), (я видел, что стоит изегать кодов аэропорта, но я оставил их на всякий случай, так как для аналитика так быстрее найти нужный аэропорт, также добавил и название ), название аэропорта (airport_name). с помощью coalesce выбираем что вписать: количество тразитных пассажиров, если они есть, если их нрет, то 0. затем высчитываем округленный с помощью round(округляем результат деления тразитных на обычных, приведенное к decimal для точности) процент транзитных и обычных пассажиров(очень удобная метрика для аналитики). Далее снова с coalesce выбираем либо 0 либо значение среднего времени, которое сначала переводим в секунды для точности, а финальный вариант переводим в часы для удобства аналитики. Затем добавляем столбцы с полученными данными через left join, предварительно отфильровав по наличию транзитных пассажиров и отсортировав по транзитным пассажирам(по убыванию). Ну и в конце я вывожу только первые 20 с помощью limit 20, это уже просто чтобы не выводилось долго при больших данных и быстрой проверки работоспособности sql-запроса 