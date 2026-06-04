# 📦 Руководство по запуску и тестированию Order Service

Это руководство содержит подробные инструкции по запуску всех микросервисов проекта и пошаговый план тестирования **Order Service** в Postman.

---

## 🏗 Архитектура и порты

Для полноценной работы заказа должны быть запущены **все 4 сервиса**, так как они тесно взаимодействуют друг с другом.

| Сервис | Порт | Роль в процессе заказа |
|--------|------|-------------------------|
| **User Service** | `8000` | Аутентификация, проверка JWT-токена пользователя |
| **Product Service** | `8001` | Резервирование и освобождение товаров на складе |
| **Cart Service** | `8002` | Получение содержимого корзины для оформления заказа |
| **Order Service** | `8003` | Создание заказа, управление статусами, статистика |

### 🛠 Зависимости
- **Redis**: Используется для обмена событиями между сервисами (например, `order.created`).
- **SQLite**: Каждый сервис имеет свою базу данных в директории `databases/`.

---

### 0. Запуск Redis (Инфраструктура)
В корне проекта выполните команду для запуска Redis в Docker:
```bash
docker-compose up -d
```
*Это запустит Redis, который необходим для обмена событиями между сервисами.*

### 1. Подготовка баз данных (только при первом запуске)
Убедитесь, что папка `databases` существует в корне проекта.

### 2. Запуск сервисов (в разных терминалах)

#### 👤 Терминал 1 — User Service
```bash
cd services/user-services
source venv/bin/activate
python manage.py migrate
python manage.py runserver 0.0.0.0:8000
```

#### 📦 Терминал 2 — Product Service
```bash
cd services/product-service
source venv/bin/activate
python manage.py migrate
python manage.py runserver 0.0.0.0:8001
```

#### 🛒 Терминал 3 — Cart Service
```bash
cd services/cart-services
source venv/bin/activate
python manage.py migrate
python manage.py runserver 0.0.0.0:8002
```

#### 📑 Терминал 4 — Order Service
```bash
cd services/order-service
source venv/bin/activate
python manage.py migrate
python manage.py runserver 0.0.0.0:8003
```

> ⚠️ **ВАЖНО**: В файле `services/order-service/config/settings.py` проверьте параметр `USER_SERVICE_URL`. Если User Service запущен на 8000, значение должно быть `http://localhost:8000`.

---

## 📮 Тестирование Order Service в Postman

Процесс создания заказа требует последовательных действий во всех сервисах.

### Шаг 1: Получение токена (User Service)
1. **Регистрация** (если еще нет): `POST http://localhost:8000/api/users/register/`
2. **Логин**: `POST http://localhost:8000/api/auth/login/`
   - Сохраните значение `access` из ответа.
   - В Postman во всех последующих запросах используйте вкладку **Authorization** -> **Bearer Token**.

### Шаг 2: Создание товара (Product Service)
Для заказа нужны товары в базе.
1. **Создать категорию**: `POST http://localhost:8001/api/categories/`
   ```json
   {"name": "Электроника"}
   ```
2. **Создать товар**: `POST http://localhost:8001/api/products/`
   ```json
   {
     "name": "Смартфон",
     "price": "50000.00",
     "category": 1,
     "stock_quantity": 10
   }
   ```
   - Запомните `id` созданного продукта.

### Шаг 3: Наполнение корзины (Cart Service)
Заказ создается на основе текущей корзины пользователя.
1. **Добавить в корзину**: `POST http://localhost:8002/api/cart/add/`
   - Headers: `Authorization: Bearer <ваш_токен>`
   ```json
   {
     "product_id": 1,
     "quantity": 2
   }
   ```

### Шаг 4: Оформление заказа (Order Service)
Теперь можно создать заказ.
1. **Создать заказ**: `POST http://localhost:8003/api/orders/create/`
   - Headers: `Authorization: Bearer <ваш_токен>`
   - Body:
     ```json
     {
       "shipping_address": "Москва, ул. Пушкина, д. 10",
       "customer_info": {
         "first_name": "Иван",
         "last_name": "Иванов",
         "email": "ivan@example.com"
       },
       "special_instructions": "Позвонить за час до доставки"
     }
     ```
   - **Что происходит внутри**:
     - Order Service проверяет ваш токен через User Service.
     - Получает товары из Cart Service.
     - Резервирует количество в Product Service.
     - Создает запись заказа и публикует событие в Redis.

### Шаг 5: Управление заказом (Order Service)
1. **Список заказов**: `GET http://localhost:8003/api/orders/`
2. **Детали заказа**: `GET http://localhost:8003/api/orders/<id>/`
3. **Обновление статуса**: `PUT http://localhost:8003/api/orders/<id>/status/`
   ```json
   {"status": "confirmed"}
   ```
   *Доступные статусы: `pending`, `confirmed`, `shipped`, `delivered`, `cancelled`.*
4. **Статистика**: `GET http://localhost:8003/api/orders/statistics/`
   - Возвращает количество заказов по статусам и общую сумму трат.

---

## 🔍 Возможные проблемы и решения

| Проблема | Причина | Решение |
|----------|---------|---------|
| `500 Internal Server Error` при создании заказа | Один из сервисов недоступен или Redis не запущен | Проверьте `health/` эндпоинты всех сервисов и статус Redis |
| `401 Unauthorized` | Токен просрочен или не передан | Получите новый токен через `/api/auth/refresh/` |
| `Failed to reserve products` | Недостаточно товара на складе | Увеличьте `stock_quantity` через Product Service |
| Заказ создается, но корзина не очищается | Не настроен обработчик событий | Убедитесь, что Cart Service слушает Redis канал событий |

---

## 📡 Health Check эндпоинты
Всегда проверяйте готовность перед тестами:
- User: `http://localhost:8000/health/`
- Product: `http://localhost:8001/health/`
- Cart: `http://localhost:8002/health/`
- Order: `http://localhost:8003/health/`
