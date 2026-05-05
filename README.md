# Rutina Gateway Service

**Rutina Gateway Service** — API Gateway для микросервисной системы **Rutina**. Единая точка входа для всех клиентских запросов, маршрутизирует трафик к соответствующим микросервисам.

Сервис является частью микросервисной системы **Rutina** и взаимодействует с:
- **Auth Service** (порт 8082) — аутентификация и пользователи
- **Main Service** (порт 8083) — управление привычками
- **Android приложение** — мобильный клиент

---

## Архитектура

```
┌──────────────────────────┐
│    Android Client         │
│    (Jetpack Compose)      │
│                           │
│  Все запросы → :8081      │
└────────────┬─────────────┘
             │
             │ HTTP (единый порт)
             │
    ┌────────▼────────────┐
    │   Gateway Service    │
    │   (порт 8081)        │
    │                      │
    │  Маршрутизация:      │
    │  /auth/**  → :8082   │
    │  /admin/** → :8082   │
    │  /users/** → :8083   │
    └──┬──────────────┬────┘
       │              │
       │ localhost    │ localhost
       │              │
┌──────▼──────┐ ┌─────▼──────┐
│ Auth Service │ │Main Service│
│  (порт 8082) │ │(порт 8083) │
└──────────────┘ └────────────┘
```

---

## Зачем нужен Gateway

В микросервисной архитектуре **Gateway** решает несколько ключевых задач:

1. **Единая точка входа** — клиенты (Android, браузер) обращаются только к Gateway, не зная о внутренних сервисах
2. **Маршрутизация** — направляет запросы в нужный микросервис на основе пути URL
3. **Сокрытие внутренней структуры** — Auth и Main сервисы недоступны извне (слушают только localhost)
4. **Упрощение клиента** — Android приложение использует только один IP и порт

---

## Таблица маршрутизации

| Путь | Микросервис | Порт | Описание |
|------|------------|------|----------|
| `/auth/**` | Auth Service | 8082 | Регистрация, вход, валидация токенов |
| `/admin/**` | Auth Service | 8082 | Админ-панель (управление пользователями) |
| `/users/**` | Main Service | 8083 | CRUD привычек |

---

## Технологический стек

- **Kotlin**
- **Spring Boot 3.2.0**
- **Spring Cloud Gateway** — реактивная маршрутизация
- **Spring WebFlux** — реактивный веб-фреймворк
- **Project Reactor** — асинхронная обработка запросов
- **Netty** — встроенный сервер
- **Gradle** — сборка проекта

---

## Структура проекта

```
src/main/kotlin/com/example/demo/
└── DemoApplication.kt          # Точка входа Spring Boot + Gateway

src/main/resources/
└── application.yaml             # Конфигурация маршрутов и портов
```
**Gateway не содержит бизнес-логики**, только маршрутизацию.

---

## Конфигурация

### `application.yaml`

```yaml
server:
  port: 8081                    # Порт для клиентов
  address: 0.0.0.0              # Слушать все сетевые интерфейсы

spring:
  application:
    name: rutina-gateway
  cloud:
    gateway:
      routes:
        # Маршрут к Auth Service
        - id: rutina-auth
          uri: http://localhost:8082
          predicates:
            - Path=/auth/**, /admin/**
          filters:
            - StripPrefix=0

        # Маршрут к Main Service
        - id: rutina-main
          uri: http://localhost:8083
          predicates:
            - Path=/users/**
          filters:
            - StripPrefix=0

logging:
  level:
    org.springframework.cloud.gateway: DEBUG  # Логирование маршрутизации
```

### Как читать маршруты:

```
Запрос: GET http://192.168.1.9:8081/auth/login
         ↓
Gateway видит путь /auth/**
         ↓
Перенаправляет на: http://localhost:8082/auth/login
         ↓
Auth Service обрабатывает запрос
         ↓
Gateway возвращает ответ клиенту
```


## Логирование

При запуске с `DEBUG` уровнем в логах видна маршрутизация:

```
DEBUG o.s.c.g.r.RouteDefinitionRouteLocator : RouteDefinition rutina-auth applying {pattern=/auth/**} to Path
DEBUG o.s.c.g.r.RouteDefinitionRouteLocator : RouteDefinition rutina-main applying {pattern=/users/**} to Path
```

При каждом запросе:
```
DEBUG o.s.c.g.h.RoutePredicateHandlerMapping : Route matched: rutina-auth
DEBUG o.s.c.g.h.RoutePredicateHandlerMapping : Mapping [Exchange: GET http://localhost:8081/auth/login] to Route{id='rutina-auth', uri=http://localhost:8082}
```

---

## Особенности реализации

- **Spring Cloud Gateway** — реактивный (на WebFlux/Netty)
- Неблокирующая обработка запросов
- Лучшая производительность под нагрузкой
- Активно поддерживается Spring Team

Порты распределены так:
- **8081** — Gateway (единая точка входа)
- **8082** — Auth Service
- **8083** — Main Service
---

## Безопасность

1. Клиент отправляет логин/пароль → Gateway → Auth Service → получает токен
2. Клиент сохраняет токен
3. Клиент отправляет запрос с токеном → Gateway → Main Service
4. Main Service сам валидирует токен через Auth Service (Feign)

Gateway **проксирует** запросы, не вмешиваясь в аутентификацию.

---

## Зависимости проекта

| Сервис | Порт | Репозиторий |
|--------|------|-------------|
| **Gateway Service** | **8081** | **этот репозиторий** |
| Auth Service | 8082 | [Rutina Auth](https://github.com/ladatkoS/Rutina_Auth_Service.git) |
| Main Service | 8083 | [Rutina Main](https://github.com/ladatkoS/Rutina_Main_Service) |
| Android App | — | [Rutina Android](https://github.com/ladatkoS/Rutina_Android.git) |
| Neural Network | 8000 | [Rutina NN](https://github.com/AntonSlon/Rutina-neural-network.git) |

---
