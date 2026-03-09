## Практическое занятие №1: Микросервисная архитектура на Go

### Выполнил: Студент ЭФМО-02-25 Пягай Даниил Игоревич

##  О проекте

Проект представляет собой учебную реализацию микросервисной архитектуры, состоящей из двух сервисов:

- **Auth Service** — сервис аутентификации и проверки токенов
- **Tasks Service** — сервис управления задачами (CRUD)

Основная цель — демонстрация взаимодействия между микросервисами через HTTP с использованием таймаутов, проброса request-id и корректной обработки ошибок.

---

## Структура
![structure](screen/structure.png)


---

## 🏗 Архитектура

### Границы сервисов

**Auth Service** (порт `8081`)
- Аутентификация пользователей
- Выдача токенов
- Проверка валидности токенов
- Упрощенная логика: `student/student` → `demo-token`

**Tasks Service** (порт `8082`)
- CRUD операции с задачами
- Проверка токенов через Auth Service
- In-memory хранилище с потокобезопасным доступом
- Таймаут запросов к Auth: 3 секунды

### Схема взаимодействия

```mermaid
sequenceDiagram
    participant Client as Клиент (Postman)
    participant Tasks as Tasks Service
    participant Auth as Auth Service
    
    Client->>Tasks: Запрос с Authorization: Bearer <token>
    Note over Tasks: Извлекает токен из заголовка
    
    Tasks->>Auth: GET /v1/auth/verify (таймаут 3с)
    Note over Tasks: Прокидывает X-Request-ID
    
    alt Токен валидный
        Auth-->>Tasks: 200 OK {valid: true, subject: "student"}
        Tasks-->>Client: Успешный ответ (200/201)
    else Токен невалидный
        Auth-->>Tasks: 401 Unauthorized
        Tasks-->>Client: 401 Unauthorized
    else Auth сервис недоступен
        Auth-->>Tasks: timeout/5xx
        Tasks-->>Client: 503 Service Unavailable
    end
```

### Список эндпоинтов

#### Auth Service (gRPC, порт 50051)

| Метод | Запрос | Ответ | Описание |
|-------|--------|-------|----------|
| `Verify` | `{ token }` | `{ valid, subject }` | Проверка токена |

#### Tasks Service (HTTP, порт 8082)

| Метод | Путь | Описание |
|-------|------|----------|
| `GET` | `/tasks` | Список задач |
| `GET` | `/tasks/{id}` | Задача по ID |
| `POST` | `/tasks` | Создать задачу |
| `PATCH` | `/tasks/{id}` | Обновить задачу |
| `DELETE` | `/tasks/{id}` | Удалить задачу |

### Проверка запросов POSTMAN

Запрос №1: Получить токен

![post](screen/post_success_1.png)

Запрос №2: Проверить токен

![get](screen/get_success_2.png)

Запрос №3: Создать задачу

![post](screen/post_create_tasks.png)

Запрос 4: Получить список задач (Успех)

![get](screen/get_list.png)

Запрос 5: Получить список задач (Ошибка)

![get](screen/get_error.png)

### Логирование

Логи Auth сервиса

```
2025/02/25 10:15:23 Auth service starting on 0.0.0.0:8081
2025/02/25 10:15:30 POST /v1/auth/login - 200 OK (request-id: req-123)
2025/02/25 10:15:35 GET /v1/auth/verify - 200 OK (request-id: req-456)
2025/02/25 10:15:42 GET /v1/auth/verify - 401 Unauthorized (request-id: req-789)
```
Логи Tasks сервиса

```
2025/02/25 10:15:30 Tasks service starting on 0.0.0.0:8082
2025/02/25 10:15:30 Using Auth service at http://auth-service:8081
2025/02/25 10:15:35 POST /v1/tasks - 201 Created (request-id: req-123)
2025/02/25 10:15:40 GET /v1/tasks - 200 OK (request-id: req-456)
2025/02/25 10:15:45 GET /v1/tasks - 401 Unauthorized - missing token (request-id: req-789)
2025/02/25 10:15:50 GET /v1/tasks - 503 Service Unavailable - auth service timeout (request-id: req-000)
```
