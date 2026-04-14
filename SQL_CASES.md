# 🗄 Кейсы тестирования базы данных (SQL)

В рамках проекта «Кордон» база данных является критическим узлом. Ниже приведены примеры SQL-запросов (PostgreSQL), которые я использовал для верификации данных «под капотом» системы, минуя графический интерфейс.

---

### Кейс №1: Верификация записи события (Data Integrity)
**Задача:** Убедиться, что после физического проезда автомобиля через зону контроля, запись в БД создается корректно и содержит все необходимые атрибуты (номер, время, ID камеры).

*   **Запрос:**
    ```sql
    SELECT id, plate_number, event_time, camera_id, confidence_score
    FROM recognition_events
    WHERE plate_number = 'A123BC777' 
      AND event_time >= '2024-05-20 10:00:00'
    ORDER BY event_time DESC;
    ```
*   **Что проверяем:** Соответствие `Timestamp` в базе реальному времени проезда и наличие высокого порога уверенности распознавания (`confidence_score > 0.8`).

---

### Кейс №2: Проверка связей между таблицами (Join Integrity)
**Задача:** Убедиться, что события в базе данных корректно привязаны к существующим устройствам и их гео-позициям.

*   **Запрос:**
    ```sql
    SELECT 
        e.event_time, 
        e.plate_number, 
        d.device_name, 
        d.location_address
    FROM recognition_events AS e
    JOIN devices AS d ON e.device_id = d.id
    WHERE d.status = 'Online'
    ORDER BY e.event_time DESC
    LIMIT 10;
    ```
*   **Что проверяем:** Отсутствие "осиротевших" записей (events без привязки к device) и корректность отображения адреса установки в логах.

---

### Кейс №3: Поиск логических ошибок (Data Duplication)
**Задача:** Проверить систему на наличие бага «Размножение дупла» — ситуации, когда из-за нестабильного сетевого сигнала один проезд фиксируется в базе дважды.

*   **Запрос:**
    ```sql
    SELECT plate_number, event_time, device_id, COUNT(*) as duplicate_count
    FROM recognition_events
    GROUP BY plate_number, event_time, device_id
    HAVING COUNT(*) > 1;
    ```
*   **Что проверяем:** Уникальность каждой фиксации. Если запрос возвращает строки — это свидетельствует о дефекте в логике сохранения данных (Race Condition или отсутствие Unique Constraint).

---

### Кейс №4: Проверка производительности (Aggregation)
**Задача:** Быстрая сверка статистики за час для сравнения данных в UI-отчете и в «сырой» базе.

*   **Запрос:**
    ```sql
    SELECT camera_id, COUNT(*) as total_cars
    FROM recognition_events
    WHERE event_time BETWEEN '2024-05-20 09:00:00' AND '2024-05-20 10:00:00'
    GROUP BY camera_id;
    ```
*   **Что проверяем:** Сходятся ли цифры в графиках мониторинга с реальным количеством записей в таблице.

---

-- Кейс 1: Верификация записи события
SELECT id, plate_number, event_time, camera_id, confidence_score
FROM recognition_events
WHERE plate_number = 'A123BC777' 
  AND event_time >= '2024-05-20 10:00:00'
ORDER BY event_time DESC;

-- Кейс 2: Проверка связей между таблицами
SELECT 
    e.event_time, 
    e.plate_number, 
    d.device_name, 
    d.location_address
FROM recognition_events AS e
JOIN devices AS d ON e.device_id = d.id
WHERE d.status = 'Online'
ORDER BY e.event_time DESC
LIMIT 10;

-- Кейс 3: Поиск дубликатов
SELECT plate_number, event_time, device_id, COUNT(*) as duplicate_count
FROM recognition_events
GROUP BY plate_number, event_time, device_id
HAVING COUNT(*) > 1;

-- Кейс 4: Агрегация по камерам
SELECT camera_id, COUNT(*) as total_cars
FROM recognition_events
WHERE event_time BETWEEN '2024-05-20 09:00:00' AND '2024-05-20 10:00:00'
GROUP BY camera_id;

-- Кейс 5: Поиск событий с низкой уверенностью
SELECT id, plate_number, event_time, camera_id, confidence_score
FROM recognition_events
WHERE confidence_score < 0.5
  AND event_time >= CURRENT_DATE - INTERVAL '7 days'
ORDER BY confidence_score ASC;

-- Кейс 6: Проверка оффлайн устройств и их последней активности
SELECT 
    d.device_name,
    d.location_address,
    d.last_seen,
    EXTRACT(EPOCH FROM (NOW() - d.last_seen))/60 AS minutes_offline
FROM devices AS d
WHERE d.status = 'Offline'
  AND d.last_seen < NOW() - INTERVAL '10 minutes'
ORDER BY d.last_seen ASC;

-- Кейс 7: Топ-10 часто встречающихся номеров
SELECT 
    plate_number,
    COUNT(*) as detection_count,
    COUNT(DISTINCT device_id) as unique_cameras
FROM recognition_events
WHERE event_time >= CURRENT_DATE - INTERVAL '24 hours'
GROUP BY plate_number
ORDER BY detection_count DESC
LIMIT 10;

-- Кейс 8: Проверка целостности внешних ключей
SELECT e.id, e.event_time, e.device_id
FROM recognition_events e
LEFT JOIN devices d ON e.device_id = d.id
WHERE d.id IS NULL;

-- Кейс 9: Анализ пиковой нагрузки по часам
SELECT 
    EXTRACT(HOUR FROM event_time) as hour_of_day,
    COUNT(*) as events_count,
    AVG(confidence_score) as avg_confidence
FROM recognition_events
WHERE event_time >= CURRENT_DATE - INTERVAL '30 days'
GROUP BY EXTRACT(HOUR FROM event_time)
ORDER BY hour_of_day;

-- Кейс 10: Поиск аномалий по скорости между камерами
SELECT 
    e1.plate_number,
    e1.event_time as time_camera_1,
    e2.event_time as time_camera_2,
    d1.location_address as location_1,
    d2.location_address as location_2,
    EXTRACT(EPOCH FROM (e2.event_time - e1.event_time)) as seconds_between
FROM recognition_events e1
JOIN recognition_events e2 
    ON e1.plate_number = e2.plate_number 
    AND e1.id < e2.id
JOIN devices d1 ON e1.device_id = d1.id
JOIN devices d2 ON e2.device_id = d2.id
WHERE e1.event_time >= CURRENT_DATE - INTERVAL '1 day'
  AND e2.event_time - e1.event_time < INTERVAL '10 seconds'
  AND e1.device_id != e2.device_id
ORDER BY seconds_between ASC
LIMIT 20;


[![НАЗАД К ПРОФИЛЮ](https://img.shields.io/badge/⬅_НАЗАД_К_ПРОФИЛЮ-6f42c1?style=for-the-badge)](https://github.com/Leonid-QA)




