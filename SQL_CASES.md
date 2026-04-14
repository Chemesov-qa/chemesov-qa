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

# 🗄 Кейсы тестирования базы данных (SQL)

Ниже приведены примеры SQL-запросов для верификации данных ПАК «Кордон». Все задачи, условия и логика проверок описаны комментариями внутри кода.

---

CASE #1: Верификация записи события (Data Integrity)
ЗАДАЧА: Проверить корректность создания записи после проезда ТС.
ПРОВЕРКА: Соответствие Timestamp и уверенность распознавания (score > 0.8).

SELECT id, plate_number, event_time, camera_id, confidence_score
FROM recognition_events
WHERE plate_number = 'A123BC777' 
  AND event_time >= '2024-05-20 10:00:00'
ORDER BY event_time DESC;



CASE #2: Поиск дубликатов (Data Consistency)
ЗАДАЧА: Выявить баг «Размножение событий» (один проезд — две записи).
ПРОВЕРКА: Отсутствие строк с идентичным номером и временем фиксации.

SELECT plate_number, event_time, COUNT(*)
FROM recognition_events
GROUP BY plate_number, event_time
HAVING COUNT(*) > 1;



CASE #3: Контроль задержек (Latency Check)
ЗАДАЧА: Выявить лаг между фиксацией на камере и записью в БД сервера.
ПРОВЕРКА: Стабильность канала передачи (допустимый порог < 10 сек).

SELECT id, camera_id, (received_at - event_time) AS latency_seconds
FROM recognition_events
WHERE (received_at - event_time) > interval '10 seconds'
ORDER BY latency_seconds DESC;



CASE #4: Проверка политики хранения (Retention Policy)
ЗАДАЧА: Проверить работу скрипта ротации старых данных.
ПРОВЕРКА: Соблюдение регламента хранения (30 дней) и отсутствие переполнения БД.

SELECT MIN(event_time) AS oldest_record, MAX(event_time) AS newest_record
FROM recognition_events;


CASE #5: Валидация нейросетевой классификации
ЗАДАЧА: Проверить точность распределения ТС по типам (легковой/грузовой).
ПРОВЕРКА: Корректность работы классификатора при высоком доверии (score > 0.95).

SELECT vehicle_type, COUNT(*) 
FROM recognition_events
WHERE confidence_score > 0.95
GROUP BY vehicle_type;



CASE #6: Проверка полноты данных (Constraints)
ЗАДАЧА: Найти записи с критическими пропусками для выписки штрафа.
ПРОВЕРКА: Отсутствие NULL в полях: номер, скорость, полоса движения.

SELECT * FROM recognition_events
WHERE plate_number IS NULL OR speed IS NULL OR lane_id IS NULL;



CASE #7: Детекция «отвалившихся» камер (Availability)
ЗАДАЧА: Выявить узлы, переставшие присылать данные (аппаратный сбой).
ПРОВЕРКА: Наличие свежих событий от каждой активной камеры за последний час.

SELECT camera_id, MAX(event_time) as last_event
FROM recognition_events
GROUP BY camera_id
HAVING MAX(event_time) < NOW() - INTERVAL '1 hour';



CASE #8: Аудит прав доступа (Security/RBAC)
ЗАДАЧА: Убедиться, что записи созданы только авторизованными службами.
ПРОВЕРКА: Отсутствие сторонних или ручных правок в таблице событий.

SELECT id, created_by, event_time
FROM recognition_events
WHERE created_by NOT IN ('camera_agent', 'api_service_worker');



CASE #9: Анализ пиковых нагрузок (Performance)
ЗАДАЧА: Оценить почасовую нагрузку на БД для планирования ресурсов.
ПРОВЕРКА: Максимальное количество транзакций (TPS) в часы пик.

SELECT date_trunc('hour', event_time) AS hour_slot, COUNT(*) AS events_per_hour
FROM recognition_events
GROUP BY hour_slot
ORDER BY events_per_hour DESC;



CASE #10: Верификация реляционных связей (Integrity)
ЗАДАЧА: Проверить корректность Join таблиц событий и реестра камер.
ПРОВЕРКА: События не должны приходить от камер со статусом 'Inactive' или 'Repair'.

SELECT e.id, e.plate_number, c.location_address, c.status
FROM recognition_events e
JOIN cameras_registry c ON e.camera_id = c.id
WHERE c.status != 'Active';


[![НАЗАД К ПРОФИЛЮ](https://img.shields.io/badge/⬅_НАЗАД_К_ПРОФИЛЮ-6f42c1?style=for-the-badge)](https://github.com/Leonid-QA)




