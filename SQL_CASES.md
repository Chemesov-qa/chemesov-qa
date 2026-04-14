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

[![НАЗАД К ПРОФИЛЮ](https://img.shields.io/badge/⬅_НАЗАД_К_ПРОФИЛЮ-6f42c1?style=for-the-badge)](https://github.com/Leonid-QA)




