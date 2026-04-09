# GIKSAN - Оптовый интернет-магазин сантехники

Платформа для оптовой продажи смесителей и отопительного оборудования с интеграцией Telegram-бота для управления каталогом и поддержки клиентов.

## Оглавление

- [Описание](#описание)
- [Архитектура](#архитектура)
- [Структура проекта](#структура-проекта)
- [Технологический стек](#технологический-стек)
- [Функциональность](#функциональность)
- [API документация](#api-документация)
- [Методы защиты](#методы-защиты)
- [Развертывание](#развертывание)
- [Конфигурация](#конфигурация)

---

## Описание

GIKSAN -- полнофункциональная веб-платформа для B2B-продаж сантехники, включающая каталог продукции с фильтрацией, систему авторизации, оформление заказов и систему live-чата между клиентами и администратором через Telegram-бота.

### Основные возможности

- Каталог товаров с фильтрацией, поиском и сортировкой
- Регистрация и авторизация (email/password + Google OAuth)
- Корзина и оформление заказов
- Система live-чата клиент-администратор
- Telegram-бот для управления каталогом (импорт из файлов) и ответов на заказы
- Адаптивный дизайн
- Кэширование на клиенте и сервере

---

## Архитектура

```
Клиент (Browser)
    |
    | HTTPS
    v
Nginx (Reverse Proxy + SSL)
    |
    | HTTP (localhost:5000)
    v
Gunicorn (WSGI Server)
    |
    +-- Flask Application (main.py)
    |       |
    |       +-- Supabase (Database)
    |       +-- Telegram Bot API
    |
    +-- Telegram Bot (telegram_bot.py, отдельный процесс)
            |
            +-- Supabase (Database)
            +-- Telegram Bot API
```

### Компоненты

| Компонент | Назначение | Порт |
|-----------|------------|------|
| Nginx | Reverse proxy, SSL-терминация, отдача статики | 80, 443 |
| Gunicorn | WSGI-сервер для Flask | 5000 (localhost) |
| Flask (main.py) | Основной бэкенд: API, авторизация, заказы | -- |
| Telegram Bot (telegram_bot.py) | Бот для админа: импорт товаров, ответы на заказы | -- |

---

## Структура проекта

```
/var/www/giksan/
|-- main.py                 # Flask-приложение (бэкенд API)
|-- telegram_bot.py         # Telegram-бот (отдельный systemd-сервис)
|-- web.html                # Главная страница (SPA)
|-- web.css                 # Стили сайта
|-- web.js                  # Клиентская логика (каталог, корзина, чат)
|-- chat-polling.js         # Polling сообщений чата (клиентская часть)
|-- products.json           # Каталог товаров (генерируется ботом)
|-- .env                    # Переменные окружения (секреты)
|-- requirements.txt        # Python-зависимости
|-- giksan.jpg              # Логотип

/etc/systemd/system/
|-- giksan.service          # Сервис Flask-приложения
|-- giksan-bot.service      # Сервис Telegram-бота

/etc/nginx/sites-enabled/
|-- 24giksan.ru.conf        # Конфигурация Nginx
```

### Таблицы базы данных (Supabase)

| Таблица | Описание |
|---------|----------|
| `users` | Пользователи: email, пароль (bcrypt), fullname, photo, orders, auth_provider |
| `chats` | Чаты заказов: order_number, user_name, user_phone, user_email, delivery_address, items, total, payment_method, comment, status, admin_chat_id |
| `messages` | Сообщения чата: chat_id, sender (customer/admin), text, created_at, is_read |

---

## Технологический стек

### Бэкенд
- Python 3.12
- Flask 2.3.3 -- веб-фреймворк
- Gunicorn 25.3.0 -- WSGI-сервер
- Supabase SDK -- клиент к PostgreSQL
- bcrypt -- хэширование паролей
- pyTelegramBotAPI -- Telegram-бот
- openpyxl -- парсинг Excel-файлов

### Фронтенд
- Vanilla JavaScript (ES6+)
- CSS3 с CSS Custom Properties
- Font Awesome 6.4.0 -- иконки
- Google Fonts (Inter, Montserrat)

### Инфраструктура
- Ubuntu 24.04 LTS
- Nginx 1.24.0 -- reverse proxy
- Let's Encrypt -- SSL-сертификаты
- systemd -- управление сервисами

---

## Функциональность

### Каталог товаров

- Фильтрация по категориям, цене, наличию
- Полнотекстовый поиск по названию
- Сортировка по цене и названию
- Пагинация (12 товаров на странице)
- Кэширование в localStorage (5 минут TTL)
- Быстрый просмотр карточки товара

### Корзина и заказы

- Добавление/удаление товаров
- Изменение количества
- Сохранение в localStorage
- Оформление заказа с валидацией данных
- Автоматическое создание чата при оформлении

### Чат с менеджером

- Live-чат через базу данных (Supabase)
- Клиент видит сообщения на сайте (polling каждые 3 секунды)
- Администратор отвечает через Telegram-бота
- История чатов в профиле пользователя

### Telegram-бот

- Импорт товаров из файлов `.xlsx`, `.csv`, `.txt`, `.json`
- Уведомления о новых заказах
- Ответы на заказы через inline-кнопки
- Поддержка нескольких вариантов товара (разные ручки/материалы)

---

## API документация

### Аутентификация

| Метод | Путь | Описание |
|-------|------|----------|
| POST | `/api/auth/register` | Регистрация нового пользователя |
| POST | `/api/auth/login` | Вход в систему (email + пароль) |
| GET | `/api/auth/session` | Проверка текущей сессии |
| POST | `/api/auth/logout` | Выход из системы |
| GET | `/api/auth/google` | Инициация Google OAuth |
| GET | `/api/auth/google/callback` | Callback Google OAuth |

### Профиль пользователя

| Метод | Путь | Описание |
|-------|------|----------|
| GET | `/api/user/profile` | Получить профиль (требует авторизации) |
| PATCH | `/api/user/profile` | Обновить профиль (требует авторизации) |
| POST | `/api/user/photo` | Обновить фото профиля (требует авторизации) |
| POST | `/api/user/orders/add` | Добавить заказ (требует авторизации) |

### Заказы и чат

| Метод | Путь | Описание |
|-------|------|----------|
| POST | `/api/order/create` | Создать заказ + чат (публичный) |
| GET | `/api/chat/<chat_id>/messages` | Сообщения чата (требует авторизации) |
| GET | `/api/chat/<chat_id>/messages/public` | Сообщения чата (публичный, по email) |
| POST | `/api/chat/<chat_id>/message` | Отправить сообщение в чат (требует авторизации) |
| GET | `/api/chat/<chat_id>/status` | Статус чата (требует авторизации) |
| GET | `/api/chat/unread` | Количество непрочитанных сообщений (требует авторизации) |
| POST | `/api/chat/<chat_id>/read` | Отметить сообщения как прочитанные (требует авторизации) |
| GET | `/api/chats` | Все чаты пользователя (требует авторизации) |

### Шифрование

| Метод | Путь | Описание |
|-------|------|----------|
| GET | `/api/crypto/key` | Получить ключ сессии шифрования |
| POST | `/api/crypto/decrypt` | Расшифровать данные |
| POST | `/api/crypto/encrypt` | Зашифровать данные |

---

## Методы защиты

### 1. Шифрование данных (GOST R 34.12-2015 "Кузнечик")

Реализован блочный шифр ГОСТ Р 34.12-2015 с параметрами:

- **Размер блока:** 128 бит
- **Размер ключа:** 256 бит
- **Режим работы:** CTR (Counter Mode)
- **Алгоритм:** S-блок из ГОСТ Р 34.13-2015, линейное преобразование L

Реализация присутствует как на сервере (`crypto.py`), так и на клиенте (`kuznyechik.js`), обеспечивая сквозное шифрование данных.

| Компонент | Описание |
|-----------|----------|
| `crypto.py` | Серверная реализация на Python |
| `kuznyechik.js` | Клиентская реализация на JavaScript |
| Режим CTR | Позволяет шифровать данные произвольной длины |
| Nonce | 128-битный случайный вектор инициализации для каждой операции |

### 2. Защита сессий

| Механизм | Значение | Назначение |
|----------|----------|------------|
| `SESSION_COOKIE_SECURE` | True | Куки передаются только по HTTPS |
| `SESSION_COOKIE_HTTPONLY` | True | Куки недоступны для JavaScript (защита от XSS) |
| `SESSION_COOKIE_SAMESITE` | Lax | Защита от CSRF-атак |
| `PERMANENT_SESSION_LIFETIME` | 604800 (7 дней) | Ограниченное время жизни сессии |
| `FLASK_SECRET_KEY` | 64+ символа | Криптографическая подпись сессионных куки |

### 3. Хэширование паролей

- Алгоритм: **bcrypt** с автоматической генерацией соли
- Стоимость: значение по умолчанию (12 раундов)
- Каждая соль уникальна, что делает радужные таблицы бесполезными

### 4. Защита на уровне приложения

#### Валидация входных данных

- Телефон: регулярное выражение `^\+?[\d\s\-\(\)]{10,20}$`
- Email: RFC-совместимая валидация через регулярное выражение
- Пароль: минимум 8 символов, обязательное наличие букв и цифр

#### HTML-экранирование

- Все данные, отправляемые в Telegram, проходят через `html.escape()`
- Предотвращает XSS через пользовательский ввод

#### Контроль доступа

- Декоратор `@login_required` для защищённых endpoint
- Проверка принадлежности чата к пользователю через email
- Доступ к сообщениям чата только для владельца

#### Защита от path traversal

- Проверка на наличие `..` в путях
- Проверка на абсолютные пути
- Нормализация пути через `os.path.normpath()`

### 5. Транспортная безопасность

| Параметр | Значение |
|----------|----------|
| Протокол | HTTPS (TLS 1.2/1.3) |
| Сертификат | Let's Encrypt (автообновление) |
| HTTP/2 | Включён |
| HSTS | Через настройки Let's Encrypt |
| Редирект | HTTP 301 на HTTPS |

### 6. Безопасность сервера

| Мера | Описание |
|------|----------|
| Файрвол UFW | Открыты только порты 22, 80, 443 |
| Изоляция сервисов | Сервисы работают от пользователя `giksan` (без root) |
| systemd | Автоматический перезапуск при падении (Restart=always) |
| Virtual Environment | Python-зависимости изолированы |
| Сессии шифрования | Автоматическая очистка сессий старше 1 часа |

### 7. Защита API

| Механизм | Описание |
|----------|----------|
| CORS | Разрешены только доверенные источники |
| Credentials | Передача куки только для своего домена |
| Supabase Service Key | Используется только на сервере, не раскрывается клиенту |
| Supabase Anon Key | Ограниченные права, только чтение публичных данных |

---

## Развертывание

### Требования к серверу

- Ubuntu 24.04 LTS (рекомендуется)
- Минимум 1 CPU, 1 ГБ RAM
- Python 3.12+
- Nginx

### Установка

1. **Подготовка сервера**

```bash
apt update && apt upgrade -y
apt install -y python3 python3-pip python3-venv nginx certbot python3-certbot-nginx
```

2. **Создание пользователя и директории**

```bash
useradd -m -s /bin/bash giksan
mkdir -p /var/www/giksan
chown giksan:giksan /var/www/giksan
```

3. **Установка зависимостей**

```bash
cd /var/www/giksan
python3 -m venv venv
source venv/bin/activate
pip install gunicorn
pip install -r requirements.txt
```

4. **Настройка переменных окружения**

Создать файл `.env` в `/var/www/giksan/` с необходимыми ключами.

5. **Запуск сервисов**

```bash
systemctl daemon-reload
systemctl enable giksan
systemctl enable giksan-bot
systemctl start giksan
systemctl start giksan-bot
```

6. **Настройка Nginx и SSL**

```bash
certbot --nginx -d 24giksan.ru -d www.24giksan.ru \
    --non-interactive --agree-tos --email info@giksan.ru
```

---

## Конфигурация

### Переменные окружения

| Переменная | Описание |
|------------|----------|
| `BOT_TOKEN` | Токен Telegram-бота |
| `TELEGRAM_ADMIN_ID` | ID администратора в Telegram |
| `SUPABASE_URL` | URL проекта Supabase |
| `SUPABASE_ANON_KEY` | Публичный ключ Supabase |
| `SUPABASE_SERVICE_KEY` | Сервисный ключ Supabase (только сервер) |
| `GOOGLE_CLIENT_ID` | Google OAuth Client ID |
| `GOOGLE_CLIENT_SECRET` | Google OAuth Client Secret |
| `GOOGLE_REDIRECT_URI` | Callback URL для Google OAuth |
| `FLASK_SECRET_KEY` | Секретный ключ Flask для подписи сессий |
| `FLASK_ENV` | Окружение: development или production |
| `FLASK_DEBUG` | Режим отладки: 0 или 1 |
| `SESSION_COOKIE_SECURE` | True для production (HTTPS) |
| `KUZNYECHIK_SERVER_KEY` | 256-битный ключ для шифрования (hex) |

### Сервисы systemd

**giksan.service** -- Flask-приложение:

```ini
ExecStart=/var/www/giksan/venv/bin/gunicorn \
    --bind 127.0.0.1:5000 \
    --workers 1 --threads 4 \
    --worker-class gthread \
    --timeout 120 \
    --keep-alive 5 \
    --preload main:app
```

**giksan-bot.service** -- Telegram-бот:

```ini
ExecStart=/var/www/giksan/venv/bin/python3 telegram_bot.py
Restart=always
RestartSec=5
```

---

## Лицензия

Все права защищены. GIKSAN (c) 2026.
