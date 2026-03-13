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

## 1. Стандарт уровней логирования

Используйте уровни логирования в соответствии с их назначением:

| Уровень | Когда использовать | Пример |
|---------|-------------------|--------|
| `logger.debug()` | Отладочная информация, промежуточные вычисления | Генерация токенов, SQL-запросы в dev-режиме |
| `logger.info()` | Успешные бизнес-события | `user_created`, `payment_confirmed`, `domain_verified` |
| `logger.warning()` | Ожидаемые проблемы, ошибки валидации | `validation_failed`, `retry_attempt`, `token_expired` |
| `logger.error()` | Неожиданные ошибки, но обработанные | Сбой внешнего сервиса с fallback-логикой |
| `logger.exception()` | **Только в `except`-блоках** с stacktrace | Любое исключение, требующее анализа |

### ✅ Правильно

```python
logger.debug("login_link_generated", uid_prefix=uid[:8], token_prefix=token[:4])
logger.info("user_created", email=email)
logger.warning("domain_validation_failed", reason="empty_domain")
logger.error("email_backend_rejected")
logger.exception("email_send_failed")  # внутри except
```

### ❌ Неправильно

```python
logger.info("evm_address_validation_failed")  # ← Должен быть warning!
logger.exception("error", error=str(e))  # ← вне except-блока
```

---

## 2. Единый формат имён событий

Имена событий должны быть в `snake_case` с глаголом в **прошедшем времени**.

### ✅ Правильно

```python
"user_created"
"payment_processed"
"domain_verified"
"validation_failed"
"link_generated"
"token_expired"
```

### ❌ Неправильно

```python
"userCreated"           # camelCase
"create_user"           # глагол в настоящем времени
"USER_CREATED"          # CONSTANT_CASE
"user-create"           # kebab-case
```

---

## 3. Стандартизация контекстных полей

Используйте единые имена для одинаковых сущностей во всём проекте.

### Стандартные имена полей

| Поле | Тип | Описание |
|------|-----|----------|
| `user_id` | UUID | ID пользователя (всегда `user_id`, не `user_pk`) |
| `account_id` | UUID | ID аккаунта |
| `domain` | str | Домен (без префиксов http/https) |
| `amount` | str | Сумма (Decimal → str для JSON) |
| `reason` | str | Причина ошибки |
| `error` | str | Текст ошибки (только в `logger.exception()`) |

### ✅ Правильно

```python
logger.info(
    "payment_processed",
    user_id=user.id,
    account_id=account.id,
    amount=str(amount),
    reason="invalid_format",
)
```

### ❌ Неправильно

```python
logger.info("error", user=user, pk=123, data=some_dict)  # ← слишком общо
logger.info("payment", user_pk=pk, sum=amount)  # ← неправильные имена
```

---

## 4. Обработка исключений с контекстом

### Общие правила

1. **`logger.exception()`** использует **только внутри `except`-блоков**
2. **Не дублируйте `error` и `error_type`** — structlog добавит stacktrace автоматически
3. **Не логируйте чувствительные данные** (полные токены, пароли, email без маскировки)

### ✅ Правильно

```python
# Поиск пользователя с обработкой ошибок
try:
    user = User.objects.get(pk=uid)
except (TypeError, ValueError, OverflowError, User.DoesNotExist) as e:
    logger.warning("invalid_uid", uidb64_prefix=uidb64[:8])  # ← только префикс
    msg = "Invalid user identification"
    raise UserCreationError(msg) from e

# Отправка email с exception()
try:
    sent = self._do_send_email(email, login_url)
except Exception as e:
    logger.exception("email_send_failed")  # ← email уже в contextvars
    raise UserCreationError("Failed to send login email") from e
```

### ❌ Неправильно

```python
# Избыточные поля в exception()
logger.exception(
    "safe_browsing_request_failed",
    domain=domain,           # ← уже в contextvars
    error_type=type(e).__name__,  # ← избыточно
    error_message=str(e),    # ← избыточно
)

# Логирование вне except
logger.exception("some_error", error=str(e))  # ← нет stacktrace!
```

---

## 5. Безопасность: маскировка чувствительных данных

### Автоматическая маскировка

В проекте настроен `mask_email_processor`, который автоматически маскирует email:

```python
logger.info("user_found", email="john.doe@example.com")
# В логе: "email": "j***@example.com"
```

### Ручная маскировка токенов и идентификаторов

**Никогда не логируйте полные токены, UID или секреты!**

### ✅ Правильно

```python
logger.debug(
    "login_link_generated",
    uid_prefix=uid[:8],      # ← только префикс
    token_prefix=token[:4],  # ← только префикс
)

logger.warning("invalid_uid", uidb64_prefix=uidb64[:8])
```

### ❌ Неправильно

```python
logger.debug("login_link", link=login_url)  # ← содержит полный токен!
logger.info("user_uid", uid=uid)  # ← полный UID
logger.debug("token", token=token)  # ← полный токен
```

---

## 6. Контекстные переменные (structlog.contextvars)

Используйте `structlog.contextvars.bind_contextvars()` для автоматического добавления
контекста ко всем последующим логам в рамках запроса/операции.

### ✅ Правильно

```python
# В начале обработки запроса
structlog.contextvars.bind_contextvars(email=email)
# ... далее все логи автоматически получат email
logger.info("auth_link_requested")  # ← будет с email в контексте
logger.info("auth_link_sent")       # ← тоже с email

# В сервисе верификации
structlog.contextvars.bind_contextvars(domain=domain, verification_uuid_prefix=verification_uuid[:8])
logger.info("txt_record_found")     # ← с domain и verification_uuid_prefix
logger.warning("verification_mismatch")  # ← тоже с контекстом
```

### Где добавлять contextvars

| Место | Какие переменные добавлять |
|-------|---------------------------|
| `AuthorizationView.form_valid()` | `email` |
| `DomainVerificationService.verify_domain()` | `raw_domain`, `clean_domain` |
| `DomainVerificationService.check_txt_record()` | `domain`, `verification_uuid_prefix` |
| `validate_safe_domain()` | `domain` |
| `AccountMetricsView.get()` | `account_id`, `account_domain` |

---

## 7. Кастомные исключения для сервисного слоя

Создавайте иерархию кастомных исключений для каждого сервиса.

### Пример структуры

```python
# landing/services.py
class UserCreationError(Exception):
    """Базовое исключение для ошибок создания и управления пользователями."""
    pass

# dashboard/services.py
class DomainVerificationError(Exception):
    """Базовое исключение для ошибок верификации домена."""
    pass

# payment/services.py (требует создания)
class PaymentProcessingError(Exception):
    """Базовое исключение для ошибок обработки платежей."""
    pass
```

### Использование

```python
# ✅ Правильно
try:
    result = DomainVerificationService.verify_domain(raw_domain, verification_uuid)
except DomainVerificationError as e:
    logger.warning("domain_verification_failed", reason=str(e))
    messages.error(request, "Domain verification failed")

# ❌ Неправильно
try:
    result = DomainVerificationService.verify_domain(raw_domain, verification_uuid)
except Exception as e:
    logger.exception("error")  # ← слишком общо
    raise
```

---

## Шаблон для нового кода

```python
import structlog

logger = structlog.get_logger(__name__)


def process_payment(payment_id: str, amount: Decimal) -> bool:
    """Обработка платежа.
    
    Args:
        payment_id: ID платежа.
        amount: Сумма платежа.
    
    Returns:
        True если платеж успешен.
    
    Raises:
        PaymentError: При ошибке обработки.
    """
    # 1. Добавляем контекст в начале
    structlog.contextvars.bind_contextvars(payment_id=payment_id)
    logger.info("payment_processing_started", amount=str(amount))
    
    # 2. Проверяем существование
    try:
        payment = Payment.objects.get(id=payment_id)
    except Payment.DoesNotExist:
        logger.warning("payment_not_found")
        raise PaymentError("Payment not found")
    
    # 3. Валидация бизнес-логики
    if amount <= 0:
        logger.warning("payment_invalid_amount", amount=str(amount))
        raise PaymentError("Invalid amount")
    
    # 4. Обработка с exception() только в except
    try:
        payment.process()
    except Exception as e:
        logger.exception("payment_processing_failed")
        raise PaymentError("Processing error") from e
    
    # 5. Успешное завершение
    logger.info("payment_processed_successfully")
    return True
```

---

## Чек-лист для код-ревью

Перед мержем проверьте:

- [ ] Используется правильный уровень логирования (warning для ошибок валидации)
- [ ] Имя события в `snake_case` с глаголом в прошедшем времени
- [ ] Контекстные переменные добавлены через `bind_contextvars()`
- [ ] В `logger.exception()` нет дублирующих полей (`error`, `error_type`)
- [ ] Чувствительные данные замаскированы (email, токены, UID)
- [ ] Кастомные исключения используются вместо общего `Exception`
- [ ] Нет полных токенов/секретов в логах (только префиксы)

---

## Инструменты

### Проверка через ruff

```bash
# Проверка логов на наличие чувствительных данных
ruff check --select LOG .
```

### Локальное тестирование

В DEBUG-режиме логи выводятся в консоль с цветовой разметкой:

```bash
docker compose up  # логи в stdout
```

В production логи выводятся в JSON-формате для сбора ELK/Graylog:

```python
# core/settings/logging.py
if DEBUG:
    structlog_processors.append(structlog.dev.ConsoleRenderer())
else:
    structlog_processors.append(structlog.processors.JSONRenderer())
```
