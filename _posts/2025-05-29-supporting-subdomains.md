---
layout: post
title: Использование концепции поддерживающие поддомены (Supporting Subdomains) в (DDD)
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

Поддерживающие поддомены (**Supporting Subdomains**) - это одна из трех категорий поддоменов в Domain-Driven Design (DDD), наряду с Core (основными) и Generic (общими) поддоменами. Давайте подробно разберем эту концепцию с примерами на Python.

<!--more-->

## Что такое поддерживающие поддомены?

Поддерживающие поддомены:

- Решают вспомогательные задачи для основного домена
- Не являются ключевыми для бизнеса, но необходимы для работы системы
- Часто могут быть заменены или переработаны без существенного влияния на бизнес
- Обычно разрабатываются внутри компании (в отличие от Generic Subdomains)

## Пример: Система управления заказами

Рассмотрим систему электронной коммерции с такими поддоменами:

- **Core Subdomain**: Обработка заказов
- **Supporting Subdomain**: Генерация отчетов о продажах
- **Generic Subdomain**: Платежная система (может быть куплена готовой)

### Реализация на Python

```python
import datetime

# Core Subdomain - Обработка заказов
class Order:
    def __init__(self, order_id, customer_id, items, total_amount):
        self.order_id = order_id
        self.customer_id = customer_id
        self.items = items  # список товаров
        self.total_amount = total_amount
        self.status = "created"

    def confirm(self):
        self.status = "confirmed"

    def cancel(self):
        self.status = "cancelled"


class OrderService:
    def __init__(self, payment_gateway):
        self.payment_gateway = payment_gateway

    def place_order(self, order):
        # Основная бизнес-логика
        if self.payment_gateway.process_payment(order.total_amount):
            order.confirm()
            return True
        return False


# Supporting Subdomain - Генерация отчетов
class SalesReportGenerator:
    def generate_daily_report(self, orders):
        """Генерация ежедневного отчета о продажах"""
        total_sales = sum(order.total_amount for order in orders)
        num_orders = len(orders)

        return {
            "date": datetime.date.today(),
            "total_sales": total_sales,
            "num_orders": num_orders,
            "average_order_value": total_sales / num_orders if num_orders else 0
        }

    def generate_customer_report(self, customer_id, orders):
        """Генерация отчета по конкретному клиенту"""
        customer_orders = [o for o in orders if o.customer_id == customer_id]
        total_spent = sum(order.total_amount for order in customer_orders)

        return {
            "customer_id": customer_id,
            "total_orders": len(customer_orders),
            "total_spent": total_spent
        }


# Generic Subdomain - Платежная система (имитация внешнего сервиса)
class PaymentGateway:
    def process_payment(self, amount):
        # В реальности здесь было бы обращение к внешнему API
        print(f"Processing payment of ${amount}")
        return True  # имитация успешного платежа


# Использование
if __name__ == "__main__":
    payment_gateway = PaymentGateway()
    order_service = OrderService(payment_gateway)

    # Создаем тестовые заказы
    orders = [
        Order(1, "cust1", ["item1", "item2"], 100),
        Order(2, "cust2", ["item3"], 50),
        Order(3, "cust1", ["item4"], 75)
    ]

    # Обработка заказов (Core Domain)
    for order in orders:
        order_service.place_order(order)

    # Генерация отчетов (Supporting Subdomain)
    report_generator = SalesReportGenerator()
    daily_report = report_generator.generate_daily_report(orders)
    customer_report = report_generator.generate_customer_report("cust1", orders)

    print("Daily Report:", daily_report)
    print("Customer Report:", customer_report)
```

## Другой пример: Система бронирования отелей

```python
# Core Subdomain - Бронирование номеров
class RoomBooking:
    def __init__(self, room_id, guest_id, check_in, check_out):
        self.room_id = room_id
        self.guest_id = guest_id
        self.check_in = check_in
        self.check_out = check_out
        self.status = "pending"

    def confirm_booking(self):
        self.status = "confirmed"


class BookingService:
    def __init__(self, notification_service):
        self.notification_service = notification_service

    def create_booking(self, booking):
        # Основная бизнес-логика бронирования
        booking.confirm_booking()
        self.notification_service.send_confirmation(booking)
        return True


# Supporting Subdomain - Уведомления
class NotificationService:
    def send_confirmation(self, booking):
        # В реальности здесь могла бы быть интеграция с email/SMS сервисом
        print(f"Sent confirmation to guest {booking.guest_id} for room {booking.room_id}")

    def send_reminder(self, booking):
        print(f"Sent reminder to guest {booking.guest_id} about upcoming stay")


# Использование
if __name__ == "__main__":
    notification_service = NotificationService()
    booking_service = BookingService(notification_service)

    from datetime import datetime, timedelta

    booking = RoomBooking(
        room_id="deluxe-123",
        guest_id="guest-456",
        check_in=datetime.now() + timedelta(days=7),
        check_out=datetime.now() + timedelta(days=14)

    booking_service.create_booking(booking)

    # Через несколько дней...
    notification_service.send_reminder(booking)
```

## Ключевые характеристики Supporting Subdomains в коде

- **Отдельная ответственность**: Каждый supporting поддомен решает свою узкую задачу
- **Зависимость от Core**: Supporting поддомены вызываются из Core поддомена, но не наоборот
- **Простота реализации**: Supporting поддомены обычно проще, чем Core
- **Возможность замены**: Легко заменить одну реализацию на другую

## Когда использовать Supporting Subdomains?

- Когда функциональность важна, но не является конкурентным преимуществом
- Когда можно выделить отдельную ограниченную ответственность
- Когда реализация может измениться без влияния на Core Domain

## Лучшие практики реализации

- Четко разделяйте код разных поддоменов
- Используйте разные модули/пакеты для разных поддоменов
- Документируйте, какие поддомены являются supporting
- Не усложняйте supporting поддомены - они должны оставаться простыми

Поддерживающие поддомены помогают сохранить фокус на основном домене, вынося вспомогательную функциональность в отдельные, более простые компоненты.
