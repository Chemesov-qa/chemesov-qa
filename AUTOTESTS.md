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

[**◀️ НАЗАД К ПРОФИЛЮ**](https://github.com)

