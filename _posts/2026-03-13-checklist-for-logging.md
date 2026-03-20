---
layout: post
title: Контрольный список для логирования проекта
excerpt_separator: <!--more-->
categories:
  - Разработка
tags:
  - python
  - logging
---

Данный документ описывает единые стандарты использования `structlog` в Python-проектах.
Соблюдение правил обеспечивает консистентность JSON-логов, удобство поиска в ELK/Grafana Loki и чистоту кода.

<!--more-->

## 1. Конфигурация: "Золотой стандарт" процессоров

Основа правильных логов — настройка пайплайна процессоров. Конфигурация должна обеспечивать одинаковый набор полей независимо от фреймворка.

**Обязательные процессоры (порядок важен):**

1.  `structlog.contextvars.merge_contextvars` — внедряет контекст запроса/задачи.
2.  `structlog.processors.add_log_level` — добавляет уровень (`info`, `error`).
3.  `structlog.processors.StackInfoRenderer` — стек вызовов (если есть).
4.  `structlog.processors.format_exc_info` — beautify исключений.
5.  `structlog.processors.TimeStamper` — ISO-8601 UTC время.
6.  `structlog.processors.UUIDRenderer` — для request_id (если используется).

**Пример универсальной конфигурации:**

```python
import structlog

def setup_logging():
    common_processors = [
        structlog.contextvars.merge_contextvars,
        structlog.processors.add_log_level,
        structlog.processors.StackInfoRenderer(),
        structlog.processors.format_exc_info,
        structlog.processors.TimeStamper(fmt="iso", utc=True),
        structlog.processors.ProcessorFormatter.wrap_for_formatter,
    ]

    structlog.configure(
        processors=common_processors + [
            # В продакшене — JSON, локально — читаемый консольный вывод
            structlog.processors.JSONRenderer() if not settings.DEBUG
            else structlog.dev.ConsoleRenderer(colors=True),
        ],
        wrapper_class=structlog.make_filtering_bound_logger(logging.INFO),
        cache_logger_on_first_use=True,
    )
```

---

## 2. Уровни логирования и семантика

`structlog` использует стандартные уровни, но с акцентом на структурированные данные.

| Уровень     | Назначение                                                     | Пример события                                                         |
| ----------- | -------------------------------------------------------------- | ---------------------------------------------------------------------- |
| `debug`     | Технические детали для разработки. Отключается в проде.        | `sql_query_executed`, `cache_hit`                                      |
| `info`      | **Бизнес-события**. Успешное завершение операций.              | `user_registered`, `order_created`, `task_completed`                   |
| `warning`   | **Ожидаемые проблемы**. Некорректные данные, фоллбеки.         | `payment_gateway_timeout`, `invalid_input_data`, `deprecated_api_call` |
| `error`     | Неожиданные ошибки, требующие внимания, но не краш приложения. | `external_api_error`, `email_send_failed`                              |
| `exception` | Критические ошибки с трейсбеком (только внутри `except`).      | `unhandled_database_error`, `payment_processing_crash`                 |

---

## 3. Именование событий (Event Names)

Имя события (первый аргумент) — это главный поисковый ключ. Человекочитаемое описание лучше выносить в отдельное поле.

**Правила:**

1.  Формат: `snake_case`.
2.  Время: прошедшее (`user_created`, не `create_user`).
3.  Стиль: констатирующий факт.

```python
# Правильно
logger.info("user_created", user_id=user.id)
logger.warning("cache_miss", key=cache_key)

# Неправильно
logger.info("User Created")  # Пробелы, заглавные буквы
logger.info("create_user")   # Глагол в настоящем времени
logger.info("error")         # Слишком общее имя
```

---

## 4. Работа с контекстом (ContextVars)

Главная фича `structlog` — возможность привязать контекст ко всем логам в рамках запроса/задачи, не пробрасывая logger через аргументы функций.

### Правила работы с контекстом:

1.  **Инициализация:** В начале запроса (Middleware) очищайте и привязывайте базовые данные.
2.  **Накопление:** В сервисах добавляйте специфичные данные по мере углубления.
3.  **Очистка:** `structlog.contextvars.clear_contextvars()` вызывается в Middleware после завершения запроса.

### Паттерны интеграции (Django / FastAPI / FastStream)

**Django (Middleware):**

```python
import structlog

class RequestLoggingMiddleware:
    def __init__(self, get_response):
        self.get_response = get_response

    def __call__(self, request):
        structlog.contextvars.bind_contextvars(request_id=str(uuid.uuid4()), user_id=request.user.id)
        logger.info("http_request_started", path=request.path)

        try:
            response = self.get_response(request)
            logger.info("http_request_finished", status_code=response.status_code)
        except Exception:
            logger.exception("http_request_failed")
            raise
        finally:
            structlog.contextvars.clear_contextvars()

        return response
```

**FastAPI (Middleware):**

```python
@app.middleware("http")
async def logging_middleware(request: Request, call_next):
    structlog.contextvars.bind_contextvars(request_id=request.headers.get("X-Request-ID"))
    logger.info("http_request_started", method=request.method, path=request.url.path)

    try:
        response = await call_next(request)
        logger.info("http_request_finished", status_code=response.status_code)
    except Exception:
        logger.exception("http_request_failed")
        raise
    finally:
        structlog.contextvars.clear_contextvars()

    return response
```

**FastStream (Broker Middleware):**

```python
from faststream import BaseMiddleware

class LoggingMiddleware(BaseMiddleware):
    async def on_consume(self, msg):
        # Извлекаем ID из заголовков сообщения или генерируем
        structlog.contextvars.bind_contextvars(
            message_id=msg.message_id,
            conversation_id=msg.headers.get("correlation_id")
        )
        logger.info("message_received", queue=msg.queue)
        return await super().on_consume(msg)

    async def after_processed(self, msg):
        structlog.contextvars.clear_contextvars()
```

---

## 5. Безопасность: маскировка данных

Никогда не логируйте полные чувствительные данные.

**Реализация через Процессор:**
Лучший способ — добавить кастомный процессор в цепочку конфигурации, который маскирует данные автоматически.

```python
def mask_sensitive_data(logger, method_name, event_dict):
    """Маскирует email и токены перед выводом."""
    if "email" in event_dict:
        email = event_dict["email"]
        if isinstance(email, str) and "@" in email:
            name, domain = email.split("@")
            event_dict["email"] = f"{name[0]}***@{domain}"

    if "token" in event_dict:
        event_dict["token"] = "******"

    # Обрезаем длинные строки (например, JWT)
    if "jwt" in event_dict:
        event_dict["jwt"] = event_dict["jwt"][:10] + "..."

    return event_dict

# Добавить в common_processors:
# [mask_sensitive_data, ...]
```

**В коде:** Логируйте только префиксы или хеши.

```python
# Правильно
logger.debug("token_generated", token_prefix=token[:8])

# Неправильно
logger.info("user_logged_in", token=token)
```

---

## 6. Обработка исключений

Главное правило: `logger.exception` автоматически добавляет поле `exception` с трейсбеком.

**Анти-паттерны:**

1.  Логировать `error=str(e)` внутри `logger.exception` — это дублирование.
2.  Использовать `logger.error` внутри `except` без `exc_info=True` (лучше `logger.exception`).
3.  Логировать одно и то же исключение на разных уровнях стека.

**Правильный подход:**

```python
try:
    process_payment()
except PaymentGatewayError as e:
    # exception() сам добавит stacktrace и тип ошибки.
    # Добавляем только бизнес-контекст.
    logger.exception("payment_gateway_unavailable", gateway=e.gateway_name)
    raise  # Пробрасываем выше, если обработка не завершена
except ValueError as e:
    # Для ожидаемых ошибок часто достаточно warning
    logger.warning("invalid_payment_data", reason=str(e))
    return None
```

---

## 7. Универсальные стандарты имен полей

Для консистентности используйте единый словарь терминов.

| Поле          | Тип | Описание                                         |
| ------------- | --- | ------------------------------------------------ |
| `request_id`  | str | UUID запроса (для трейсинга)                     |
| `user_id`     | str | ID пользователя (не `user_pk`, не `uid`)         |
| `duration_ms` | int | Длительность операции в миллисекундах            |
| `status_code` | int | HTTP статус или код ошибки бизнес-логики         |
| `error_code`  | str | Внутренний код ошибки (e.g., `VALIDATION_ERROR`) |
| `reason`      | str | Человекочитаемая причина (кратко)                |

---

## Итоговый чек-лист для Code Review

1.  **Конфигурация:**
    - [ ] Используется `merge_contextvars` в процессорах.
    - [ ] В проде используется `JSONRenderer`, в деве — `ConsoleRenderer`.
    - [ ] Время логируется в UTC (`TimeStamper(fmt="iso")`).

2.  **Код:**
    - [ ] Импортирован `structlog`, логгер создан через `structlog.get_logger(__name__)`.
    - [ ] Имя события в `snake_case` прошедшего времени.
    - [ ] Используется `bind_contextvars` в начале запроса/задачи.
    - [ ] Чувствительные данные (пароли, токены) отсутствуют или маскируются.
    - [ ] В `logger.exception` нет ручного добавления `stacktrace` или `error=str(e)`.

3.  **Контекст:**
    - [ ] Есть `request_id` (или `correlation_id`) для связи логов микросервисов.
    - [ ] `clear_contextvars()` вызывается в `finally` блоке или middleware.
