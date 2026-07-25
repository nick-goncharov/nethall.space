# Nethall - Диаграммы и Схемы

## 🏗️ Архитектурные диаграммы

### 1. Общая архитектура системы

```mermaid
graph TB
    subgraph "Client Layer"
        WEB[Web Browser]
        MOBILE[Mobile App]
    end
    
    subgraph "CDN Layer"
        CF[CloudFlare CDN]
    end
    
    subgraph "Frontend Layer"
        NEXT[Next.js App<br/>Vercel]
    end
    
    subgraph "Backend Layer"
        LB[Load Balancer]
        API1[API Server 1]
        API2[API Server 2]
        WS[WebSocket Server]
    end
    
    subgraph "Video Layer"
        LK1[LiveKit Moscow]
        LK2[LiveKit EU]
        LK3[LiveKit US]
    end
    
    subgraph "Data Layer"
        PG[(PostgreSQL)]
        REDIS[(Redis)]
        S3[S3 Storage]
    end
    
    subgraph "External Services"
        STRIPE[Stripe]
        CP[CloudPayments]
        SG[SendGrid]
        FCM[Firebase]
    end
    
    WEB --> CF
    MOBILE --> CF
    CF --> NEXT
    NEXT --> LB
    LB --> API1
    LB --> API2
    WEB --> WS
    MOBILE --> WS
    
    API1 --> PG
    API2 --> PG
    API1 --> REDIS
    API2 --> REDIS
    API1 --> S3
    API2 --> S3
    
    WEB --> LK1
    WEB --> LK2
    WEB --> LK3
    
    API1 --> STRIPE
    API1 --> CP
    API1 --> SG
    API1 --> FCM
```

### 2. Поток данных при создании события

```mermaid
sequenceDiagram
    participant U as User
    participant FE as Frontend
    participant API as Backend API
    participant DB as PostgreSQL
    participant LK as LiveKit
    participant S3 as S3 Storage
    
    U->>FE: Создать событие
    FE->>API: POST /api/events
    API->>DB: Сохранить событие
    DB-->>API: Event ID
    API->>LK: Создать комнату
    LK-->>API: Room Token
    API-->>FE: Event данные
    FE-->>U: Событие создано
    
    Note over U,S3: Начало трансляции
    
    U->>FE: Начать трансляцию
    FE->>API: POST /api/events/:id/start
    API->>DB: Обновить статус
    API->>LK: Активировать запись
    API-->>FE: Трансляция началась
    
    Note over U,S3: Завершение трансляции
    
    U->>FE: Завершить трансляцию
    FE->>API: POST /api/events/:id/end
    API->>LK: Остановить запись
    LK->>S3: Загрузить видео
    API->>DB: Обновить статус
    API-->>FE: Трансляция завершена
```

### 3. Процесс оплаты

```mermaid
sequenceDiagram
    participant U as User
    participant FE as Frontend
    participant API as Backend
    participant DB as Database
    participant STRIPE as Stripe
    participant EMAIL as Email Service
    
    U->>FE: Выбрать событие
    FE->>API: GET /api/events/:id
    API-->>FE: Event details
    
    U->>FE: Оплатить участие
    FE->>API: POST /api/events/:id/register
    API->>DB: Создать транзакцию (pending)
    API->>STRIPE: Создать Checkout Session
    STRIPE-->>API: Session URL
    API-->>FE: Redirect URL
    FE->>STRIPE: Redirect to Stripe
    
    U->>STRIPE: Ввести данные карты
    STRIPE->>API: Webhook: payment_intent.succeeded
    API->>DB: Обновить транзакцию (completed)
    API->>DB: Добавить участника
    API->>EMAIL: Отправить подтверждение
    API-->>STRIPE: 200 OK
    
    STRIPE->>FE: Redirect success
    FE-->>U: Оплата успешна
```

### 4. Real-time чат архитектура

```mermaid
graph LR
    subgraph "Clients"
        C1[Client 1]
        C2[Client 2]
        C3[Client 3]
    end
    
    subgraph "Backend"
        WS[WebSocket Server<br/>Socket.io]
        REDIS[(Redis<br/>Pub/Sub)]
        DB[(PostgreSQL)]
    end
    
    C1 -->|WebSocket| WS
    C2 -->|WebSocket| WS
    C3 -->|WebSocket| WS
    
    WS -->|Publish| REDIS
    REDIS -->|Subscribe| WS
    WS -->|Save| DB
    
    WS -->|Broadcast| C1
    WS -->|Broadcast| C2
    WS -->|Broadcast| C3
```

### 5. Система уведомлений

```mermaid
graph TB
    subgraph "Triggers"
        EVENT[Event Created]
        PAYMENT[Payment Received]
        REMINDER[Scheduled Reminder]
        DONATION[Donation Received]
    end
    
    subgraph "Notification Service"
        NS[Notification Service]
        QUEUE[Job Queue]
    end
    
    subgraph "Channels"
        EMAIL[Email<br/>SendGrid]
        PUSH[Push<br/>FCM]
        INAPP[In-App<br/>Database]
    end
    
    subgraph "Users"
        USER[User Device]
    end
    
    EVENT --> NS
    PAYMENT --> NS
    REMINDER --> NS
    DONATION --> NS
    
    NS --> QUEUE
    QUEUE --> EMAIL
    QUEUE --> PUSH
    QUEUE --> INAPP
    
    EMAIL --> USER
    PUSH --> USER
    INAPP --> USER
```

---

## 📊 Модель данных (ER диаграмма)

```mermaid
erDiagram
    USERS ||--o{ OAUTH_ACCOUNTS : has
    USERS ||--o{ GROUPS : creates
    USERS ||--o{ EVENTS : organizes
    USERS ||--o{ EVENT_PARTICIPANTS : participates
    USERS ||--|| WALLETS : owns
    USERS ||--o{ TRANSACTIONS : makes
    USERS ||--o{ MESSAGES : sends
    USERS ||--o{ DONATIONS : sends
    USERS ||--o{ DONATIONS : receives
    USERS ||--o{ GUARANTORS : guarantees
    USERS ||--o{ GUARANTORS : guaranteed_by
    USERS ||--o{ NOTIFICATIONS : receives
    USERS ||--o{ REPORTS : reports
    USERS ||--o{ REPORTS : reported
    USERS ||--|| USER_SETTINGS : has
    
    GROUPS ||--o{ EVENTS : contains
    GROUPS }o--|| CATEGORIES : belongs_to
    
    EVENTS ||--o{ EVENT_PARTICIPANTS : has
    EVENTS ||--o{ MESSAGES : contains
    EVENTS ||--o{ DONATIONS : receives
    EVENTS ||--o{ EVENT_TAGS : tagged_with
    
    TAGS ||--o{ EVENT_TAGS : used_in
    
    WALLETS ||--o{ TRANSACTIONS : records
    WALLETS ||--o{ WITHDRAWAL_REQUESTS : requests
    
    TRANSACTIONS ||--o{ DONATIONS : tracks
    
    USERS {
        uuid id PK
        string email UK
        string password_hash
        string username UK
        string full_name
        string avatar_url
        text bio
        enum role
        enum status
        boolean email_verified
        timestamp created_at
    }
    
    EVENTS {
        uuid id PK
        uuid group_id FK
        string title
        text description
        string slug UK
        timestamp scheduled_at
        integer duration_minutes
        enum status
        boolean is_recorded
        string recording_url
        boolean chat_enabled
        decimal participant_price
        decimal viewer_price
        timestamp created_at
    }
    
    WALLETS {
        uuid id PK
        uuid user_id FK
        decimal balance
        string currency
        timestamp updated_at
    }
    
    TRANSACTIONS {
        uuid id PK
        uuid wallet_id FK
        enum type
        decimal amount
        enum status
        string payment_provider
        timestamp created_at
    }
```

---

## 🔄 Бизнес-процессы

### Процесс становления организатором

```mermaid
stateDiagram-v2
    [*] --> Regular_User
    Regular_User --> Seeking_Guarantors: Запрос статуса
    Seeking_Guarantors --> Has_3_Guarantors: 3 поручителя найдены
    Has_3_Guarantors --> First_Event_Created: Создано первое событие
    First_Event_Created --> Guarantors_Notified: Уведомления отправлены
    Guarantors_Notified --> Guarantors_Attended: Поручители посетили
    Guarantors_Attended --> Organizer: Статус активирован
    Organizer --> [*]
    
    Seeking_Guarantors --> Regular_User: Отмена
    Has_3_Guarantors --> Regular_User: Отмена
```

### Жизненный цикл события

```mermaid
stateDiagram-v2
    [*] --> Draft: Создание
    Draft --> Scheduled: Публикация
    Scheduled --> Live: Начало трансляции
    Live --> Processing: Завершение
    Processing --> Ended: Обработка завершена
    Ended --> [*]
    
    Draft --> Cancelled: Отмена
    Scheduled --> Cancelled: Отмена
    Cancelled --> [*]
    
    note right of Live
        Участники подключены
        Чат активен
        Запись идет
    end note
    
    note right of Processing
        Конвертация видео
        Загрузка на S3
        Генерация превью
    end note
```

### Процесс вывода средств

```mermaid
stateDiagram-v2
    [*] --> User_Request: Заявка создана
    User_Request --> Pending_Review: В очереди
    Pending_Review --> Under_Review: Модератор проверяет
    Under_Review --> Approved: Одобрено
    Under_Review --> Rejected: Отклонено
    Approved --> Processing: Обработка платежа
    Processing --> Completed: Выплачено
    Processing --> Failed: Ошибка
    Failed --> Pending_Review: Повтор
    Completed --> [*]
    Rejected --> [*]
    
    note right of Under_Review
        Проверка:
        - Минимальная сумма
        - Верификация данных
        - Антифрод
    end note
```

---

## 🎨 UI/UX Flow

### Пользовательский путь: Участие в событии

```mermaid
graph TD
    START([Пользователь заходит на сайт]) --> BROWSE[Просмотр событий]
    BROWSE --> SEARCH{Поиск события}
    SEARCH -->|Найдено| DETAIL[Страница события]
    SEARCH -->|Не найдено| BROWSE
    
    DETAIL --> AUTH{Авторизован?}
    AUTH -->|Нет| LOGIN[Вход/Регистрация]
    LOGIN --> DETAIL
    AUTH -->|Да| PAID{Платное?}
    
    PAID -->|Да| PAYMENT[Оплата]
    PAYMENT --> SUCCESS{Успешно?}
    SUCCESS -->|Нет| PAYMENT
    SUCCESS -->|Да| REGISTERED[Зарегистрирован]
    
    PAID -->|Нет| REGISTERED
    REGISTERED --> WAIT[Ожидание начала]
    WAIT --> REMINDER[Получение напоминания]
    REMINDER --> JOIN[Присоединение к трансляции]
    JOIN --> PARTICIPATE[Участие в событии]
    PARTICIPATE --> DONATE{Отправить донат?}
    DONATE -->|Да| DONATION[Донат]
    DONATE -->|Нет| END
    DONATION --> END([Завершение])
```

### Организатор: Создание и проведение события

```mermaid
graph TD
    START([Организатор входит]) --> DASHBOARD[Dashboard]
    DASHBOARD --> CREATE[Создать событие]
    CREATE --> FORM[Заполнить форму]
    FORM --> DETAILS[Детали события]
    DETAILS --> PRICING[Настройка цен]
    PRICING --> PUBLISH[Публикация]
    PUBLISH --> SHARE[Поделиться ссылкой]
    
    SHARE --> WAIT[Ожидание начала]
    WAIT --> PREP[Подготовка к эфиру]
    PREP --> START_STREAM[Начать трансляцию]
    START_STREAM --> MANAGE[Управление участниками]
    MANAGE --> CHAT[Модерация чата]
    CHAT --> RECEIVE_DONATIONS[Получение донатов]
    RECEIVE_DONATIONS --> END_STREAM[Завершить трансляцию]
    END_STREAM --> STATS[Просмотр статистики]
    STATS --> WITHDRAW{Вывести средства?}
    WITHDRAW -->|Да| WITHDRAWAL[Заявка на вывод]
    WITHDRAW -->|Нет| FINISH([Завершение])
    WITHDRAWAL --> FINISH
```

---

## 🔐 Безопасность: Слои защиты

```mermaid
graph TB
    subgraph "Layer 1: Network"
        CF[CloudFlare DDoS Protection]
        WAF[Web Application Firewall]
    end
    
    subgraph "Layer 2: Application"
        RATE[Rate Limiting]
        CORS[CORS Policy]
        CSP[Content Security Policy]
    end
    
    subgraph "Layer 3: Authentication"
        JWT[JWT Tokens]
        OAUTH[OAuth 2.0]
        MFA[2FA Optional]
    end
    
    subgraph "Layer 4: Authorization"
        RBAC[Role-Based Access Control]
        PERMS[Permission Checks]
    end
    
    subgraph "Layer 5: Data"
        ENCRYPT[Encryption at Rest]
        HASH[Password Hashing]
        SANITIZE[Input Sanitization]
    end
    
    subgraph "Layer 6: Monitoring"
        SENTRY[Error Tracking]
        LOGS[Audit Logs]
        ALERTS[Security Alerts]
    end
    
    CF --> WAF
    WAF --> RATE
    RATE --> CORS
    CORS --> CSP
    CSP --> JWT
    JWT --> OAUTH
    OAUTH --> MFA
    MFA --> RBAC
    RBAC --> PERMS
    PERMS --> ENCRYPT
    ENCRYPT --> HASH
    HASH --> SANITIZE
    SANITIZE --> SENTRY
    SENTRY --> LOGS
    LOGS --> ALERTS
```

---

## 📈 Масштабирование стратегия

```mermaid
graph TB
    subgraph "Phase 1: MVP - 100 users"
        S1[Single API Server]
        DB1[(PostgreSQL)]
        R1[(Redis)]
        LK1[LiveKit Single]
    end
    
    subgraph "Phase 2: Growth - 1K users"
        LB1[Load Balancer]
        S2[API Server 1]
        S3[API Server 2]
        DB2[(PostgreSQL Primary)]
        DB3[(Read Replica)]
        R2[(Redis Cluster)]
        LK2[LiveKit Moscow]
        LK3[LiveKit EU]
    end
    
    subgraph "Phase 3: Scale - 10K+ users"
        LB2[Global Load Balancer]
        MS1[Microservice: Auth]
        MS2[Microservice: Events]
        MS3[Microservice: Payments]
        MS4[Microservice: Video]
        DB4[(PostgreSQL Cluster)]
        R3[(Redis Cluster)]
        ES[(Elasticsearch)]
        LK4[LiveKit Multi-Region]
    end
    
    S1 --> LB1
    LB1 --> S2
    LB1 --> S3
    
    S2 --> LB2
    S3 --> LB2
    LB2 --> MS1
    LB2 --> MS2
    LB2 --> MS3
    LB2 --> MS4
```

---

## 🎯 Метрики и мониторинг

```mermaid
graph LR
    subgraph "Application"
        APP[Nethall App]
    end
    
    subgraph "Metrics Collection"
        PROM[Prometheus]
        SENTRY[Sentry]
        GA[Google Analytics]
    end
    
    subgraph "Visualization"
        GRAF[Grafana Dashboards]
        ALERT[Alert Manager]
    end
    
    subgraph "Notifications"
        EMAIL[Email]
        SLACK[Slack]
        SMS[SMS]
    end
    
    APP -->|Metrics| PROM
    APP -->|Errors| SENTRY
    APP -->|Analytics| GA
    
    PROM --> GRAF
    SENTRY --> GRAF
    
    GRAF --> ALERT
    ALERT --> EMAIL
    ALERT --> SLACK
    ALERT --> SMS
```

---

**Дата создания**: 24 июля 2026  
**Версия**: 1.0
