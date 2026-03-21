---
layout: post
title: Контрольный список для тестирования проекта
excerpt_separator: <!--more-->
categories:
  - Разработка
tags:
  - python
  - testing
---

Основной принцип — не тестируем то, что уже тестировали, и не тестируем очевидную логику. По умолчанию используется функциональное (BDD) тестирование. Unit-тесты пишутся только для логики, которую невозможно или неэффективно проверить через BDD-сценарии.

<!--more-->

## 1. Граница между BDD и Unit-тестами

### Что покрываем BDD (Behavior Driven Development)

| Фреймворк      | Что тестируем                                                                    |
| -------------- | -------------------------------------------------------------------------------- |
| **Django**     | HTTP-эндпоинты (Views/ViewSets), интеграция ORM + API, пользовательские сценарии |
| **FastAPI**    | Endpoint handlers, dependency injection, интеграция с БД через repositories      |
| **FastStream** | Pub/Sub сценарии, обработка сообщений, интеграция брокеров                       |

**BDD покрывает:**

- Пользовательские сценарии end-to-end (создал платеж → получил результат)
- Интеграцию между слоями (API → Service → DB)
- Бизнес-требования в терминах пользователя/домена
- Контракты API (валидация входных/выходных данных)

### Что покрываем Unit-тестами

**Unit-тесты изолируют внутреннюю сложность:**

- Чистая бизнес-логика без I/O (domain services, entities)
- Асинхронные задачи вне HTTP-цикла (Celery, Dramatiq, FastStream workers)
- Сигналы, хуки, middleware (side effects)
- Сложные вычисления, алгоритмы, агрегации
- Трансформации данных (mappers, exporters)
- Тестирование миграций (обратная совместимость данных)

---

## 2. Что НЕ тестируем (Framework guarantees)

### Не тестируем стандартное поведение фреймворков

**Django:**

- `auto_now_add`, `auto_now` для дат
- `default=` для полей модели
- `on_delete=CASCADE/PROTECT/SET_NULL`
- Индексы БД (`db_index=True`, `unique=True`)
- Простые `__str__` без бизнес-логики
- Стандартные ModelAdmin конфигурации

**FastAPI:**

- Валидацию Pydantic (типы, constraints) — тестируется через BDD на уровне endpoint
- Dependency injection контейнер (resolving)
- Автоматическую генерацию OpenAPI схемы
- Статус-коды по умолчанию (200 OK для успеха)

**FastStream:**

- Декораторы `@broker.subscriber` / `@broker.publisher`
- Автоматическую сериализацию/десериализацию сообщений
- Управление connection pool к брокеру
- Retry-логику на уровне библиотеки (если не кастомная)

### Не тестируем очевидную логику

**Не тестируем:**

- Геттеры/сеттеры без логики (`@property` returning attribute)
- Простые присваивания (`self.status = new_status`)
- Проброс аргументов (`def func(a, b): return service.call(a, b)`)
- DTO/Schema классы без методов (только поля)
- Enum определения

---

## 3. Что тестируем (Risk-based approach)

### 🔴 Критичный приоритет (Ошибка = деньги/данные/безопасность)

| Категория               | Django                                      | FastAPI                         | FastStream           | Примеры                                         |
| ----------------------- | ------------------------------------------- | ------------------------------- | -------------------- | ----------------------------------------------- |
| **Финансовые операции** | Models `save()`, `clean()`                  | Service layer                   | Message handlers     | Расчет комиссий, конвертация валют, округление  |
| **Транзакции**          | `select_for_update()`, `transaction.atomic` | `async with db.begin()`         | —                    | Блокировки при списании средств                 |
| **Асинхронные задачи**  | Celery tasks                                | BackgroundTasks                 | `@broker.subscriber` | Отправка платежей, начисление процентов         |
| **Сигналы/Hooks**       | Django signals                              | Middleware, Dependencies        | Middleware brokers   | Side effects при создании пользователя          |
| **Внешние интеграции**  | API clients                                 | HTTP clients                    | External API calls   | Blockchain, платежные шлюзы, SMS                |
| **Безопасность**        | Permission classes                          | Dependencies `get_current_user` | Auth middleware      | Проверка прав, валидация токенов, rate limiting |

### 🟡 Высокий приоритет (Сложная логика)

| Категория            | Примеры                                                    |
| -------------------- | ---------------------------------------------------------- |
| **Агрегация данных** | Сложные SQL-запросы (Raw SQL, CTE), оптимизация N+1        |
| **Кэширование**      | Инвалидация кэша, cache-aside pattern, race conditions     |
| **Сложные условия**  | Многоуровневые if/else, state machines, workflow engines   |
| **Парсинг**          | Регулярные выражения, XML/CSV парсеры, нормализация данных |
| **Маппинги**         | Сложные преобразования Domain ↔ DTO ↔ ORM                  |

### 🟢 Низкий приоритет (Unit-тесты не нужны)

| Категория              | Обоснование                                  |
| ---------------------- | -------------------------------------------- |
| CRUD без бизнес-логики | Покрывается BDD интеграционными тестами      |
| Простые фильтры        | `.filter(status='active')` — проверяется BDD |
| Форматирование вывода  | `f"{amount} USD"` — тривиально               |
| Конфигурация           | Settings, URLs, простые фабрики              |

---

## 4. Тестирование миграций (django-test-migrations)

Миграции данных — это код, который выполняется один раз в production и не может быть откачен без бэкапа. **Тестируем сложные миграции обязательно.**

### Когда тестируем миграции

- 🔴 **Data migrations**: перенос данных между полями, разделение таблиц
- 🔴 **Rename + логика**: переименование + трансформация значений
- 🔴 **Удаление с миграцией**: удаление поля с сохранением данных в другое место
- 🟡 **Сложные RunSQL**: кастомные SQL операции
- 🟢 **Не тестируем**: Auto-generated миграции (AddField, CreateModel без defaults)

### Пример теста миграции

```python
# tests/test_migrations.py
import pytest
from django_test_migrations.contrib.unittest_case import MigratorTestCase

class TestPaymentStatusMigration(MigratorTestCase):
    """Тест миграции 0042: разделение статуса 'pending' на 'pending_card' и 'pending_crypto'."""

    migrate_from = ('payment', '0041_auto_20240320')
    migrate_to = ('payment', '0042_split_pending_status')

    def prepare(self):
        """Создаем данные в старой схеме."""
        Payment = self.old_state.apps.get_model('payment', 'Payment')
        self.payment_id = Payment.objects.create(
            amount=100,
            status='pending',  # Старый статус
            payment_type='card'
        ).id

    def test_migration_splits_pending_status(self):
        """Проверяем корректность разделения статусов."""
        Payment = self.new_state.apps.get_model('payment', 'Payment')
        payment = Payment.objects.get(id=self.payment_id)

        # Логика миграции: card → pending_card, crypto → pending_crypto
        assert payment.status == 'pending_card'
```

### Критические проверки в миграциях

- [ ] **Обратная совместимость**: старый код читает новые данные корректно
- [ ] **Nullable поля**: корректная обработка NULL значений
- [ ] **Default значения**: логика применения defaults к существующим записям
- [ ] **Индексы**: проверка unique constraints после миграции (граничные случаи)

---

## 5. Чек-лист перед написанием Unit-теста

### Вопросы для самопроверки

- [ ] **Этот тест можно заменить BDD-сценарием?**
  - Если да → пишем BDD (интеграционный), не unit
  - Если нет → продолжаем

- [ ] **Это стандартное поведение фреймворка?**
  - Django/FastAPI/FastStream гарантирует это без моего кода?
  - Если да → не тестируем

- [ ] **Ошибка в этом коде стоит денег/данных/репутации?**
  - Финансы, безопасность, compliance?
  - Если да → тестируем обязательно (приоритет 🔴)

- [ ] **Здесь есть нетривиальная логика?**
  - Условия, циклы, расчеты, state machine?
  - Regex, парсинг, сложная агрегация?
  - Если да → тестируем (приоритет 🟡)

- [ ] **Этот код уже покрыт другими тестами?**
  - Другой unit-тест проверяет эту же ветку?
  - BDD сценарий проходит через эту логику?
  - Если да → не дублируем

- [ ] **Это миграция данных с трансформацией?**
  - Если да → тестируем через django-test-migrations

---

## 6. Структура и паттерны тестов

### Паттерн Arrange-Act-Assert (AAA)

```python
def test_calculate_commission_with_tiered_rates():
    """Тест: расчет комиссии по прогрессивной шкале."""
    # Arrange: подготовка изолированных данных
    transaction = TransactionFactory(amount=10000, currency="USD")
    calculator = CommissionCalculator(
        tiers=[(0, 1000, 0.01), (1000, 5000, 0.008), (5000, None, 0.005)]
    )

    # Act: вызов тестируемой логики (без I/O)
    result = calculator.calculate(transaction)

    # Assert: проверка результата и side effects
    assert result.base_amount == 10000
    assert result.fee == 60  # 1000*0.01 + 4000*0.008 + 5000*0.005
    assert result.net_amount == 9940
```

### Фреймворк-специфичные примеры

**Django (Models + Tasks):**

```python
@pytest.mark.django_db
def test_payment_save_calculates_exchange_rate():
    """Тест: сохранение платежа пересчитывает курс если валюта отличается."""
    # Arrange
    account = Account.objects.create(currency='EUR', balance=1000)

    # Act
    payment = Payment.objects.create(
        account=account,
        amount=100,
        currency='USD'  # Отличная от счета валюта
    )

    # Assert
    assert payment.amount_eur == 92  # Курс 0.92
    assert payment.exchange_rate == 0.92
```

**FastAPI (Services):**

```python
async def test_payment_service_with_currency_conversion():
    """Тест: сервис платежей с моком внешнего API курсов."""
    # Arrange
    mock_rates = AsyncMock()
    mock_rates.get_rate.return_value = Decimal('0.92')

    service = PaymentService(rate_provider=mock_rates)
    dto = PaymentCreate(amount=Decimal('100'), currency='USD', target_currency='EUR')

    # Act
    result = await service.process(dto)

    # Assert
    assert result.converted_amount == Decimal('92.00')
    mock_rates.get_rate.assert_called_once_with('USD', 'EUR')
```

**FastStream (Handlers):**

```python
async def test_payment_processed_handler_updates_balance():
    """Тест: обработчик сообщений обновляет баланс с блокировкой."""
    # Arrange
    msg = PaymentProcessedMessage(payment_id='uuid-123', amount=100, user_id=1)
    mock_repo = Mock()
    mock_repo.update_balance.return_value = AsyncMock()

    handler = PaymentHandler(repository=mock_repo)

    # Act
    await handler.on_payment_processed(msg)

    # Assert
    mock_repo.update_balance.assert_called_once_with(
        user_id=1,
        amount=100,
        operation='credit'
    )
```

---

## 7. Матрица покрытия по компонентам

| Компонент              | Django         | FastAPI             | FastStream      | Почему тестируем             | Приоритет |
| ---------------------- | -------------- | ------------------- | --------------- | ---------------------------- | --------- |
| **Миграции данных**    | ✅             | N/A                 | N/A             | Необратимые изменения        | 🔴        |
| **Tasks/Workers**      | Celery         | BackgroundTasks     | @subscriber     | Асинхронность, потеря данных | 🔴        |
| **Signals/Middleware** | Django signals | Dependencies        | Middleware      | Side effects                 | 🔴        |
| **Services**           | Service layer  | Service layer       | Handler logic   | Бизнес-логика                | 🔴        |
| **Domain Models**      | Model methods  | Pydantic validators | Dataclasses     | Инварианты                   | 🟡        |
| **Repositories**       | ORM Querysets  | SQLAlchemy          | —               | Сложные запросы              | 🟡        |
| **Exporters/Reports**  | Pandas/CSV     | Pandas/CSV          | —               | Агрегация                    | 🟡        |
| **Utils/Helpers**      | Regex, parsers | Regex, parsers      | Message parsers | Сложность                    | 🟡        |
| **Schema definitions** | ❌             | ❌                  | ❌              | Фреймворк валидирует         | 🟢        |
| **Config/Settings**    | ❌             | ❌                  | ❌              | Нет логики                   | 🟢        |

---

## 8. Критерии качества Unit-теста

### Обязательные требования

- [ ] **Изоляция**: тест не зависит от других тестов (порядок не важен)
- [ ] **Детерминизм**: одинаковый результат при каждом запуске (нет случайных данных)
- [ ] **Скорость**: < 50ms на тест (нет реальных I/O, sleep, HTTP вызовов)
- [ ] **Читаемость**: понятно что тестируется без чтения исходного кода
- [ ] **Фокус**: один тест = одно поведение (один assert или логическая группа)
- [ ] **Mocking**: внешние зависимости (API, БД, брокер) замоканы

### Антипаттерны (переписать или удалить)

- [ ] **Зависимость от порядка**: использует данные из другого теста
- [ ] **Интеграция в unit**: реальные HTTP запросы, БД (кроме миграций), файловая система
- [ ] **Сложная логика в тесте**: if/else, циклы, try-except в теле теста
- [ ] **Дублирование**: проверяет то же самое, что другие тесты
- [ ] **Непонятные данные**: магические числа без контекста (что значит `amount=42`?)

---

## Примеры миграций, которые НУЖНО тестировать

### 1. Data Migration с бизнес-логикой

```python
# migrations/0042_migrate_payment_status.py
def split_pending_status(apps, schema_editor):
    Payment = apps.get_model('payment', 'Payment')
    for payment in Payment.objects.filter(status='pending'):
        # Бизнес-логика: определяем новый статус по полю method
        if payment.method in ['card', 'bank_transfer']:
            payment.status = 'pending_fiat'
        else:
            payment.status = 'pending_crypto'
        payment.save(update_fields=['status'])
```

**Почему тестируем**: Ошибка в логике разделения приведет к неконсистентности данных.

### 2. Обратная миграция (Backward)

```python
def combine_status(apps, schema_editor):
    """Обратная миграция: объединяем статусы обратно."""
    Payment = apps.get_model('payment', 'Payment')
    Payment.objects.filter(
        status__in=['pending_fiat', 'pending_crypto']
    ).update(status='pending')
```

**Почему тестируем**: Rollback миграции должен восстанавливать консистентное состояние.

### 3. Миграция с Default значением

```python
def set_default_category(apps, schema_editor):
    Payment = apps.get_model('payment', 'Payment')
    # Нельзя использовать .update() из-за логики определения категории
    for payment in Payment.objects.filter(category__isnull=True):
        payment.category = detect_category(payment.description)  # NLP/SQL логика
        payment.save()
```

**Почему тестируем**: Логика `detect_category` может меняться, миграция должна быть детерминирована.

---

## Итоговые принципы

1. **BDD first**: 80% покрытия через функциональные тесты, 20% — критичная unit-логика
2. **Risk-based**: Тестируем не "все подряд", а то, где ошибка дорога
3. **Framework guarantees**: Не тестируем то, что гарантирует фреймворк (валидаторы, ORM базовое поведение)
4. **Миграции — это код**: Data migrations тестируем как бизнес-логику
5. **Изоляция**: Unit-тесты не трогают БД (кроме миграций), сеть, файловую систему
