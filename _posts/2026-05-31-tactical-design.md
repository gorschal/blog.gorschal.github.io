---
layout: post
title: Тактическое проектирование (Tactical Design) в DDD
excerpt_separator: <!--more-->
categories:
  - Разработка
  - Управление
tags:
  - design
  - architecture
  - ddd
---

[Начало и оглавление здесь](https://blog.gorschal.com/domain-driven-design.html)

**Тактическое проектирование (Tactical Design)** — это набор строительных блоков Domain-Driven Design, из которых собирается сама доменная модель: сущности (`Entities`), объекты-значения (`Value Objects`), агрегаты (`Aggregates`), репозитории (`Repositories`), сервисы домена (`Domain Services`), фабрики (`Factories`) и события домена (`Domain Events`).

<!--more-->

Отдельные блоки мы уже разбирали [в статье про Domain](https://blog.gorschal.com/domain.html). Здесь же посмотрим на них как на единую систему: как они взаимодействуют, где проходят границы ответственности и как не превратить модель в анемичную.

## Кратко о каждом блоке

| Блок | Ответственность | Признак |
|------|-----------------|---------|
| **Сущность (Entity)** | Объект с идентичностью и жизненным циклом | Есть `id`, состояние меняется |
| **Объект-значение (Value Object)** | Неизменяемое описание свойств | Нет `id`, сравнивается по атрибутам |
| **Агрегат (Aggregate)** | Кластер объектов с единым корнем | Изменения идут только через корень |
| **Репозиторий (Repository)** | Доступ к агрегатам, скрытие хранилища | Интерфейс в домене, реализация в инфраструктуре |
| **Сервис домена (Domain Service)** | Логика, не принадлежащая одному объекту | Без состояния, работает с несколькими агрегатами |
| **Фабрика (Factory)** | Создание сложных объектов/агрегатов | Инкапсулирует инварианты создания |
| **Событие домена (Domain Event)** | Факт, который произошёл в домене | Именуется в прошедшем времени (`OrderPlaced`) |

## Полный пример: как блоки работают вместе

Соберём систему заказов, где каждый блок выполняет свою роль.

### 1. Объекты-значения

```python
from dataclasses import dataclass

@dataclass(frozen=True)
class Money:
    amount: int  # в минимальных денежных единицах, чтобы не было float-ошибок
    currency: str = "RUB"

    def __add__(self, other: "Money") -> "Money":
        if self.currency != other.currency:
            raise ValueError("Currencies don't match")
        return Money(self.amount + other.amount, self.currency)
```

Деньги — классический объект-значение. Он неизменяемый, не имеет идентичности и защищает домен от ошибок округления `float`.

### 2. Сущность и агрегат

Корень агрегата `Order` управляет своими `OrderItem` (внутренними объектами) и поддерживает инварианты — правила, которые должны выполняться всегда:

```python
from uuid import UUID, uuid4

class OrderItem:
    def __init__(self, product_id: UUID, product_name: str, unit_price: Money, quantity: int):
        if quantity <= 0:
            raise ValueError("Quantity must be positive")
        self.product_id = product_id
        self.product_name = product_name
        self.unit_price = unit_price
        self.quantity = quantity

    @property
    def total_price(self) -> Money:
        return Money(self.unit_price.amount * self.quantity, self.unit_price.currency)


class Order:
    def __init__(self, order_id: UUID, customer_id: UUID):
        self.order_id = order_id
        self.customer_id = customer_id
        self.items: list[OrderItem] = []
        self.status = "created"
        self.events: list[DomainEvent] = []

    def add_item(self, product_id: UUID, product_name: str, unit_price: Money, quantity: int):
        if self.status != "created":
            raise ValueError("Cannot modify a non-created order")
        self.items.append(OrderItem(product_id, product_name, unit_price, quantity))

    def confirm(self) -> None:
        if self.status != "created":
            raise ValueError("Order is already confirmed or cancelled")
        if not self.items:
            raise ValueError("Cannot confirm an empty order")
        self.status = "confirmed"
        self.events.append(OrderPlaced(self.order_id, self.customer_id, self.total_price))

    def cancel(self) -> None:
        if self.status != "created":
            raise ValueError("Order is already confirmed or cancelled")
        self.status = "cancelled"
        self.events.append(OrderCancelled(self.order_id))

    @property
    def total_price(self) -> Money:
        return sum((item.total_price for item in self.items), start=Money(0))

    def collect_events(self) -> list["DomainEvent"]:
        events = self.events.copy()
        self.events.clear()
        return events
```

Здесь видны ключевые правила тактического проектирования:

- Все изменения агрегата идут **только через корень** (`Order`), а не через его внутренние объекты.
- **Инварианты** («нельзя подтвердить пустой заказ», «нельзя менять подтверждённый заказ») защищены на уровне самого агрегата.
- Инварианты проверяются **внутри агрегата**, а не в сервисах или контроллерах.

### 3. События домена

```python
from datetime import datetime, timezone
from dataclasses import dataclass

@dataclass
class DomainEvent:
    occurred_on: datetime = datetime.now(timezone.utc)


@dataclass
class OrderPlaced(DomainEvent):
    order_id: UUID
    customer_id: UUID
    total: Money


@dataclass
class OrderCancelled(DomainEvent):
    order_id: UUID
```

События нужны, чтобы другие части системы (уведомления, склад, отчёты) реагировали на изменения домена, не нарушая его границы. События собираются в агрегате и публикуются после сохранения (обычно через `collect_events()`).

### 4. Репозиторий

```python
from abc import ABC, abstractmethod

class OrderRepository(ABC):
    @abstractmethod
    def save(self, order: Order) -> None: ...

    @abstractmethod
    def find_by_id(self, order_id: UUID) -> Order | None: ...

    @abstractmethod
    def find_by_customer(self, customer_id: UUID) -> list[Order]: ...
```

Интерфейс живёт в доменном слое, а реализация (SQLAlchemy, Django ORM, in-memory) — в инфраструктурном. Домен не знает про БД вообще.

```python
class InMemoryOrderRepository(OrderRepository):
    def __init__(self):
        self._orders: dict[UUID, Order] = {}

    def save(self, order: Order) -> None:
        self._orders[order.order_id] = order

    def find_by_id(self, order_id: UUID) -> Order | None:
        return self._orders.get(order_id)

    def find_by_customer(self, customer_id: UUID) -> list[Order]:
        return [o for o in self._orders.values() if o.customer_id == customer_id]
```

### 5. Сервис домена

Сервис домена нужен, когда операция затрагивает несколько агрегатов и не принадлежит ни одному из них:

```python
class PaymentService:
    def charge(self, order: Order, payment_method) -> None:
        # Логика списания средств
        pass


class OrderService:
    def __init__(self, order_repository: OrderRepository, payment_service: PaymentService):
        self.order_repository = order_repository
        self.payment_service = payment_service

    def place_order(self, order_id: UUID) -> None:
        order = self.order_repository.find_by_id(order_id)
        if not order:
            raise ValueError("Order not found")

        self.payment_service.charge(order, order.customer_id)
        order.confirm()
        self.order_repository.save(order)

        # Публикация событий после успешного сохранения
        for event in order.collect_events():
            event_bus.publish(event)
```

Обратите внимание: сервис домена **не содержит бизнес-правил** — их проверяет сам агрегат. Сервис лишь координирует агрегаты и инфраструктуру (платёж, отправка событий).

### 6. Фабрика

Фабрика инкапсулирует создание сложного агрегата со стартовым набором инвариантов:

```python
class OrderFactory:
    @staticmethod
    def create(customer_id: UUID, items: list[tuple[UUID, str, Money, int]]) -> Order:
        order = Order(uuid4(), customer_id)
        for product_id, name, price, qty in items:
            order.add_item(product_id, name, price, qty)
        return order
```

## Правила тактического проектирования

1. **Граница агрегата = граница транзакции.** Один агрегат — одна транзакция. Нельзя атомарно менять два разных агрегата; для этого используется событие или сага.
2. **Агрегат должен быть небольшим.** Оптимальный размер — один корень + несколько объектов-значений. Чем больше агрегат, тем больше конфликтов записи.
3. **Корень агрегата — единственная точка входа.** Внутренние объекты не должны быть доступны извне для прямого изменения.
4. **Объекты-значения неизменяемы.** Вместо изменения создаём новый объект.
5. **Инварианты проверяются в момент изменения**, а не «по требованию» на слое приложения.
6. **Репозиторий хранит агрегаты целиком**, а не строки таблиц. Это делает домен независимым от схемы БД.
7. **Домен не зависит от инфраструктуры.** Никаких `requests`, `sqlalchemy`, Django-моделей в доменных объектах.
8. **События публикуются после сохранения агрегата**, чтобы не было гонок между записью и реакцией.

## Чего избегать

- **Анемичная модель** — когда у объекта только геттеры/сеттеры, а вся логика живёт в сервисах. Это DTO, а не доменная модель.
- **Гигантские агрегаты** — попытка сделать «один заказ на всё» приводит к блокировкам и хрупким инвариантам.
- **Умные сервисы, глупые объекты** — если сервис проверяет «не пустой ли заказ», значит, инвариант живёт не там.
- **Избыточная глубина** — для простых CRUD-поддоменов агрегаты и события не нужны; тактическое проектирование окупается на сложном Core Domain.

## Итог

Тактическое проектирование — это способ превратить стратегическую карту (bounded contexts, поддомены) в конкретный код. Сущности и объекты-значения моделируют поведение, агрегаты защищают инварианты, репозитории и фабрики отделяют домен от инфраструктуры, а события позволяют системе реагировать на изменения без жёстких связей.

Главное — не применять все блоки «по обязательной программе». Начните с богатой модели и агрегатов, а события и сервисы добавляйте только тогда, когда реально появляется потребность. Подробнее о том, как это делать итеративно, — в следующей статье про [итеративный процесс (Iterative Development)](https://blog.gorschal.com/iterative-development.html).
