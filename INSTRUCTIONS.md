# Microservices Shop — Инструкция по запуску и тестированию

## 🏗 Архитектура

```
microservices-shop/
├── databases/
│   ├── product.db      ← БД продуктов (SQLite)
│   └── user.db         ← БД пользователей (SQLite)
├── services/
│   ├── product-service/ ← Сервис продуктов (порт 8001)
│   └── user-services/   ← Сервис пользователей (порт 8000)
```

Два **независимых** Django-проекта, каждый со своей БД и `venv`.

---

## 🚀 Запуск

### Терминал 1 — User Service (порт 8000)

```bash
cd services/user-services
source venv/bin/activate
pip install -r requirements.txt    # только первый раз
python manage.py migrate           # только первый раз
python manage.py createsuperuser   # только первый раз
python manage.py runserver 0.0.0.0:8000
```

### Терминал 2 — Product Service (порт 8001)

```bash
cd services/product-service
source venv/bin/activate
pip install -r requirements.txt    # только первый раз
python manage.py migrate           # только первый раз
python manage.py createsuperuser   # только первый раз
python manage.py runserver 0.0.0.0:8001
```

> ⚠️ Сервисы обязательно на **разных портах**! User → `8000`, Product → `8001`.

---

## 👤 Модель пользователя

**User Service** использует **кастомную модель** `User` (наследуется от `AbstractUser`):
- Логин по `email` (не по username)
- Поля: email, username, first_name, last_name
- Есть `UserProfile` (phone, address, date_of_birth) — создаётся отдельно

**Product Service** — своей модели юзера **нет**. Используется стандартная Django `auth.User` только для админки.

---

## 🔑 Админ-панель

| Сервис | URL | Что внутри |
|---|---|---|
| User Service | http://localhost:8000/admin/ | Пользователи + профили |
| Product Service | http://localhost:8001/admin/ | Категории + продукты |

Для каждого сервиса нужен **свой суперпользователь** (разные БД!).

---

## 🔐 Нужен ли токен для создания продуктов/категорий?

**Сейчас — НЕТ.** В `product-service` стоит `AllowAny`:

```python
# product-service/config/settings.py
REST_FRAMEWORK = {
    'DEFAULT_PERMISSION_CLASSES': [
        'rest_framework.permissions.AllowAny',  # ← любой может создавать
    ],
}
```

Все эндпоинты продуктов **открыты** — создавать, редактировать и удалять может кто угодно без токена.

Для **user-service** токен нужен только для `profile/` и `profile/update/` (там стоит JWT-аутентификация).

---

## 📨 Как передавать токен в Postman

Два способа — **результат одинаковый**:

### Способ 1: Вкладка Authorization (рекомендуется ✅)
1. Открой вкладку **Authorization**
2. Type → выбери **Bearer Token**
3. Вставь токен в поле **Token**
4. Postman сам добавит заголовок `Authorization: Bearer <token>`

### Способ 2: Вкладка Headers (вручную)
1. Открой вкладку **Headers**
2. Key: `Authorization`
3. Value: `Bearer eyJ0eXAiOiJKV1Qi...` (слово Bearer + пробел + токен)

> 💡 Оба способа отправляют **один и тот же** HTTP-заголовок. `Authorization` tab — просто удобнее.

---

## 📮 Все запросы для Postman

### User Service — `http://localhost:8000`

---

#### 1. Регистрация

**POST** `http://localhost:8000/api/users/register/`

```json
{
    "email": "test@example.com",
    "username": "testuser",
    "first_name": "Иван",
    "last_name": "Иванов",
    "password": "SecurePass123!",
    "password_confirm": "SecurePass123!"
}
```

---

#### 2. Логин (получение токенов)

**POST** `http://localhost:8000/api/auth/login/`

```json
{
    "email": "test@example.com",
    "password": "SecurePass123!"
}
```

Ответ:
```json
{
    "access": "eyJ0eXAi...",   ← этот токен вставляй в Authorization
    "refresh": "eyJ0eXAi...",  ← для обновления access-токена
    "user": { "id": 1, "email": "test@example.com", ... }
}
```

---

#### 3. Получить профиль 🔒

**GET** `http://localhost:8000/api/users/profile/`

Authorization: `Bearer <access_token>`

---

#### 4. Обновить профиль 🔒

**PUT** `http://localhost:8000/api/users/profile/update/`

Authorization: `Bearer <access_token>`

```json
{
    "phone": "+7-900-123-45-67",
    "address": "г. Москва, ул. Пушкина, д. 10",
    "date_of_birth": "1990-01-15"
}
```

---

#### 5. Обновить токен

**POST** `http://localhost:8000/api/auth/refresh/`

```json
{
    "refresh": "<refresh_token>"
}
```

---

### Product Service — `http://localhost:8001`

> 🔓 Все эндпоинты продуктов открыты (`AllowAny`), токен НЕ нужен.

---

#### 1. Список категорий

**GET** `http://localhost:8001/api/categories/`

---

#### 2. Создать категорию

**POST** `http://localhost:8001/api/categories/`

```json
{
    "name": "Электроника",
    "description": "Гаджеты и электроника"
}
```

---

#### 3. Детали/обновление/удаление категории

- **GET** `http://localhost:8001/api/categories/elektronika/` (по slug)
- **PUT** `http://localhost:8001/api/categories/elektronika/`
- **DELETE** `http://localhost:8001/api/categories/elektronika/`

---

#### 4. Список продуктов

**GET** `http://localhost:8001/api/products/`

Фильтры (query params):
- `?search=iPhone` — поиск по имени/описанию
- `?min_price=100&max_price=5000` — диапазон цен
- `?in_stock=true` — только в наличии
- `?category=1` — по ID категории
- `?ordering=price` / `?ordering=-price` — сортировка

---

#### 5. Создать продукт

**POST** `http://localhost:8001/api/products/`

```json
{
    "name": "iPhone 15",
    "description": "Новый iPhone",
    "price": "99990.00",
    "category": 1,
    "stock_quantity": 50,
    "image_url": "https://example.com/iphone.jpg"
}
```

---

#### 6. Детали/обновление/удаление продукта

- **GET** `http://localhost:8001/api/products/1/`
- **PUT** `http://localhost:8001/api/products/1/`
- **DELETE** `http://localhost:8001/api/products/1/`

---

#### 7. Резервирование товара

**POST** `http://localhost:8001/api/products/1/reserve/`

```json
{ "quantity": 2 }
```

---

#### 8. Освобождение товара

**POST** `http://localhost:8001/api/products/1/release/`

```json
{ "quantity": 2 }
```

---

#### 9. Проверка наличия

**GET** `http://localhost:8001/api/products/1/check-availability/?quantity=5`

---

#### 10. Health Check

**GET** `http://localhost:8001/health/`

Ответ: `{"status": "healthy", "service": "product-service"}`

---

## ⚡ Порядок тестирования

1. Запустить оба сервиса
2. **POST** регистрация → **POST** логин → скопировать `access` токен
3. С токеном: **GET** профиль, **PUT** обновить профиль
4. Без токена: создать категорию → создать продукт → проверить список
5. Попробовать резервирование/освобождение товара
