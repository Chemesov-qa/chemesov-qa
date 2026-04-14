# Тестирование API: ПАК «Кордон» (Внутренние и внешние сервисы)

Этот раздел посвящен проверке программных интерфейсов (API), обеспечивающих взаимодействие между камерами фиксации, центральным сервером и внешними базами данных ГИБДД.

---

## 1. Стек инструментов
*   **Postman:** Создание коллекций, окружений (Environments) и скриптов автоматизации.
*   **Newman:** CLI-запуск коллекций для интеграции в CI/CD процессы.
*   **JavaScript:** Написание автотестов внутри Postman (библиотека `pm.test`).
*   **Swagger:** Изучение документации и структуры эндпоинтов.

---

## 2. Основные объекты проверки
1.  **Event Logger API:** Отправка данных о зафиксированном нарушении (JSON с фото + метаданные).
2.  **Heartbeat Service:** Проверка статуса доступности камер и сетевых узлов (Healthcheck).
3.  **Config Manager:** Дистанционное изменение параметров камер (выдержка, зоны детекции).
4.  **Auth Service:** Работа с токенами доступа (Bearer/Basic Auth) для защиты каналов связи.

---

## 3. Примеры автотестов (Snippet)
Пример скрипта на JavaScript (Postman Tests) для проверки корректности регистрации нарушения:

```javascript
pm.test("Status code is 201 (Created)", function () {
    pm.response.to.have.status(201);
});

pm.test("Response time is less than 500ms", function () {
    pm.expect(pm.response.responseTime).to.be.below(500);
});

pm.test("Check Plate Number in response", function () {
    var jsonData = pm.response.json();
    pm.expect(jsonData.detected_plate).to.not.be.undefined;
});
```

---

[![НАЗАД К ПРОФИЛЮ](https://img.shields.io/badge/⬅_НАЗАД_К_ПРОФИЛЮ-6f42c1?style=for-the-badge)](https://github.com/Leonid-QA)
