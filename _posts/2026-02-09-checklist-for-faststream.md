---
layout: post
title: Чек-лист для Fastapi
excerpt_separator: <!--more-->
categories:
  - Разработка
tags:
  - python
  - testing
---

Документ описывает стандарты разработки с использованием фреймворка FastStream. Основной фокус — надежность обработки сообщений, идемпотентность, чистая архитектура, правильная работа с контекстом и операционная устойчивость.

<!--more-->

## 1. Архитектура: Handlers & Services

Подписчики (Consumers) должны быть "тонкими". Их задача — принять сообщение, извлечь данные и делегировать выполнение слою логики.

| Слой               | Ответственность                                                                       |
| ------------------ | ------------------------------------------------------------------------------------- |
| **Router/Handler** | Прием сообщения, валидация Pydantic, вызов сервиса, возврат подтверждения (Ack/Nack). |
| **Service**        | Бизнес-логика, идемпотентность, работа с БД, вызов внешних API.                       |
| **Publisher**      | Отправка результатов/событий в другие топики (через DI или декларированные объекты).  |

### Правильно

```python
# consumers/user_consumer.py
from faststream import FastStream
from users.services import UserService

@broker.subscriber("user_created")
async def handle_user_created(
    msg: UserCreatedSchema,
    service: UserService = Depends()
):
    # Хендлер только координирует
    await service.process_new_user(msg)

# users/services.py
class UserService:
    def __init__(self, user_publisher: Publisher):
        self.publisher = user_publisher

    async def process_new_user(self, msg: UserCreatedSchema):
        # Проверка на идемпотентность
        if await self.repo.exists(msg.event_id):
            logger.warning("event_already_processed", event_id=msg.event_id)
            return

        # Бизнес-логика
        user = await self.repo.create_user(msg.user_data)

        # Публикация через внедренный Publisher
        await self.publisher.publish(
            UserProcessedEvent(user_id=user.id)
        )
```

### Неправильно

```python
@broker.subscriber("user_created")
async def handle_user_created(msg: dict):
    # Вся логика в хендлере
    user = await User.objects.create(**msg)
    # Прямая отправка email без обработки ошибок
    await send_email(user.email)
```

---

## 2. Сериализация и Валидация (Pydantic)

FastStream использует Pydantic для автоматической валидации сообщений. Никогда не используйте `dict` или `Any` для тела сообщения, если схема известна. Заголовки (headers) также могут проходить валидацию, если они критичны для бизнес-логики.

### Правильно

Явная типизация обеспечивает автоматическую валидацию и документацию (AsyncAPI).

```python
class OrderMessage(BaseModel):
    order_id: int
    items: List[OrderItem]
    created_at: datetime

@broker.subscriber("orders")
async def process_order(msg: OrderMessage):
    # Гарантированно валидные данные
    await service.process(msg.order_id)
```

### Неправильно

```python
# Нет валидации, риск получения некорректных данных
@broker.subscriber("orders")
async def process_order(msg: dict):
    order_id = msg.get("id")  # Может быть None или другого типа
```

---

## 3. Обработка ошибок и Retries

Это критическая часть для очередей. Неправильная обработка может забить очередь "ядовитыми" сообщениями (Poison Pills) или вызвать бесконечный цикл.

### Правила:

1.  Используйте `@broker.subscriber(retry=...)` для ожидаемых сетевых ошибок.
2.  Валидируйте данные до начала логики.
3.  Для невосстановимых ошибок (ошибка данных) — логируйте и ack-айте (подтверждайте), отправляя в Dead Letter Queue (DLQ), иначе сообщение будет приходить бесконечно.
4.  Учитывайте, что механизм retry требует корректной настройки параллелизма (см. раздел 7).

### Использование Retry

```python
from faststream import FastStream
from faststream.exceptions import RejectMessage

# Повторить 3 раза с экспоненциальной задержкой при ошибках подключения
@broker.subscriber(
    "notifications",
    retry=RetryPolicy(max_attempts=3, delay=1, max_delay=10)
)
async def send_notification(msg: Notification):
    try:
        await external_api.send(msg)
    except ExternalAPIUnavailable:
        # Этот exception триггерит механизм retry
        raise
    except InvalidPhoneNumber:
        # Эту ошибку ретраить бесполезно, подтверждаем обработку, но шлем в DLQ
        logger.error("invalid_phone", msg=msg)
        raise RejectMessage()  # Не отправлять в retry, отбросить (или отправить в DLQ)
```

---

## 4. Идемпотентность

Потребители должны быть готовы к тому, что одно и то же сообщение может прийти дважды (at-least-once delivery).

**Реализация:**

- Используйте уникальный `event_id` (или `message_id`) из заголовков сообщения.
- Проверяйте, была ли уже обработана эта операция в БД (таблица processed_events или флаг в сущности).

```python
@broker.subscriber("payments")
async def handle_payment(msg: PaymentSchema, db: Session = Depends(get_db)):
    # Проверка идемпотентности
    if await db.get(ProcessedEvent, msg.event_id):
        logger.info("skipping_duplicate_event", event_id=msg.event_id)
        return  # FastStream автоматически Ack-нет сообщение

    # Обработка...
    await db.save(ProcessedEvent(id=msg.event_id))
```

---

## 5. Контекст и Middlewares

Для связного логирования в распределенной системе необходимо передавать контекст (trace_id, user_id) между сервисами через заголовки сообщений.

### Правильно

Middleware извлекает заголовки и устанавливает их в contextvars до вызова хендлера.

```python
class ContextMiddleware:
    async def on_receive(self, message: RMQMessage, **kwargs):
        # Извлекаем trace_id из заголовков брокера
        trace_id = message.headers.get("x-trace-id", "unknown")
        structlog.contextvars.bind_contextvars(trace_id=trace_id)
        return message

# Подключение middleware
broker = RabbitBroker("amqp://guest:guest@localhost:5672/")
broker.add_middleware(ContextMiddleware())

@broker.subscriber("orders")
async def handle(message: Order, logger=Depends(get_logger)):
    # logger уже содержит trace_id из middleware
    logger.info("order_received", order_id=message.id)
```

### Неправильно

Игнорирование заголовков приводит к потере связности между логами разных сервисов.

---

## 6. Жизненный цикл и Graceful Shutdown

Корректное завершение работы — критический аспект, предотвращающий потерю сообщений и нарушение консистентности данных.

### Правильно

Настройка таймаута на завершение и проверка health.

```python
from faststream import FastStream

# shutdown_timeout должен превышать максимальное время обработки сообщения
app = FastStream(broker, shutdown_timeout=60.0)

# Healthchecks для оркестрации (Kubernetes)
@app.get("/health/live")
async def liveness():
    # Проверка, что приложение запущено и не в deadlock
    return {"status": "ok"}

@app.get("/health/ready")
async def readiness():
    # Проверка, что брокер доступен и приложение готово принимать сообщения
    if await broker.health():
        return {"status": "ok"}
    return {"status": "unavailable"}, 503
```

### Чек-лист

- [ ] **shutdown_timeout** настроен и превышает максимальное время обработки.
- [ ] **liveness** probe проверяет, что приложение живо.
- [ ] **readiness** probe проверяет доступность брокера.

---

## 7. Тонкая настройка Consumer (Prefetch & Parallel)

Контроль параллелизма и предварительной выборки необходим для предотвращения перегрузки сервиса и обеспечения стабильной работы под нагрузкой.

### Правила:

1.  **Prefetch:** Установите разумный лимит `prefetch_count` (обычно 1–5 для долгих операций), чтобы избежать перегрузки пула соединений с БД и накопления большого количества сообщений в памяти.
2.  **Concurrency:** Настройте количество воркеров через `broker.subscriber(consumers=...)` для горизонтального масштабирования внутри одного инстанса.

```python
# Для RabbitMQ
@broker.subscriber(
    "orders",
    prefetch_count=5,      # Не загружать больше 5 сообщений одновременно
    consumers=2            # Запустить 2 воркера внутри процесса
)
async def process_order(msg: Order):
    await service.process(msg)
```

```python
# Для Kafka (важен порядок в партиции)
@broker.subscriber(
    "orders",
    batch=True,            # Обработка батчами для повышения пропускной способности
    max_records=100        # Максимум 100 записей в батче
)
async def process_orders(msgs: List[Order]):
    await service.process_batch(msgs)
```

---

## 8. Логирование и Трейсинг

Интеграция с `structlog` обязательна для микросервисов. FastStream позволяет легко внедрить свой логгер.

```python
# core/logging.py
import structlog

def setup_faststream_logging():
    # Настройка FastStream на использование structlog
    pass

# В middleware (см. раздел 5) добавляем trace_id
# При обработке сообщения извлекаем headers для trace_id
@broker.subscriber("orders")
async def handle(message: Order, logger=Depends(get_logger)):
    # logger уже содержит contextvars из middleware
    logger.info("order_received", order_id=message.id)
```

---

## 9. Тестирование

FastStream предоставляет отличные инструменты для тестирования без реального брокера через `TestBroker`.

### Правильно

```python
import pytest
from faststream.testing import TestBroker

@pytest.mark.asyncio
async def test_user_creation():
    # Мокаем брокера
    async with TestBroker(broker) as test_broker:
        # Публикуем тестовое сообщение
        await test_broker.publish(
            {"user_id": 1, "email": "test@test.com"},
            "user_created"
        )

        # Проверяем, что был вызов публикации в другой топик
        # или проверяем состояние БД
```

---

## Чек-лист для Code Review (FastStream)

Перед мержем проверьте:

- [ ] **Pydantic Models:** Используются типизированные схемы сообщений, а не `dict`.
- [ ] **Идемпотентность:** Обработана ситуация повторного получения сообщения.
- [ ] **Error Handling:** Не проглатываются исключения, которые должны привести к retry.
- [ ] **Poison Pill:** Невалидные данные (которые нельзя обработать) не блокируют очередь (используется DLQ или Reject с логированием).
- [ ] **DI:** Сервисы и сессии БД внедряются через Depends, не создаются глобально.
- [ ] **Async:** Нет синхронных блокирующих вызовов внутри async-хендлеров.
- [ ] **Publishing:** Ответные сообщения публикуются через декларированные объекты `Publisher`, а не прямой вызов `broker.publish`.
- [ ] **Graceful Shutdown:** Настроен `shutdown_timeout`, превышающий максимальное время обработки.
- [ ] **Health Checks:** Реализованы liveness и readiness probes.
- [ ] **Prefetch:** Установлен разумный `prefetch_count` для защиты от перегрузки.
- [ ] **Context Propagation:** Middleware передает `trace_id` и другие контекстные данные.
