---
layout: post
title: Итеративный процесс (Iterative Development) в DDD
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

Domain-Driven Design — это не про «нарисовать модель один раз и писать код по ней». Это про **итеративный процесс**, в котором доменная модель и код развиваются вместе: мы постоянно уточняем модель через общение с экспертами и тут же отражаем изменения в коде и в универсальном языке.

<!--more-->

## Почему итерации важны

Модель предметной области редко удаётся построить с первого раза. На старте мы знаем о домене поверхностно, а правила всплывают по мере погружения. Если зафиксировать модель в начале и больше к ней не возвращаться, получится схема, которая не совпадает с реальностью.

Итеративный процесс позволяет:

- Подтверждать гипотезы о домене на маленьких шагах.
- Рано замечать, что термины и границы контекстов неверны.
- Обновлять **и модель, и код, и язык** одновременно — иначе они разъедутся.
- Снижать риск переписывания: каждый шаг даёт работающий результат.

## Основные практики

### 1. Event Storming

Совместный воркшоп с экспертами домена. На стикерах фиксируются события (`OrderPlaced`, `PaymentFailed`), команды (`ConfirmOrder`), действующие лица (`Клиент`, `Менеджер`), политики и агрегаты. За пару сессий получается черновая карта процессов — отличная точка старта для модели.

### 2. Модель и код — единое целое

В DDD модель — это не диаграмма, а **исполняемый код**. Классы, методы и имена переменных и есть модель. Поэтому любой шаг выглядит так:

1. Уточнили правило у эксперта → обновили термин в глоссарии.
2. Переименовали/изменили классы и методы в коде.
3. Написали тест на новое поведение.

Так код не отстаёт от модели, а глоссарий не превращается в мёртвую документацию.

### 3. Модельный шторм (Model Storming)

Короткие сессии (15–30 минут) перед конкретными задачами: обсуждаем один участок модели, фиксируем решения прямо в коде, не отвлекаясь на постороннее.

## Пример: система заказов на трёх итерациях

Разберём, как выглядит итеративное развитие на практике.

### Итерация 1: базовый заказ

Начинаем с простого: заказ содержит позиции и подтверждается.

```python
from dataclasses import dataclass

@dataclass(frozen=True)
class Money:
    amount: int
    currency: str = "RUB"


class Order:
    def __init__(self, order_id: str, customer_id: str):
        self.order_id = order_id
        self.customer_id = customer_id
        self.items: list[tuple[str, Money, int]] = []
        self.status = "created"

    def add_item(self, product_id: str, price: Money, qty: int) -> None:
        self.items.append((product_id, price, qty))

    def confirm(self) -> None:
        if not self.items:
            raise ValueError("Cannot confirm an empty order")
        self.status = "confirmed"
```

### Итерация 2: эксперт уточняет правило скидок

В ходе разговора выясняется: «Скидка применяется только если сумма заказа больше 10 000 и товар не в акции». Это доменное правило, а не деталь расчёта. Добавляем его в модель как инвариант.

```python
class Order:
    def __init__(self, order_id: str, customer_id: str):
        self.order_id = order_id
        self.customer_id = customer_id
        self.items: list[OrderItem] = []
        self.status = "created"
        self.discount: Money = Money(0)

    def apply_discount(self, percent: int) -> None:
        if self.status != "created":
            raise ValueError("Cannot change a confirmed order")
        if not (0 < percent <= 30):
            raise ValueError("Discount must be between 1 and 30 percent")
        total = self.total_price
        if total.amount < 10_000:
            raise ValueError("Discount requires order total above 10 000")
        self.discount = Money(total.amount * percent // 100)
```

Обратите внимание: после итерации изменились и модель, и язык. В глоссарии появляется «порог скидки», в коде — метод `apply_discount`, и теперь обе стороны говорят об одном и том же.

### Итерация 3: появляется новый поддомен

Бизнес добавляет программу лояльности. Это отдельная ответственность, поэтому мы не раздуваем `Order`, а выделяем новый ограниченный контекст.

```python
class LoyaltyProgram:
    """Новый bounded context: программа лояльности"""
    def __init__(self, customer_id: str):
        self.customer_id = customer_id
        self.points: int = 0

    def earn(self, amount: Money) -> None:
        self.points += amount.amount // 100

    def redeem(self, points: int) -> Money:
        if points > self.points:
            raise ValueError("Not enough points")
        self.points -= points
        return Money(points * 10)
```

Интеграцию с заказом делаем через событие, а не через прямое обращение:

```python
@dataclass
class OrderConfirmed:
    order_id: str
    customer_id: str
    total: Money


class OrderService:
    def __init__(self, order_repo, loyalty_service):
        self.order_repo = order_repo
        self.loyalty_service = loyalty_service

    def confirm_order(self, order_id: str) -> None:
        order = self.order_repo.find_by_id(order_id)
        order.confirm()
        self.order_repo.save(order)
        # Поддомены реагируют через события
        self.loyalty_service.on_order_confirmed(
            OrderConfirmed(order.order_id, order.customer_id, order.total_price)
        )
```

Так мы итеративно расширяем систему, не ломая существующую модель.

## Признаки того, что итерация сработала

- Модель стала **проще объяснить эксперту**, чем в прошлый раз.
- Новое требование укладывается в существующие классы без «костылей».
- Тесты на доменную логику обновляются вместе с моделью, а не живут отдельной жизнью.
- Глоссарий и код не разошлись: термины в обсуждениях совпадают с именами в коде.

## Частые ошибки

- **Big Design Up Front** — месяцы на «идеальную» модель без кода. Модель не проверена, а значит, почти наверняка неверна.
- **Код без модели** — итерации только про фичи, а модель не трогаем. Рано или поздно логика «утечёт» в сервисы и модель станет анемичной.
- **Глоссарий отдельно от кода** — термины обновляются в документации, но не в коде, и язык перестаёт быть универсальным.
- **Прыжки между контекстами в каждой задаче** — границы bounded contexts должны быть устойчивыми; итерации уточняют модель внутри контекста, а не перетаскивают логику между ними.

## Итог

Итеративный процесс — это связующее звено между стратегическим и тактическим проектированием. Стратегия даёт карту (контексты и поддомены), тактика — строительные блоки, а итерации позволяют всё это развивать по мере появления знаний о домене.

Начинайте с малого, подтверждайте модель на реальных задачах и всегда обновляйте вместе три вещи: модель, код и универсальный язык. Именно так DDD становится практикой, а не теорией.
