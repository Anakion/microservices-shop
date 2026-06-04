# 📬 Полное руководство по тестированию микросервисов в Postman

> Это руководство покрывает **все 3 сервиса** проекта: User Service, Product Service и Cart Service.
> Включает тестирование каждого сервиса **отдельно** и **межсервисное взаимодействие**.

---

## 📑 Содержание

1. [Архитектура проекта](#-архитектура-проекта)
2. [Подготовка к тестированию](#-подготовка-к-тестированию)
3. [User Service (порт 8000)](#-user-service--порт-8000)
4. [Product Service (порт 8001)](#-product-service--порт-8001)
5. [Cart Service (порт 8002)](#-cart-service--порт-8002)
6. [Межсервисное тестирование](#-межсервисное-тестирование)
7. [Автоматизация через Collection Runner](#-автоматизация-через-collection-runner)
8. [Устранение типичных ошибок](#-устранение-типичных-ошибок)

---

## 🏗 Архитектура проекта

```
┌─────────────────┐     ┌──────────────────┐     ┌─────────────────┐
│  USER SERVICE   │     │ PRODUCT SERVICE   │     │  CART SERVICE   │
│   :8000         │     │   :8001           │     │   :8002         │
│                 │     │                   │     │                 │
│ • Регистрация   │     │ • Категории       │     │ • Корзина       │
│ • Авторизация   │     │ • Продукты        │     │ • Добавление    │
│ • JWT-токены    │     │ • Резервирование  │     │ • Обновление    │
│ • Профиль       │     │ • Проверка наличия│     │ • Удаление      │
│                 │     │                   │     │                 │
│   SQLite:       │     │   SQLite:         │     │   SQLite:       │
│   user.db       │     │   product.db      │     │   cart.db       │
└────────┬────────┘     └────────┬──────────┘     └────────┬────────┘
         │                       │                          │
         │       ┌───────────────┴──────────────┐           │
         │       │                              │           │
         └───────┤  Cart Service обращается к:  ├───────────┘
                 │  • User Service (валидация    │
                 │    токена)                    │
                 │  • Product Service (проверка  │
                 │    наличия товара)            │
                 └──────────────────────────────┘
```

### Порты сервисов

| Сервис | Порт | Аутентификация | БД |
|--------|------|----------------|-----|
| **User Service** | `8000` | JWT (SimpleJWT) | `databases/user.db` |
| **Product Service** | `8001` | `AllowAny` (открытый) | `databases/product.db` |
| **Cart Service** | `8002` | JWT через middleware → User Service | `databases/cart.db` |

### Как сервисы связаны между собой

```
Cart Service ──HTTP GET──► Product Service   (проверка наличия товара)
Cart Service ──HTTP GET──► User Service      (валидация JWT-токена пользователя)
Product Service ◄──Redis──► Cart Service     (события: order.created, order.cancelled)
```

---

## 🔧 Подготовка к тестированию

### 1. Создание окружения (Environment) в Postman

Откройте Postman → **Environments** → **+** → введите:

| Variable | Initial Value | Описание |
|----------|--------------|----------|
| `user_url` | `http://localhost:8000` | Базовый URL User Service |
| `product_url` | `http://localhost:8001` | Базовый URL Product Service |
| `cart_url` | `http://localhost:8002` | Базовый URL Cart Service |
| `access_token` | *(пусто)* | JWT access-токен (заполнится автоматически) |
| `refresh_token` | *(пусто)* | JWT refresh-токен (заполнится автоматически) |
| `user_id` | *(пусто)* | ID зарегистрированного пользователя |
| `category_id` | *(пусто)* | ID созданной категории |
| `category_slug` | *(пусто)* | Slug созданной категории |
| `product_id` | *(пусто)* | ID созданного продукта |
| `cart_item_id` | *(пусто)* | ID товара в корзине |

> 💡 **Совет:** Назовите окружение `Microservices Shop — Dev`

### 2. Запуск всех сервисов

Откройте **3 отдельных терминала**:

**Терминал 1 — User Service:**
```bash
cd services/user-services
source venv/bin/activate
python manage.py migrate          # только первый раз
python manage.py createsuperuser  # только первый раз
python manage.py runserver 0.0.0.0:8000
```

**Терминал 2 — Product Service:**
```bash
cd services/product-service
source venv/bin/activate
python manage.py migrate          # только первый раз
python manage.py createsuperuser  # только первый раз
python manage.py runserver 0.0.0.0:8001
```

**Терминал 3 — Cart Service:**
```bash
cd services/cart-services
source venv/bin/activate
python manage.py migrate          # только первый раз
python manage.py runserver 0.0.0.0:8002
```

> ⚠️ **ВАЖНО:** Cart Service в `config/settings.py` имеет `USER_SERVICE_URL = 'http://localhost:8004'`.
> Если User Service работает на порту `8000`, нужно исправить на `http://localhost:8000`.

### 3. Проверка Health Check каждого сервиса

Перед началом тестирования убедитесь что все сервисы запущены:

| Сервис | URL | Ожидаемый ответ |
|--------|-----|-----------------|
| User Service | `GET http://localhost:8000/health/` | `{"status": "healthy", "service": "user-service"}` |
| Product Service | `GET http://localhost:8001/health/` | `{"status": "healthy", "service": "product-service"}` |
| Cart Service | `GET http://localhost:8002/health/` | `{"status": "healthy", "service": "cart-service"}` |

---

## 👤 User Service — порт 8000

> User Service отвечает за регистрацию, авторизацию (JWT) и управление профилем пользователя.
> Используется **кастомная модель** `User` (логин по `email`, не по `username`).
> **Аутентификация:** `SimpleJWT` — access-токен живёт 60 мин, refresh — 7 дней.

### Структура URL

| Метод | Endpoint | Аутентификация | Описание |
|-------|----------|----------------|----------|
| `POST` | `/api/users/register/` | 🔓 Нет | Регистрация нового пользователя |
| `POST` | `/api/auth/login/` | 🔓 Нет | Авторизация, получение токенов |
| `POST` | `/api/auth/refresh/` | 🔓 Нет | Обновление access-токена |
| `GET` | `/api/users/profile/` | 🔒 JWT | Получение профиля (с профилем) |
| `PUT` | `/api/users/profile/update/` | 🔒 JWT | Обновление профиля пользователя |
| `GET` | `/health/` | 🔓 Нет | Проверка работоспособности |

---

### 1.1 Регистрация пользователя

```
POST {{user_url}}/api/users/register/
```

**Headers:**
```
Content-Type: application/json
```

**Body (raw → JSON):**
```json
{
    "email": "ivan@example.com",
    "username": "ivan_ivanov",
    "first_name": "Иван",
    "last_name": "Иванов",
    "password": "SecurePass123!",
    "password_confirm": "SecurePass123!"
}
```

**✅ Ожидаемый ответ (201 Created):**
```json
{
    "id": 1,
    "email": "ivan@example.com",
    "username": "ivan_ivanov",
    "first_name": "Иван",
    "last_name": "Иванов",
    "message": "User registered successfully"
}
```

**Postman Tests (вкладка Scripts → Post-response):**
```javascript
pm.test("Регистрация успешна (201)", function () {
    pm.response.to.have.status(201);
});

pm.test("Ответ содержит ID пользователя", function () {
    const json = pm.response.json();
    pm.expect(json).to.have.property('id');
    pm.environment.set("user_id", json.id);
});
```

---

### 1.2 Авторизация (получение JWT-токенов)

```
POST {{user_url}}/api/auth/login/
```

**Headers:**
```
Content-Type: application/json
```

**Body (raw → JSON):**
```json
{
    "email": "ivan@example.com",
    "password": "SecurePass123!"
}
```

**✅ Ожидаемый ответ (200 OK):**
```json
{
    "access": "eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9...",
    "refresh": "eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9...",
    "user": {
        "id": 1,
        "email": "ivan@example.com",
        "username": "ivan_ivanov",
        "first_name": "Иван",
        "last_name": "Иванов"
    }
}
```

**Postman Tests:**
```javascript
pm.test("Авторизация успешна (200)", function () {
    pm.response.to.have.status(200);
});

pm.test("Получены access и refresh токены", function () {
    const json = pm.response.json();
    pm.expect(json).to.have.property('access');
    pm.expect(json).to.have.property('refresh');
    pm.expect(json).to.have.property('user');

    // ✅ Автоматически сохраняем токены
    pm.environment.set("access_token", json.access);
    pm.environment.set("refresh_token", json.refresh);
    pm.environment.set("user_id", json.user.id);
});
```

> 💡 **Важно:** После выполнения этого запроса токены сохранятся в переменных окружения. Все последующие защищённые запросы будут использовать `{{access_token}}` автоматически.

---

### 1.3 Получение профиля 🔒

```
GET {{user_url}}/api/users/profile/
```

**Headers:**
```
Authorization: Bearer {{access_token}}
Content-Type: application/json
```

> 💡 **Или используйте вкладку Authorization** → Type: `Bearer Token` → Token: `{{access_token}}`

**✅ Ожидаемый ответ (200 OK):**
```json
{
    "id": 1,
    "email": "ivan@example.com",
    "username": "ivan_ivanov",
    "first_name": "Иван",
    "last_name": "Иванов",
    "is_active": true,
    "date_joined": "2026-04-17T12:00:00Z",
    "profile": {
        "phone": "",
        "address": "",
        "date_of_birth": null
    }
}
```

**Postman Tests:**
```javascript
pm.test("Профиль получен (200)", function () {
    pm.response.to.have.status(200);
});

pm.test("Профиль содержит данные пользователя и profile", function () {
    const json = pm.response.json();
    pm.expect(json).to.have.property('email');
    pm.expect(json).to.have.property('profile');
});
```

---

### 1.4 Обновление профиля 🔒

```
PUT {{user_url}}/api/users/profile/update/
```

**Headers:**
```
Authorization: Bearer {{access_token}}
Content-Type: application/json
```

**Body (raw → JSON):**
```json
{
    "phone": "+7-900-123-45-67",
    "address": "г. Москва, ул. Пушкина, д. 10",
    "date_of_birth": "1995-06-15"
}
```

**✅ Ожидаемый ответ (200 OK):**
```json
{
    "phone": "+7-900-123-45-67",
    "address": "г. Москва, ул. Пушкина, д. 10",
    "date_of_birth": "1995-06-15"
}
```

**Postman Tests:**
```javascript
pm.test("Профиль обновлён (200)", function () {
    pm.response.to.have.status(200);
});

pm.test("Данные профиля обновлены корректно", function () {
    const json = pm.response.json();
    pm.expect(json.phone).to.equal("+7-900-123-45-67");
    pm.expect(json.address).to.equal("г. Москва, ул. Пушкина, д. 10");
    pm.expect(json.date_of_birth).to.equal("1995-06-15");
});
```

---

### 1.5 Обновление access-токена

```
POST {{user_url}}/api/auth/refresh/
```

**Headers:**
```
Content-Type: application/json
```

**Body (raw → JSON):**
```json
{
    "refresh": "{{refresh_token}}"
}
```

**✅ Ожидаемый ответ (200 OK):**
```json
{
    "access": "eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9..."
}
```

**Postman Tests:**
```javascript
pm.test("Токен обновлён (200)", function () {
    pm.response.to.have.status(200);
});

pm.test("Новый access-токен получен", function () {
    const json = pm.response.json();
    pm.expect(json).to.have.property('access');
    pm.environment.set("access_token", json.access);
});
```

---

### 1.6 Тесты на ошибки (User Service)

#### ❌ Регистрация с несовпадающими паролями

```
POST {{user_url}}/api/users/register/
```

```json
{
    "email": "wrong@example.com",
    "username": "wronguser",
    "first_name": "Тест",
    "last_name": "Тестов",
    "password": "SecurePass123!",
    "password_confirm": "DifferentPass456!"
}
```

**Ожидаемый ответ (400 Bad Request):**
```json
{
    "password_confirm": ["Passwords do not match"]
}
```

---

#### ❌ Регистрация с уже существующим email

```
POST {{user_url}}/api/users/register/
```

```json
{
    "email": "ivan@example.com",
    "username": "another_user",
    "first_name": "Другой",
    "last_name": "Пользователь",
    "password": "SecurePass123!",
    "password_confirm": "SecurePass123!"
}
```

**Ожидаемый ответ (400 Bad Request):**
```json
{
    "email": ["A user with this email already exists."]
}
```

---

#### ❌ Авторизация с неверными данными

```
POST {{user_url}}/api/auth/login/
```

```json
{
    "email": "nonexistent@example.com",
    "password": "wrongpassword"
}
```

**Ожидаемый ответ (401 Unauthorized):**
```json
{
    "error": "Invalid credentials"
}
```

---

#### ❌ Доступ к профилю без токена

```
GET {{user_url}}/api/users/profile/
```

**Headers:** *(без Authorization)*

**Ожидаемый ответ (401 Unauthorized):**
```json
{
    "detail": "Authentication credentials were not provided."
}
```

---

## 📦 Product Service — порт 8001

> Product Service управляет категориями и продуктами. 
> **Все эндпоинты открыты** (`AllowAny`), токен НЕ нужен.
> Поддерживает фильтрацию, поиск, сортировку и пагинацию (по 20 элементов).

### Структура URL

| Метод | Endpoint | Описание |
|-------|----------|----------|
| `GET` | `/api/categories/` | Список всех категорий |
| `POST` | `/api/categories/` | Создание новой категории |
| `GET` | `/api/categories/{slug}/` | Детали категории (по slug) |
| `PUT` | `/api/categories/{slug}/` | Обновление категории |
| `DELETE` | `/api/categories/{slug}/` | Удаление категории |
| `GET` | `/api/products/` | Список продуктов (с фильтрами) |
| `POST` | `/api/products/` | Создание нового продукта |
| `GET` | `/api/products/{id}/` | Детали продукта |
| `PUT/PATCH` | `/api/products/{id}/` | Обновление продукта |
| `DELETE` | `/api/products/{id}/` | Удаление продукта |
| `POST` | `/api/products/{id}/reserve/` | Резервирование товара |
| `POST` | `/api/products/{id}/release/` | Освобождение резерва |
| `GET` | `/api/products/{id}/check-availability/` | Проверка наличия |
| `GET` | `/health/` | Проверка работоспособности |

---

### 2.1 Создание категории

```
POST {{product_url}}/api/categories/
```

**Headers:**
```
Content-Type: application/json
```

**Body (raw → JSON):**
```json
{
    "name": "Электроника",
    "description": "Гаджеты, смартфоны и электронные устройства"
}
```

**✅ Ожидаемый ответ (201 Created):**
```json
{
    "id": 1,
    "name": "Электроника",
    "slug": "elektronika",
    "description": "Гаджеты, смартфоны и электронные устройства",
    "products_count": 0,
    "created_at": "2026-04-17T12:00:00Z"
}
```

**Postman Tests:**
```javascript
pm.test("Категория создана (201)", function () {
    pm.response.to.have.status(201);
});

pm.test("Сохраняем ID и slug категории", function () {
    const json = pm.response.json();
    pm.expect(json).to.have.property('id');
    pm.expect(json).to.have.property('slug');
    pm.environment.set("category_id", json.id);
    pm.environment.set("category_slug", json.slug);
});
```

---

### 2.2 Список категорий

```
GET {{product_url}}/api/categories/
```

**✅ Ожидаемый ответ (200 OK):**
```json
[
    {
        "id": 1,
        "name": "Электроника",
        "slug": "elektronika",
        "description": "Гаджеты, смартфоны и электронные устройства",
        "products_count": 0,
        "created_at": "2026-04-17T12:00:00Z"
    }
]
```

---

### 2.3 Детали категории (по slug)

```
GET {{product_url}}/api/categories/{{category_slug}}/
```

**✅ Ожидаемый ответ (200 OK):**
```json
{
    "id": 1,
    "name": "Электроника",
    "slug": "elektronika",
    "description": "Гаджеты, смартфоны и электронные устройства",
    "products_count": 0,
    "created_at": "2026-04-17T12:00:00Z"
}
```

---

### 2.4 Обновление категории

```
PUT {{product_url}}/api/categories/{{category_slug}}/
```

**Body (raw → JSON):**
```json
{
    "name": "Электроника и гаджеты",
    "description": "Обновлённое описание: смартфоны, планшеты, ноутбуки"
}
```

---

### 2.5 Создание продукта

```
POST {{product_url}}/api/products/
```

**Headers:**
```
Content-Type: application/json
```

**Body (raw → JSON):**
```json
{
    "name": "iPhone 15 Pro",
    "description": "Новый iPhone 15 Pro с процессором A17 Pro",
    "price": "129990.00",
    "category": {{category_id}},
    "stock_quantity": 50,
    "image_url": "https://example.com/iphone15pro.jpg"
}
```

**✅ Ожидаемый ответ (201 Created):**
```json
{
    "id": 1,
    "name": "iPhone 15 Pro",
    "description": "Новый iPhone 15 Pro с процессором A17 Pro",
    "price": "129990.00",
    "category": 1,
    "category_name": "Электроника",
    "stock_quantity": 50,
    "image_url": "https://example.com/iphone15pro.jpg",
    "is_active": true,
    "is_in_stock": true,
    "created_at": "2026-04-17T12:00:00Z",
    "updated_at": "2026-04-17T12:00:00Z"
}
```

**Postman Tests:**
```javascript
pm.test("Продукт создан (201)", function () {
    pm.response.to.have.status(201);
});

pm.test("Сохраняем ID продукта", function () {
    const json = pm.response.json();
    pm.expect(json).to.have.property('id');
    pm.environment.set("product_id", json.id);
});
```

> 💡 **Создайте ещё несколько продуктов** для тестирования фильтрации:

**Продукт 2:**
```json
{
    "name": "Samsung Galaxy S24",
    "description": "Флагман Samsung с AI-фичами",
    "price": "89990.00",
    "category": {{category_id}},
    "stock_quantity": 30,
    "image_url": "https://example.com/samsung-s24.jpg"
}
```

**Продукт 3 (нет в наличии):**
```json
{
    "name": "Google Pixel 8",
    "description": "Google Pixel 8 с чистым Android",
    "price": "59990.00",
    "category": {{category_id}},
    "stock_quantity": 0,
    "image_url": "https://example.com/pixel8.jpg"
}
```

---

### 2.6 Список продуктов (с фильтрами)

#### Базовый список
```
GET {{product_url}}/api/products/
```

#### Поиск по имени/описанию
```
GET {{product_url}}/api/products/?search=iPhone
```

#### Фильтр по диапазону цен
```
GET {{product_url}}/api/products/?min_price=50000&max_price=100000
```

#### Только товары в наличии
```
GET {{product_url}}/api/products/?in_stock=true
```

#### По ID категории
```
GET {{product_url}}/api/products/?category={{category_id}}
```

#### Сортировка по цене (от дешёвого к дорогому)
```
GET {{product_url}}/api/products/?ordering=price
```

#### Сортировка по цене (от дорогого к дешёвому)
```
GET {{product_url}}/api/products/?ordering=-price
```

#### Комбинированный запрос
```
GET {{product_url}}/api/products/?category={{category_id}}&min_price=50000&in_stock=true&ordering=price
```

**✅ Формат ответа (с пагинацией):**
```json
{
    "count": 2,
    "next": null,
    "previous": null,
    "results": [
        {
            "id": 2,
            "name": "Samsung Galaxy S24",
            "price": "89990.00",
            "is_in_stock": true,
            ...
        },
        {
            "id": 1,
            "name": "iPhone 15 Pro",
            "price": "129990.00",
            "is_in_stock": true,
            ...
        }
    ]
}
```

> 📄 **Пагинация:** По 20 элементов на страницу. Для навигации: `?page=2`

---

### 2.7 Детали продукта

```
GET {{product_url}}/api/products/{{product_id}}/
```

**✅ Ответ содержит полную информацию о категории (вложенный объект):**
```json
{
    "id": 1,
    "name": "iPhone 15 Pro",
    "description": "Новый iPhone 15 Pro с процессором A17 Pro",
    "price": "129990.00",
    "category": {
        "id": 1,
        "name": "Электроника",
        "slug": "elektronika",
        "description": "Гаджеты, смартфоны и электронные устройства",
        "products_count": 3,
        "created_at": "2026-04-17T12:00:00Z"
    },
    "stock_quantity": 50,
    "image_url": "https://example.com/iphone15pro.jpg",
    "is_active": true,
    "is_in_stock": true,
    "created_at": "2026-04-17T12:00:00Z",
    "updated_at": "2026-04-17T12:00:00Z"
}
```

---

### 2.8 Обновление продукта (частичное)

```
PATCH {{product_url}}/api/products/{{product_id}}/
```

**Body (raw → JSON):**
```json
{
    "price": "119990.00",
    "stock_quantity": 45
}
```

---

### 2.9 Резервирование товара

```
POST {{product_url}}/api/products/{{product_id}}/reserve/
```

**Body (raw → JSON):**
```json
{
    "quantity": 2
}
```

**✅ Ожидаемый ответ (200 OK):**
```json
{
    "success": true,
    "message": "Reserved 2 units of iPhone 15 Pro",
    "remaining_stock": 48
}
```

**❌ Если товара недостаточно (400 Bad Request):**
```json
{
    "success": false,
    "message": "Insufficient stock",
    "available_stock": 48
}
```

---

### 2.10 Освобождение резерва

```
POST {{product_url}}/api/products/{{product_id}}/release/
```

**Body (raw → JSON):**
```json
{
    "quantity": 2
}
```

**✅ Ожидаемый ответ (200 OK):**
```json
{
    "success": true,
    "message": "Released 2 units of iPhone 15 Pro",
    "current_stock": 50
}
```

---

### 2.11 Проверка наличия

```
GET {{product_url}}/api/products/{{product_id}}/check-availability/?quantity=5
```

**✅ Ожидаемый ответ (200 OK):**
```json
{
    "product_id": 1,
    "name": "iPhone 15 Pro",
    "price": "129990.00",
    "available": true,
    "stock_quantity": 50,
    "requested_quantity": 5
}
```

---

### 2.12 Удаление продукта

```
DELETE {{product_url}}/api/products/{{product_id}}/
```

**✅ Ожидаемый ответ:** `204 No Content` (пустое тело)

---

### 2.13 Удаление категории

```
DELETE {{product_url}}/api/categories/{{category_slug}}/
```

**✅ Ожидаемый ответ:** `204 No Content`

> ⚠️ **Внимание:** Удаление категории каскадно удалит все связанные продукты!

---

## 🛒 Cart Service — порт 8002

> Cart Service управляет корзиной покупок. 
> ⚠️ **Все эндпоинты требуют JWT-токен** (через middleware → User Service).
> Cart Service обращается к Product Service для проверки наличия товаров.

### Как работает аутентификация в Cart Service

```
┌──────────┐    Authorization: Bearer <token>    ┌──────────────┐
│  Postman  │ ────────────────────────────────►  │ Cart Service │
│           │                                    │  Middleware   │
└──────────┘                                     └──────┬───────┘
                                                        │
                                    Пересылает токен    │
                                    для валидации       │
                                                        ▼
                                                 ┌──────────────┐
                                                 │ User Service │
                                                 │ GET /profile/│
                                                 └──────┬───────┘
                                                        │
                                              Возвращает│
                                              user_id   │
                                                        ▼
                                                 ┌──────────────┐
                                                 │ Cart Service │
                                                 │  Обработка   │
                                                 │  запроса     │
                                                 └──────────────┘
```

> 💡 Cart Service **НЕ валидирует JWT сам**. Он пересылает токен в User Service (`GET /api/users/profile/`) и по ответу получает `user_id`.

### Структура URL

| Метод | Endpoint | Описание |
|-------|----------|----------|
| `GET` | `/api/cart/` | Получение корзины пользователя |
| `POST` | `/api/cart/add/` | Добавление товара в корзину |
| `PUT` | `/api/cart/update/{item_id}/` | Обновление количества в корзине |
| `DELETE` | `/api/cart/remove/{item_id}/` | Удаление товара из корзины |
| `DELETE` | `/api/cart/clear/` | Очистка всей корзины |
| `GET` | `/api/cart/summary/` | Краткая сводка по корзине |
| `GET` | `/health/` | Проверка работоспособности |

> 🔒 **Все запросы (кроме `/health/`) требуют заголовок** `Authorization: Bearer {{access_token}}`

---

### 3.1 Получение корзины 🔒

```
GET {{cart_url}}/api/cart/
```

**Headers:**
```
Authorization: Bearer {{access_token}}
Content-Type: application/json
```

**✅ Ожидаемый ответ (200 OK) — пустая корзина:**
```json
{
    "id": 1,
    "user_id": 1,
    "items": [],
    "total_amount": "0.00",
    "total_items": 0,
    "created_at": "2026-04-17T12:00:00Z",
    "updated_at": "2026-04-17T12:00:00Z"
}
```

**Postman Tests:**
```javascript
pm.test("Корзина получена (200)", function () {
    pm.response.to.have.status(200);
});

pm.test("Корзина содержит нужные поля", function () {
    const json = pm.response.json();
    pm.expect(json).to.have.property('items');
    pm.expect(json).to.have.property('total_amount');
    pm.expect(json).to.have.property('total_items');
});
```

---

### 3.2 Добавление товара в корзину 🔒

> ⚠️ **Предусловие:** Продукт с `product_id` должен существовать в Product Service!

```
POST {{cart_url}}/api/cart/add/
```

**Headers:**
```
Authorization: Bearer {{access_token}}
Content-Type: application/json
```

**Body (raw → JSON):**
```json
{
    "product_id": {{product_id}},
    "quantity": 2
}
```

**✅ Ожидаемый ответ (201 Created):**
```json
{
    "message": "Product added to cart successfully",
    "cart_item": {
        "id": 1,
        "product_id": 1,
        "product_name": "iPhone 15 Pro",
        "quantity": 2,
        "price": "129990.00",
        "subtotal": "259980.00",
        "product_info": {
            "name": "iPhone 15 Pro",
            "current_price": "129990.00",
            "image_url": "https://example.com/iphone15pro.jpg",
            "is_active": true,
            "stock_quantity": 50
        },
        "created_at": "2026-04-17T12:00:00Z"
    }
}
```

**Postman Tests:**
```javascript
pm.test("Товар добавлен в корзину (201)", function () {
    pm.response.to.have.status(201);
});

pm.test("Сохраняем ID элемента корзины", function () {
    const json = pm.response.json();
    pm.expect(json.cart_item).to.have.property('id');
    pm.environment.set("cart_item_id", json.cart_item.id);
});
```

---

### 3.3 Обновление количества товара в корзине 🔒

```
PUT {{cart_url}}/api/cart/update/{{cart_item_id}}/
```

**Headers:**
```
Authorization: Bearer {{access_token}}
Content-Type: application/json
```

**Body (raw → JSON):**
```json
{
    "quantity": 5
}
```

**✅ Ожидаемый ответ (200 OK):**
```json
{
    "message": "Cart item updated successfully",
    "cart_item": {
        "id": 1,
        "product_id": 1,
        "product_name": "iPhone 15 Pro",
        "quantity": 5,
        "price": "129990.00",
        "subtotal": "649950.00",
        ...
    }
}
```

---

### 3.4 Краткая сводка по корзине 🔒

```
GET {{cart_url}}/api/cart/summary/
```

**Headers:**
```
Authorization: Bearer {{access_token}}
```

**✅ Ожидаемый ответ (200 OK):**
```json
{
    "total_items": 5,
    "total_amount": "649950.00",
    "items_count": 1
}
```

> 💡 `total_items` — сумма всех quantity, `items_count` — количество уникальных товаров.

---

### 3.5 Удаление товара из корзины 🔒

```
DELETE {{cart_url}}/api/cart/remove/{{cart_item_id}}/
```

**Headers:**
```
Authorization: Bearer {{access_token}}
```

**✅ Ожидаемый ответ (204 No Content):**
```json
{
    "message": "Item removed from cart successfully"
}
```

---

### 3.6 Очистка всей корзины 🔒

```
DELETE {{cart_url}}/api/cart/clear/
```

**Headers:**
```
Authorization: Bearer {{access_token}}
```

**✅ Ожидаемый ответ (204 No Content):**
```json
{
    "message": "Cart cleared successfully"
}
```

---

### 3.7 Тесты на ошибки (Cart Service)

#### ❌ Запрос без токена

```
GET {{cart_url}}/api/cart/
```
*(без заголовка Authorization)*

**Ожидаемый ответ (401):**
```json
{
    "error": "Authentication required",
    "message": "Authorization header with Bearer token is required"
}
```

#### ❌ Добавление несуществующего товара

```
POST {{cart_url}}/api/cart/add/
```

```json
{
    "product_id": 99999,
    "quantity": 1
}
```

**Ожидаемый ответ (400):**
```json
{
    "product_id": ["Product not found"]
}
```

#### ❌ Добавление товара с количеством больше stock

```
POST {{cart_url}}/api/cart/add/
```

```json
{
    "product_id": {{product_id}},
    "quantity": 99999
}
```

**Ожидаемый ответ (400):**
```json
{
    "error": "Product is not available in requested quantity"
}
```

---

## 🔗 Межсервисное тестирование

> Самая важная часть! Здесь мы тестируем **как сервисы работают вместе**.

### Сценарий 1: Полный цикл покупки

Выполняйте запросы **строго по порядку**:

```
┌──────────────────────────────────────────────────────────────┐
│  Шаг 1: User Service — Регистрация                          │
│  POST {{user_url}}/api/users/register/                       │
│                                                              │
│  Шаг 2: User Service — Авторизация → получить токен          │
│  POST {{user_url}}/api/auth/login/                           │
│                                                              │
│  Шаг 3: Product Service — Создать категорию                  │
│  POST {{product_url}}/api/categories/                        │
│                                                              │
│  Шаг 4: Product Service — Создать товар                      │
│  POST {{product_url}}/api/products/                          │
│                                                              │
│  Шаг 5: Product Service — Проверить наличие                  │
│  GET {{product_url}}/api/products/{id}/check-availability/   │
│                                                              │
│  Шаг 6: Cart Service — Добавить товар в корзину              │
│  POST {{cart_url}}/api/cart/add/  (с JWT-токеном!)            │
│  → Cart обращается к User Service для валидации токена        │
│  → Cart обращается к Product Service для проверки наличия     │
│                                                              │
│  Шаг 7: Cart Service — Посмотреть корзину                    │
│  GET {{cart_url}}/api/cart/                                   │
│                                                              │
│  Шаг 8: Cart Service — Получить сводку                       │
│  GET {{cart_url}}/api/cart/summary/                           │
│                                                              │
│  Шаг 9: Product Service — Зарезервировать товар              │
│  POST {{product_url}}/api/products/{id}/reserve/             │
│                                                              │
│  Шаг 10: Product Service — Проверить остаток после резерва   │
│  GET {{product_url}}/api/products/{id}/check-availability/   │
└──────────────────────────────────────────────────────────────┘
```

---

### Сценарий 2: Cart → Product Service взаимодействие

**Цель:** Проверить, что Cart Service корректно проверяет наличие товаров через Product Service.

#### Шаг 1. Создайте товар с ограниченным количеством

```
POST {{product_url}}/api/products/
```
```json
{
    "name": "Limited Edition Watch",
    "description": "Лимитированные часы",
    "price": "50000.00",
    "category": {{category_id}},
    "stock_quantity": 3,
    "image_url": "https://example.com/watch.jpg"
}
```

*Запомните ID (например, `product_id = 4`).*

#### Шаг 2. Добавьте 2 шт. в корзину (должно пройти ✅)

```
POST {{cart_url}}/api/cart/add/
```
```json
{
    "product_id": 4,
    "quantity": 2
}
```

**Ожидание:** `201 Created` — товар добавлен.

#### Шаг 3. Попробуйте добавить ещё 2 шт. (должно отказать ❌)

```
POST {{cart_url}}/api/cart/add/
```
```json
{
    "product_id": 4,
    "quantity": 2
}
```

**Ожидание:** `400 Bad Request` — `"Not enough stock available"` (уже 2 в корзине + 2 = 4 > 3 на складе).

---

### Сценарий 3: Cart → User Service взаимодействие

**Цель:** Проверить, что Cart Service правильно идентифицирует пользователя через User Service.

#### Шаг 1. Зарегистрируйте двух пользователей

**Пользователь A:**
```json
{
    "email": "alice@example.com",
    "username": "alice",
    "first_name": "Алиса",
    "last_name": "Алисова",
    "password": "AlicePass123!",
    "password_confirm": "AlicePass123!"
}
```

**Пользователь B:**
```json
{
    "email": "bob@example.com",
    "username": "bob",
    "first_name": "Борис",
    "last_name": "Борисов",
    "password": "BobPass123!",
    "password_confirm": "BobPass123!"
}
```

#### Шаг 2. Авторизуйтесь как Алиса, добавьте товар

1. `POST /api/auth/login/` → получите `access_token_alice`
2. `POST /api/cart/add/` с токеном Алисы → добавьте iPhone

#### Шаг 3. Авторизуйтесь как Борис, проверьте что его корзина пуста

1. `POST /api/auth/login/` → получите `access_token_bob`
2. `GET /api/cart/` с токеном Бориса → корзина должна быть **пуста**

**Ожидание:** Каждый пользователь видит только свою корзину. Корзины изолированы по `user_id`.

---

### Сценарий 4: Резервирование и остаток товара

**Цель:** Убедиться, что резервирование через Product Service изменяет доступное количество.

```
1. POST {{product_url}}/api/products/    → stock_quantity = 10
2. GET  {{product_url}}/api/products/{id}/check-availability/?quantity=10  → available: true
3. POST {{product_url}}/api/products/{id}/reserve/   → {"quantity": 7}
4. GET  {{product_url}}/api/products/{id}/check-availability/?quantity=5   → available: false (осталось 3)
5. POST {{product_url}}/api/products/{id}/release/   → {"quantity": 7}
6. GET  {{product_url}}/api/products/{id}/check-availability/?quantity=10  → available: true (снова 10)
```

---

### Сценарий 5: Обновление токена и продолжение работы

**Цель:** Проверить, что после обновления access-токена Cart Service продолжает работать.

```
1. POST {{user_url}}/api/auth/login/     → access + refresh
2. GET  {{cart_url}}/api/cart/            → ✅ работает
3. (ждём 60 минут или имитируем истечение)
4. POST {{user_url}}/api/auth/refresh/   → новый access
5. GET  {{cart_url}}/api/cart/            → ✅ работает с новым токеном
```

---

### Сценарий 6: Что случится при выключении сервиса?

**Цель:** Проверить устойчивость к отказам (graceful degradation).

#### Тест A: Product Service недоступен → Cart Service

1. Остановите Product Service (Ctrl+C в терминале)
2. Попробуйте добавить товар в корзину:
   ```
   POST {{cart_url}}/api/cart/add/
   ```
   ```json
   { "product_id": 1, "quantity": 1 }
   ```
3. **Ожидание:** `400` или `500` — Cart Service не может проверить наличие

#### Тест B: User Service недоступен → Cart Service

1. Остановите User Service
2. Попробуйте получить корзину:
   ```
   GET {{cart_url}}/api/cart/
   ```
3. **Ожидание:** `401` — middleware не может валидировать токен

---

## 🏃 Автоматизация через Collection Runner

### Организация коллекции в Postman

Создайте коллекцию `Microservices Shop` со следующей структурой папок:

```
📁 Microservices Shop
├── 📁 01. Setup
│   ├── Health Check — User Service
│   ├── Health Check — Product Service
│   └── Health Check — Cart Service
├── 📁 02. User Service
│   ├── Регистрация пользователя
│   ├── Авторизация
│   ├── Получение профиля
│   ├── Обновление профиля
│   └── Обновление токена
├── 📁 03. Product Service
│   ├── Создание категории
│   ├── Список категорий
│   ├── Создание продукта
│   ├── Список продуктов
│   ├── Детали продукта
│   ├── Фильтрация продуктов
│   ├── Резервирование товара
│   ├── Освобождение резерва
│   └── Проверка наличия
├── 📁 04. Cart Service  
│   ├── Получение корзины
│   ├── Добавление товара
│   ├── Обновление количества
│   ├── Сводка по корзине
│   ├── Удаление товара
│   └── Очистка корзины
├── 📁 05. Inter-Service Tests
│   ├── Полный цикл покупки
│   ├── Cart → Product (проверка наличия)
│   └── Cart → User (изоляция корзин)
└── 📁 06. Error Cases
    ├── Регистрация — дублирование email
    ├── Авторизация — неверные данные
    ├── Профиль — без токена
    ├── Корзина — несуществующий товар
    └── Корзина — превышение stock
```

### Pre-request Script (уровень коллекции)

Добавьте в **Pre-request Script** на уровне коллекции для автоматической генерации данных:

```javascript
// Генерация случайных данных для каждого прогона
if (!pm.environment.get("test_email")) {
    const randomId = Math.floor(Math.random() * 100000);
    pm.environment.set("test_email", `user_${randomId}@example.com`);
    pm.environment.set("test_username", `user_${randomId}`);
    pm.environment.set("test_password", "SecurePass123!");
}
```

### Запуск Collection Runner

1. Нажмите **▶ Run collection** рядом с коллекцией
2. Выберите окружение `Microservices Shop — Dev`
3. Настройте:
   - Delay: `200ms` (задержка между запросами)
   - Iterations: `1`
4. Нажмите **Run Microservices Shop**

---

## 🛠 Устранение типичных ошибок

### ❌ `Connection refused` / `Could not get any response`

**Причина:** Сервис не запущен.
**Решение:** Проверьте что все 3 терминала работают и нет ошибок.

```bash
# Проверка портов
lsof -i :8000  # User Service
lsof -i :8001  # Product Service
lsof -i :8002  # Cart Service
```

---

### ❌ `401 Unauthorized` в Cart Service

**Причина 1:** Токен истёк (access живёт 60 минут).
**Решение:** Обновите токен:
```
POST {{user_url}}/api/auth/refresh/
```

**Причина 2:** Cart Service не может связаться с User Service.
**Решение:** Проверьте `USER_SERVICE_URL` в `cart-services/config/settings.py`:
```python
# Должно быть:
USER_SERVICE_URL = 'http://localhost:8000'

# ⚠️ Сейчас в коде установлено:
USER_SERVICE_URL = 'http://localhost:8004'  # ← НУЖНО ИСПРАВИТЬ!
```

---

### ❌ `CSRF verification failed`

**Причина:** Django CSRF-защита блокирует POST-запросы.
**Решение:** CSRF не должен влиять на API через DRF, но если проблема появляется:
```python
# В settings.py добавьте:
CSRF_TRUSTED_ORIGINS = ['http://localhost:3000']
```

---

### ❌ `"Product not found"` при добавлении в корзину

**Причина:** Cart Service не может получить продукт из Product Service.
**Решение:**
1. Убедитесь что Product Service запущен на порту `8001`
2. Проверьте `PRODUCT_SERVICE_URL` в настройках Cart Service:
   ```python
   PRODUCT_SERVICE_URL = 'http://localhost:8001'
   ```
3. Убедитесь что продукт существует: `GET http://localhost:8001/api/products/{id}/`

---

### ❌ `"Authentication credentials were not provided"` в User Service

**Причина:** Не передан заголовок `Authorization` для защищённого эндпоинта.
**Решение:** На вкладке **Authorization** → выберите **Bearer Token** → вставьте `{{access_token}}`

---

### ❌ Пустой ответ / `500 Internal Server Error`

**Решение:** Проверьте логи в терминале соответствующего сервиса. Django выводит traceback ошибок прямо в консоль.

---

## 📊 Сводная таблица всех эндпоинтов

| # | Сервис | Метод | Endpoint | Auth | Описание |
|---|--------|-------|----------|------|----------|
| 1 | User | `GET` | `/health/` | ❌ | Health check |
| 2 | User | `POST` | `/api/users/register/` | ❌ | Регистрация |
| 3 | User | `POST` | `/api/auth/login/` | ❌ | Авторизация |
| 4 | User | `POST` | `/api/auth/refresh/` | ❌ | Обновление токена |
| 5 | User | `GET` | `/api/users/profile/` | ✅ JWT | Получение профиля |
| 6 | User | `PUT` | `/api/users/profile/update/` | ✅ JWT | Обновление профиля |
| 7 | Product | `GET` | `/health/` | ❌ | Health check |
| 8 | Product | `GET` | `/api/categories/` | ❌ | Список категорий |
| 9 | Product | `POST` | `/api/categories/` | ❌ | Создание категории |
| 10 | Product | `GET` | `/api/categories/{slug}/` | ❌ | Детали категории |
| 11 | Product | `PUT` | `/api/categories/{slug}/` | ❌ | Обновление категории |
| 12 | Product | `DELETE` | `/api/categories/{slug}/` | ❌ | Удаление категории |
| 13 | Product | `GET` | `/api/products/` | ❌ | Список продуктов |
| 14 | Product | `POST` | `/api/products/` | ❌ | Создание продукта |
| 15 | Product | `GET` | `/api/products/{id}/` | ❌ | Детали продукта |
| 16 | Product | `PUT/PATCH` | `/api/products/{id}/` | ❌ | Обновление продукта |
| 17 | Product | `DELETE` | `/api/products/{id}/` | ❌ | Удаление продукта |
| 18 | Product | `POST` | `/api/products/{id}/reserve/` | ❌ | Резервирование |
| 19 | Product | `POST` | `/api/products/{id}/release/` | ❌ | Освобождение резерва |
| 20 | Product | `GET` | `/api/products/{id}/check-availability/` | ❌ | Проверка наличия |
| 21 | Cart | `GET` | `/health/` | ❌ | Health check |
| 22 | Cart | `GET` | `/api/cart/` | ✅ JWT | Получение корзины |
| 23 | Cart | `POST` | `/api/cart/add/` | ✅ JWT | Добавление товара |
| 24 | Cart | `PUT` | `/api/cart/update/{item_id}/` | ✅ JWT | Обновление количества |
| 25 | Cart | `DELETE` | `/api/cart/remove/{item_id}/` | ✅ JWT | Удаление из корзины |
| 26 | Cart | `DELETE` | `/api/cart/clear/` | ✅ JWT | Очистка корзины |
| 27 | Cart | `GET` | `/api/cart/summary/` | ✅ JWT | Сводка по корзине |

---

## ✅ Чеклист тестирования

### Основные проверки

- [ ] Все 3 сервиса запущены и отвечают на health check
- [ ] User Service: регистрация → логин → профиль → обновление профиля
- [ ] Product Service: категория → продукт → фильтрация → резервирование
- [ ] Cart Service: добавление → обновление → сводка → удаление → очистка

### Межсервисные проверки

- [ ] Cart Service валидирует JWT через User Service
- [ ] Cart Service проверяет наличие через Product Service
- [ ] Корзины изолированы для разных пользователей
- [ ] Резервирование и освобождение корректно изменяют stock

### Проверки на ошибки

- [ ] Дублирование email при регистрации → 400
- [ ] Неверные учётные данные → 401
- [ ] Запрос без токена к защищённому endpoint → 401
- [ ] Несуществующий товар в корзине → 400/404
- [ ] Превышение stock при добавлении в корзину → 400
- [ ] Недоступный Product Service → корректная ошибка
- [ ] Недоступный User Service → 401
