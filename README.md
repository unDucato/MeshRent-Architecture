# 🏠 MeshRent — Premium SaaS Rental Platform & Marketplace

![Status](https://img.shields.io/badge/Status-Production-success)
![Platform](https://img.shields.io/badge/Platform-Web%20%7C%20PWA%20%7C%20iOS%20%7C%20Android-blue)
![Backend](https://img.shields.io/badge/Backend-Python_3.12_%7C_AsyncIO-3776AB?logo=python&logoColor=white)
![Frontend](https://img.shields.io/badge/Frontend-Vanilla_JS-F7DF1E?logo=javascript&logoColor=black)

**MeshRent** (Mesh) — это независимая глобальная SaaS-платформа и маркетплейс для управления и поиска аренды (недвижимость, транспорт, вещи, услуги). Проект объединяет в себе мощную CRM-систему для владельцев (Hosts) и удобную витрину с умным поиском для клиентов (Guests).

> 🔒 **Примечание:** Данный репозиторий представляет собой **архитектурный концепт** и содержит выдержки из кодовой базы. Полный исходный код является закрытым коммерческим продуктом. Готов продемонстрировать исходники, DevOps-инфраструктуру и процессы деплоя на техническом интервью.

🚀 **Production / Рабочий проект:** [meshrent.com](https://meshrent.com) (Полноценный релиз: открывается в браузере или устанавливается как PWA)

---

## 🎯 Продуктовое видение (Product Vision)
Цель проекта — предоставить универсальную и быструю систему для шеринговой экономики, не уступающую нативным приложениям по скорости и UX. Проект спроектирован как продвинутое PWA-решение с единой кодовой базой для Web, устанавливаемых приложений и сборки под iOS/Android через Capacitor.

**Ключевые продуктовые фичи:**
- **Zero-Flicker UX & Smart Render:** Мгновенный рендеринг интерфейса. Данные моментально отрисовываются из локального кэша, а затем *бесшовно* и "тихо" обновляются из фона без морганий (Skeletons используются только при "холодном старте").
- **AI-Ассистент и Оркестрация LLM:** Встроенный нейросетевой бизнес-радар для владельцев (генерация утренних сводок, анализ логистики, расчет цен, автоответы на отзывы) и NLP-парсер для умного поиска гостями.
- **Глобальная локализация:** Бесшовная мультивалютность с автоконвертацией по актуальным курсам и фоновый поточный перевод карточек объектов на лету (Google Translate API).
- **Интеграция и Синхронизация (iCal):** Двусторонний обмен календарями с Airbnb, Booking, Avito, Суточно.ру и другими площадками для предотвращения овербукинга.
- **Умные Договоры:** Автоматическая генерация и предзаполнение договоров аренды (HTML/PDF) на основе данных владельца и гостя.

---

## 🏗 Архитектура системы (System Architecture)

Система построена на базе асинхронного ядра **Python (AsyncIO + aiohttp)**. Все I/O операции строго неблокирующие. Разделение бизнес-логики реализовано через изолированные процессы (API, Admin, Background Workers), общающиеся через **Redis Pub/Sub**.

```mermaid
flowchart TD
    Client["Web / PWA / Native APK"] -->|"HTTPS / WSS"| Nginx["Nginx Reverse Proxy"]
    
    Nginx -->|"API Requests / WS"| MainApp["Main App API (aiohttp)"]
    Nginx -->|"Port 8081"| AdminApp["Admin Server (aiohttp)"]
    
    subgraph Storage ["Data Layer"]
        DB[("PostgreSQL")]
        Redis[("Redis")]
    end
    
    MainApp -->|"AsyncPG / SQLModel"| DB
    MainApp -->|"Cache & Pub/Sub"| Redis
    
    AdminApp -.-> DB
    
    MainApp -->|"S3 Async Uploads"| S3["Yandex Cloud S3"]
    
    MainApp -.->|"Push (WebPush/APNs)"| FCM["Firebase Admin SDK"]
    MainApp -.->|"LLM API"| AI["OpenRouter / IO.net"]
    MainApp -.->|"Payments"| Payment["Robokassa / TG Stars"]
    
    subgraph Background_Workers ["Background Task Loop"]
        Worker1["AI Night Worker"]
        Worker2["iCal Sync Loop"]
        Worker3["Garbage Collector (S3/Auth)"]
        Worker4["Auto-Translation"]
    end
    
    MainApp --> Background_Workers
```

---

## 🗄 Схема Базы Данных (E-R Diagram)

База данных спроектирована в **PostgreSQL**. Активно используются поля `JSONB` для хранения кэша переводов, сложных настроек объектов, AI-отчетов и метаданных аналитики. ORM: **SQLModel**.

```mermaid
erDiagram
    USERS ||--o{ RENT_OBJECTS : owns
    USERS ||--o{ BOOKINGS : makes
    USERS ||--o{ TRANSACTIONS : completes
    RENT_OBJECTS ||--o{ BOOKINGS : has
    BOOKINGS ||--o{ REVIEWS : receives
    
    USERS {
        int user_id PK
        string email UK
        jsonb tech_specs_json
        string communication_lang
        int balance_mesh
        datetime expiry_date "PRO Status"
    }
    
    RENT_OBJECTS {
        int id PK
        string public_id UK
        string entity_type "object / service"
        string billing_type "hourly / daily / monthly"
        jsonb translations_json "Translations Cache"
        jsonb ical_links_json
    }
    
    BOOKINGS {
        int id PK
        int object_id FK
        int guest_user_id FK
        datetime start_date
        datetime end_date
        int price
        string status "pending / active / finished / cancelled"
    }
    
    REVIEWS {
        int id PK
        int booking_id FK
        int target_object_id FK
        float rating
        jsonb details_json
    }
    
    TRANSACTIONS {
        int id PK
        int user_id FK
        string type "topup / spend"
        int amount_mesh
        string gateway "robokassa / stars"
    }
```

---

## 🚀 Ключевые технические решения (Engineering Highlights)

### 1. Архитектура Frontend: DOM vs Network Chunking (Vanilla JS)
Плавность работы со списками достигается за счет двойной пагинации:
- **Network Chunking:** Данные запрашиваются с бэкенда большими пакетами (по 50 элементов), ответы сжимаются с помощью прозрачного GZIP.
- **DOM Chunking (Lazy Rendering):** Для предотвращения фризов `Main Thread` в DOM элементы рендерятся малыми "чанками" (по 20 шт) по мере скролла пользователя (отслеживание через `requestAnimationFrame`).

### 2. Resilient AI Orchestration & AST Fallback
- Мульти-провайдерная логика LLM (переключение между IO.net и OpenRouter). В случае получения HTTP 429 от провайдера, ключ мгновенно уходит в `Redis Cooldown`, и запрос бесшовно передается на следующий ключ/модель.
- **AST Recovery:** Поскольку нейросети периодически возвращают "грязный" JSON (с Markdown-тегами или одинарными кавычками), реализован интеллектуальный парсер с фоллбэком на `ast.literal_eval` Python для гарантированного восстановления бизнес-данных.

### 3. Картографический движок (MapLibre GL JS)
- Использование MapLibre 5.x с поддержкой аппаратного ускорения (WebGL) и 3D Глобуса. 
- Динамическая кластеризация маркеров на клиенте и расчет дистанции (Haversine formula).
- Интеграция процедурных эффектов (Огни городов ночью, Звездное небо, Атмосфера), зависящих от `pitch` и `zoom` камеры.

### 4. Zero Trust & Anti-Spam Security
- Проверка авторизации на уровне кастомных aiohttp Middlewares. На клиенте состояние визуально оптимистично, но сервер валидирует все изменения данных.
- Подтверждения по Email через временные коды (OTP), хранящиеся в БД с жестким лимитом попыток (`brute-force protection`).
- Все I/O (Загрузка в S3 бакет) использует валидацию `Magic Bytes`, отсекая исполняемые файлы до этапа обработки.

### 5. Unified Notification Protocol & DND
- Единый интерфейс пуш-уведомлений через **Firebase Cloud Messaging**.
- **Серверный Do Not Disturb (DND):** Бэкенд автоматически вычисляет локальное время пользователя с помощью `pytz`. Если пользователь спит, стандартный алерт понижается до "Тихого пуша" (`silent=True`), обновляющего локальную IndexedDB без звука и вибрации.

---

## 🛠 Стек технологий (Tech Stack)

**Backend:**
* `Python 3.12`, `AsyncIO`
* `Aiohttp` (Web Server, Middlewares, WebSockets)
* `PostgreSQL` + `asyncpg` + `SQLModel`
* `Redis` (Pub/Sub & Rate Limiting ZSET)
* `OpenRouter API` / `IO.net API` (AI LLMs: LLaMA 3, Qwen, DeepSeek)

**Frontend / Mobile:**
* `Vanilla JavaScript` (ES6+), `HTML5`, `CSS3`
* `MapLibre GL JS 5.24.0` (Map Engine & 3D Globe)
* `Chart.js` (Analytics visualization)
* `Service Workers` & `IndexedDB` (PWA Offline First)
* `Capacitor` (Сборка Native iOS/Android App)

**Infrastructure / Integrations:**
* `Docker` & `Docker Compose`
* `Nginx`
* `Yandex Cloud S3` (`aioboto3`)
* `Firebase Admin SDK` (Push Notifications / WebPush)
* `Robokassa API` & `Telegram Bot API` (Payments Gateway)

---
*Проект разработан и поддерживается Александром Кузиным.*
