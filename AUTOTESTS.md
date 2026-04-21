# 🤖 Автоматизация UI-тестирования (Selenium + Python)

В данном разделе представлены примеры автотестов, которые я разработал для оптимизации регрессионного тестирования системы «Кордон». 

> **Примечание:** Скрипты являются демонстрационными. Для запуска в реальной среде требуется доступ к закрытому контуру ПАК «Кордон».

---

### 1. Тест авторизации (Smoke Test)
**Зачем написал:** Чтобы не вводить логин/пароль вручную при каждой проверке. Скрипт проверяет базовую доступность системы и корректность работы БД пользователей.

<details>
<summary>📦 Посмотреть код скрипта</summary>

```python

from selenium import webdriver
from selenium.webdriver.common.by import By

def test_login():
    driver = webdriver.Chrome()
    driver.get("http://cordon-system.local")
    driver.find_element(By.ID, "user").send_keys("Operator_1")
    driver.find_element(By.ID, "pass").send_keys("Pass123")
    driver.find_element(By.ID, "submit").click()
    assert "Dashboard" in driver.title
    driver.quit()
```
</details>

---
## 2. Тест проверки поступления данных с рубежа (Данные с радара)
**Зачем написал:** Автоматизация проверки того, что данные с физического датчика скорости успешно доходят до веб-интерфейса и отображаются в таблице событий.

<details>
<summary>📦 Посмотреть код скрипта</summary>

```python

from selenium import webdriver
from selenium.webdriver.common.by import By
import time

def test_speed_data_arrival():
    driver = webdriver.Chrome()
    driver.get("http://cordon-photo.local:8080/events")
    time.sleep(5)  # Ожидание обновления AJAX
    
    # Поиск последней записи с зафиксированной скоростью
    last_record = driver.find_element(By.XPATH, "//table[@id='events-table']/tbody/tr[1]/td[5]")
    assert "км/ч" in last_record.text
    driver.quit()
```
</details>

---
## 3. Тест проверки SNR и качества канала связи с камерой
**Зачем написал:** Как инженеру связи мне важно оперативно проверять уровень сигнала (SNR) на удаленных рубежах без захода в CLI коммутатора.

<details>
<summary>📦 Посмотреть код скрипта</summary>

```python
from selenium import webdriver
from selenium.webdriver.common.by import By

def test_channel_quality_snr():
    driver = webdriver.Chrome()
    driver.get("http://cordon-photo.local:8080/devices/status")
    
    # Переход на вкладку диагностики конкретного рубежа (ID=103)
    driver.find_element(By.LINK_TEXT, "Рубеж 103 (М-4 Дон)").click()
    snr_value = driver.find_element(By.ID, "snr_camera_1").text
    
    # Проверка, что SNR не ниже порогового значения 25 dB
    assert float(snr_value.replace(" dB", "")) > 25.0
    driver.quit()
```
</details>

---
## 4. Тест корректности распознавания ГРЗ (Госномер)
**Зачем написал:** Проверка, что модуль распознавания возвращает строку формата РФ, а не ошибку "Undefined" при отображении в GUI.

<details>
<summary>📦 Посмотреть код скрипта</summary>

```python

from selenium import webdriver
from selenium.webdriver.common.by import By
import re

def test_lpr_plate_format():
    driver = webdriver.Chrome()
    driver.get("http://cordon-photo.local:8080/events")
    
    plate_element = driver.find_element(By.XPATH, "//table[@id='events-table']/tbody/tr[1]/td[3]")
    plate_text = plate_element.text
    
    # Простая проверка маски российского номера (A000AA)
    pattern = r'^[АВЕКМНОРСТУХ]\d{3}[АВЕКМНОРСТУХ]{2}\s*\d{2,3}$'
    assert re.match(pattern, plate_text) is not None
    driver.quit()
```
</details>

---
## 5. Тест наличия фотоматериала высокого разрешения
**Зачем написал:** Валидация того, что ссылка на полный кадр с камеры (full frame) активна и ведет к изображению, а не к битой ссылке (404).

<details>
<summary>📦 Посмотреть код скрипта</summary>

```python

from selenium import webdriver
from selenium.webdriver.common.by import By
import requests

def test_full_frame_image_availability():
    driver = webdriver.Chrome()
    driver.get("http://cordon-photo.local:8080/events")
    
    # Клик по иконке просмотра фото
    driver.find_element(By.XPATH, "//table[@id='events-table']/tbody/tr[1]/td[7]/a").click()
    
    img = driver.find_element(By.ID, "full-frame-image")
    src = img.get_attribute("src")
    
    # Проверяем HTTP статус ответа для URL картинки
    response = requests.get(src, auth=('operator_gibdd', 'SecurePass2024!'))
    assert response.status_code == 200
    driver.quit()
```
</details>

---
## 6. Тест фильтрации событий по полосе движения
**Зачем написал:** Проверка работоспособности AJAX-фильтра в АРМ оператора (актуально для многополосных рубежей контроля).

<details>
<summary>📦 Посмотреть код скрипта</summary>

```python

from selenium import webdriver
from selenium.webdriver.common.by import By
from selenium.webdriver.support.ui import Select
import time

def test_lane_filter():
    driver = webdriver.Chrome()
    driver.get("http://cordon-photo.local:8080/events")
    
    lane_selector = Select(driver.find_element(By.ID, "lane_filter"))
    lane_selector.select_by_visible_text("Полоса 2")
    driver.find_element(By.ID, "apply_filter_btn").click()
    time.sleep(2)
    
    # Проверяем, что все отображаемые записи действительно с полосы 2
    rows = driver.find_elements(By.XPATH, "//table[@id='events-table']/tbody/tr/td[4]")
    for row in rows:
        assert row.text == "2"
    
    driver.quit()
```
</details>

---
## 7. Тест синхронизации времени NTP в веб-интерфейсе
**Зачем написал:** Критически важно для доказательной базы. Проверяю, что время на сервере не расходится с эталонным более чем на 1 секунду.

<details>
<summary>📦 Посмотреть код скрипта</summary>

```python

from selenium import webdriver
from selenium.webdriver.common.by import By
from datetime import datetime
import time

def test_ntp_time_sync():
    driver = webdriver.Chrome()
    driver.get("http://cordon-photo.local:8080/status")
    
    server_time_str = driver.find_element(By.ID, "server_time_display").text
    # Пример формата: "2026-04-18 15:30:45"
    server_time = datetime.strptime(server_time_str, "%Y-%m-%d %H:%M:%S")
    local_time = datetime.now()
    
    diff = abs((local_time - server_time).total_seconds())
    assert diff < 2.0  # Допустимая погрешность
    driver.quit()
```
</details>

---
## 8. Тест проверки статуса накопителя (HDD/SSD) на сервере видеофиксации
**Зачем написал:** Мониторинг свободного места на диске архива, чтобы избежать остановки записи из-за переполнения хранилища.

<details>
<summary>📦 Посмотреть код скрипта</summary>

```python

from selenium import webdriver
from selenium.webdriver.common.by import By

def test_storage_capacity_check():
    driver = webdriver.Chrome()
    driver.get("http://cordon-photo.local:8080/admin/storage")
    
    free_space_str = driver.find_element(By.ID, "archive_free_space").text
    # Пример: "3.2 TB free"
    value = float(free_space_str.split()[0])
    unit = free_space_str.split()[1]
    
    # Приводим всё к GB для единой проверки
    if "TB" in unit:
        value *= 1024
    
    # Тревога если осталось меньше 200 GB
    assert value > 200.0
    driver.quit()
```
</details>

---
## 9. Тест отображения наложенных метаданных на изображении
**Зачем написал:** Убедиться, что оверлей (скорость, дата, координаты) корректно рендерится поверх фото на стороне браузера.

<details>
<summary>📦 Посмотреть код скрипта</summary>

```python

from selenium import webdriver
from selenium.webdriver.common.by import By
import time

def test_overlay_metadata_display():
    driver = webdriver.Chrome()
    driver.get("http://cordon-photo.local:8080/events")
    driver.find_element(By.XPATH, "//table[@id='events-table']/tbody/tr[1]/td[7]/a").click()
    time.sleep(2)
    
    # Проверяем наличие canvas или svg слоя с метаданными
    overlay = driver.find_element(By.ID, "photo_overlay")
    assert overlay.is_displayed()
    assert "скорость" in overlay.text.lower()
    driver.quit()
```
</details>

---
## 10. Тест экспорта отчета в Excel (Выгрузка постановлений)
**Зачем написал:** Проверка целостности файла экспорта, который потом загружается в систему ГИС ГМП.

<details>
<summary>📦 Посмотреть код скрипта</summary>


```python

python
from selenium import webdriver
from selenium.webdriver.common.by import By
import os
import glob
import time

def test_export_resolutions_xls():
    driver = webdriver.Chrome()
    download_dir = "C:\\Users\\Engineer\\Downloads"
    
    # Настройка браузера на автоматическое скачивание без диалога
    prefs = {"download.default_directory": download_dir}
    options = webdriver.ChromeOptions()
    options.add_experimental_option("prefs", prefs)
    driver = webdriver.Chrome(options=options)
    
    driver.get("http://cordon-photo.local:8080/reports")
    driver.find_element(By.ID, "export_xls_btn").click()
    time.sleep(5)  # Ожидание загрузки
    
    # Поиск самого свежего xls файла
    list_of_files = glob.glob(f"{download_dir}\\*.xls")
    latest_file = max(list_of_files, key=os.path.getctime)
    
    assert os.path.exists(latest_file)
    assert os.path.getsize(latest_file) > 1024  # Файл не пустой
    
    driver.quit()
```
</details>

---

### 📊 Мои проекты

### 🛠 Проекты и опыт (QA Engineering)

| Проект | Описание | Стек технологий | Ссылка |
| :---: | :---: | :---: | :---: |
| **Тестовая документация** | Создание тест-кейсов и чек-листов для ПАК «Кордон» | Jira, TestRail, MS Excel | [![Doc](https://img.shields.io/badge/🗄️_Тестовая_документация-3A6D8C?style=for-the-badge)](./TEST_DOCS_KORDON.md) |
| **API тестирование** | Автоматизированные коллекции тестов для REST-сервисов | Postman, Swagger | [![API](https://img.shields.io/badge/🔌_API_тесты-C75B39?style=for-the-badge)](./API_TESTING_POSTMAN.md) |
| **SQL запросы** | Валидация данных и сложные выборки для проверки БД | MySQL, DBeaver | [![SQL](https://img.shields.io/badge/🗄️_SQL_запросы-B8860B?style=for-the-badge)](./TEST_DOCS_KORDON.md) |
| **Mind-map** | Визуализация стратегии покрытия<br>Декомпозиция проекта | XMind | [![Mind-map](https://img.shields.io/badge/🗺️_Mind--Map-2D7A4B?style=for-the-badge)](./MAP/KORDON_MINDMAP_CASE.md) |
| **План / Test Plan** | Стратегия обеспечения качества<br> Методология проверок | Markdown, Confluence | [![Test Plan](https://img.shields.io/badge/📋_Test_Plan-E67E22?style=for-the-badge)](./TEST_PLAN_KORDON.md) |
| **Test Summary Report** | Отчет по итогам цикла тестирования с метриками | Allure, MS Word | [![Отчёт](https://img.shields.io/badge/📊_Test_Report-6B4A7A?style=for-the-badge)](./TEST_SUMMARY.md) |

---

[![НАЗАД К ПРОФИЛЮ](https://img.shields.io/badge/⬅_НАЗАД_К_ПРОФИЛЮ-6f42c1?style=for-the-badge)](https://github.com/Leonid-QA)

