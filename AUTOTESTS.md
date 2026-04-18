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
## 2. Тест проверки создания новой заявки на пропуск
**Зачем написал:** Автоматизация рутинной операции создания заявки для проверки работы формы ввода и сохранения данных в системе.

<details>
<summary> Посмотреть код скрипта</summary>

python
from selenium import webdriver
from selenium.webdriver.common.by import By
from selenium.webdriver.support.ui import Select
import time

def test_create_pass_request():
    driver = webdriver.Chrome()
    driver.get("http://cordon-system.local/requests/new")
    
    # Заполнение формы
    driver.find_element(By.ID, "visitor_name").send_keys("Иванов Иван Иванович")
    driver.find_element(By.ID, "visitor_passport").send_keys("4510 123456")
    Select(driver.find_element(By.ID, "pass_type")).select_by_visible_text("Временный")
    driver.find_element(By.ID, "valid_from").send_keys("01.12.2023")
    driver.find_element(By.ID, "valid_to").send_keys("31.12.2023")
    driver.find_element(By.ID, "car_number").send_keys("А123ВС177")
    
    driver.find_element(By.ID, "btn_save").click()
    time.sleep(1)
    
    success_message = driver.find_element(By.CLASS_NAME, "alert-success").text
    assert "Заявка успешно создана" in success_message
    driver.quit()

</details>

---
## 3. Тест выхода из системы (Logout)
**Зачем написал:** Проверка корректности завершения сессии и возврата на страницу входа.

<details>
<summary> Посмотреть код скрипта</summary>

python
from selenium import webdriver
from selenium.webdriver.common.by import By

def test_logout():
    driver = webdriver.Chrome()
    driver.get("http://cordon-system.local")
    
    # Предварительный вход
    driver.find_element(By.ID, "user").send_keys("Operaton_1")
    driver.find_element(By.ID, "pass").send_keys("Pass123")
    driver.find_element(By.ID, "submit").click()
    
    # Выход
    driver.find_element(By.ID, "user_menu_dropdown").click()
    driver.find_element(By.LINK_TEXT, "Выход").click()
    
    assert "Вход в систему" in driver.page_source
    driver.quit()

</details>

---
## 4. Тест поиска по номеру пропуска
**Зачем написал:** Валидация работы поискового фильтра в реестре выданных пропусков.

<details>
<summary> Посмотреть код скрипта</summary>

python
from selenium import webdriver
from selenium.webdriver.common.by import By
from selenium.webdriver.common.keys import Keys

def test_search_pass_by_number():
    driver = webdriver.Chrome()
    driver.get("http://cordon-system.local/passes")
    
    search_field = driver.find_element(By.ID, "search_query")
    search_field.send_keys("PASS-2023-001")
    search_field.send_keys(Keys.RETURN)
    
    # Проверка результата
    result_row = driver.find_element(By.XPATH, "//table/tbody/tr[1]/td[2]")
    assert "PASS-2023-001" in result_row.text
    driver.quit()

</details>

---
## 5. Тест проверки валидации пустого поля "Номер автомобиля"
**Зачем написал:** Убедиться, что система не сохраняет заявку без обязательного поля и выводит предупреждение.

<details>
<summary> Посмотреть код скрипта</summary>

python
from selenium import webdriver
from selenium.webdriver.common.by import By
import time

def test_validation_empty_car_number():
    driver = webdriver.Chrome()
    driver.get("http://cordon-system.local/requests/new")
    
    driver.find_element(By.ID, "visitor_name").send_keys("Петров Петр Петрович")
    driver.find_element(By.ID, "visitor_passport").send_keys("4511 654321")
    driver.find_element(By.ID, "btn_save").click()
    time.sleep(1)
    
    error_message = driver.find_element(By.ID, "car_number-error").text
    assert "Поле обязательно для заполнения" in error_message
    driver.quit()

</details>

---
## 6. Тест редактирования профиля оператора
**Зачем написал:** Проверка изменения контактных данных в личном кабинете оператора КПП.

<details>
<summary> Посмотреть код скрипта</summary>

python
from selenium import webdriver
from selenium.webdriver.common.by import By
import time

def test_edit_operator_profile():
    driver = webdriver.Chrome()
    driver.get("http://cordon-system.local")
    
    # Логин
    driver.find_element(By.ID, "user").send_keys("Operaton_1")
    driver.find_element(By.ID, "pass").send_keys("Pass123")
    driver.find_element(By.ID, "submit").click()
    
    # Переход в профиль
    driver.find_element(By.LINK_TEXT, "Профиль").click()
    driver.find_element(By.ID, "phone").clear()
    driver.find_element(By.ID, "phone").send_keys("+7 999 123 45 67")
    driver.find_element(By.ID, "btn_update_profile").click()
    time.sleep(1)
    
    updated_phone = driver.find_element(By.ID, "phone").get_attribute("value")
    assert updated_phone == "+7 999 123 45 67"
    driver.quit()

</details>

---
## 7. Тест проверки работы календаря при выборе даты
**Зачем написал:** Автоматизация проверки UI-компонента DatePicker, чтобы избежать ручного кликанья по датам.

<details>
<summary> Посмотреть код скрипта</summary>

python
from selenium import webdriver
from selenium.webdriver.common.by import By
import time

def test_datepicker_functionality():
    driver = webdriver.Chrome()
    driver.get("http://cordon-system.local/reports")
    
    driver.find_element(By.ID, "date_from").click()
    driver.find_element(By.XPATH, "//div[@class='datepicker-days']//td[text()='15']").click()
    
    selected_date = driver.find_element(By.ID, "date_from").get_attribute("value")
    assert selected_date != ""
    driver.quit()

</details>

---
## 8. Тест проверки наличия элементов интерфейса на главной странице (UI Elements Check)
**Зачем написал:** Быстрый тест после деплоя, чтобы убедиться, что верстка не развалилась и все ключевые блоки на месте.

<details>
<summary> Посмотреть код скрипта</summary>

python
from selenium import webdriver
from selenium.webdriver.common.by import By

def test_dashboard_ui_elements_presence():
    driver = webdriver.Chrome()
    driver.get("http://cordon-system.local")
    
    driver.find_element(By.ID, "user").send_keys("Operaton_1")
    driver.find_element(By.ID, "pass").send_keys("Pass123")
    driver.find_element(By.ID, "submit").click()
    
    assert driver.find_element(By.ID, "sidebar_menu").is_displayed()
    assert driver.find_element(By.ID, "stats_widget").is_displayed()
    assert driver.find_element(By.ID, "recent_activity").is_displayed()
    
    driver.quit()

</details>

---
## 9. Тест фильтрации списка посетителей по статусу
**Зачем написал:** Проверка работы выпадающего списка с AJAX-обновлением таблицы на странице "Журнал посетителей".

<details>
<summary> Посмотреть код скрипта</summary>

python
from selenium import webdriver
from selenium.webdriver.common.by import By
from selenium.webdriver.support.ui import Select
import time

def test_filter_visitors_by_status():
    driver = webdriver.Chrome()
    driver.get("http://cordon-system.local/journal")
    
    status_filter = Select(driver.find_element(By.ID, "filter_status"))
    status_filter.select_by_visible_text("На территории")
    driver.find_element(By.ID, "apply_filter").click()
    time.sleep(2)
    
    rows = driver.find_elements(By.XPATH, "//table/tbody/tr")
    for row in rows:
        status_cell = row.find_element(By.XPATH, "./td[5]")
        assert status_cell.text == "На территории"
    
    driver.quit()

</details>

---
## 10. Тест загрузки фотографии при регистрации посетителя
**Зачем написал:** Проверка функционала прикрепления файлов (имитация сканирования паспорта) к заявке.

<details>
<summary> Посмотреть код скрипта</summary>

python
from selenium import webdriver
from selenium.webdriver.common.by import By
import os
import time

def test_upload_passport_scan():
    driver = webdriver.Chrome()
    driver.get("http://cordon-system.local/requests/new")
    
    # Путь к тестовому файлу
    file_path = os.path.abspath("test_passport.jpg")
    
    upload_input = driver.find_element(By.ID, "passport_scan")
    upload_input.send_keys(file_path)
    
    # Проверка появления превьюшки
    time.sleep(2)
    preview = driver.find_element(By.ID, "upload_preview")
    assert preview.is_displayed()
    
    driver.quit()

</details>

[![[НАЗАД К ПРОФИЛЮ](https://img.shields.io/badge/-НаЗАД_К_ПРОФИЛЮ-6f42c1?style=for-the-badge)](https://github.com/Leonid-QA)

