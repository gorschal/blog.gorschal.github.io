---
layout: post
title: Модель предметной области (Domain Model) в DDD
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

**Модель предметной области (Domain Model)** — это ключевая концепция в **Domain-Driven Design (DDD)**, которая представляет собой абстракцию бизнес-логики и правил предметной области в виде программной модели.

<!--more-->

Domain Model в DDD — это не просто структура данных, а живая модель бизнеса, которая помогает создавать гибкие и понятные системы. Её правильное проектирование требует глубокого понимания предметной области и тесного взаимодействия с бизнес-экспертами.

### **1. Что такое Domain Model?**  
Это **структурированное представление бизнес-логики**, включающее:  
- **Сущности (Entities)** – объекты с идентификатором (например, `User`, `Order`).  
- **Значения (Value Objects)** – неизменяемые объекты без идентификатора (например, `Address`, `Money`).  
- **Агрегаты (Aggregates)** – группы связанных объектов, управляемые через корневой объект (например, `Order` с `OrderItems`).  
- **Сервисы предметной области (Domain Services)** – логика, не принадлежащая конкретному объекту (например, `TransferService`).  
- **Репозитории (Repositories)** – интерфейсы для работы с хранилищами данных.  
- **Фабрики (Factories)** – способы создания сложных объектов.  

### **2. Зачем нужна Domain Model?**  
- **Избегание анемичной модели** (когда данные и логика разделены, как в DTO).  
- **Четкое отражение бизнес-правил** в коде.  
- **Упрощение коммуникации** между разработчиками и экспертами предметной области.  
- **Гибкость и поддерживаемость** за счет инкапсуляции логики внутри объектов.  

### **3. Как проектировать Domain Model?**  
1. **Анализ предметной области** (общение с экспертами).  
2. **Выделение ограниченных контекстов (Bounded Contexts)**.  
3. **Определение ключевых агрегатов и сущностей**.  
4. **Инкапсуляция бизнес-правил внутри объектов**.  
5. **Использование Ubiquitous Language** (единый язык для кода и документации).  


## **Пример Domain Model на Python (DDD)**  

### **1. Сущности (Entities)**  
Объекты с уникальным идентификатором.  

```python
from dataclasses import dataclass
from uuid import UUID, uuid4

@dataclass
class Product:
    id: UUID
    name: str
    price: float

@dataclass
class Customer:
    id: UUID
    name: str

@dataclass
class OrderItem:
    product: Product
    quantity: int

    @property
    def total_price(self) -> float:
        return self.product.price * self.quantity

class Order:
    def __init__(self, id: UUID, customer: Customer):
        self.id = id
        self.customer = customer
        self.items: list[OrderItem] = []
        self._is_confirmed = False

    def add_item(self, product: Product, quantity: int) -> None:
        if self._is_confirmed:
            raise ValueError("Cannot modify a confirmed order")
        self.items.append(OrderItem(product, quantity))

    def confirm(self) -> None:
        if not self.items:
            raise ValueError("Cannot confirm an empty order")
        self._is_confirmed = True

    @property
    def total_price(self) -> float:
        return sum(item.total_price for item in self.items)
```

---

### **2. Value Objects**  
Неизменяемые объекты без идентификатора.  

```python
@dataclass(frozen=True)  # frozen=True делает объект неизменяемым
class Address:
    street: str
    city: str
    postal_code: str

# Пример использования
address = Address("123 Main St", "Springfield", "12345")
```

---

### **3. Domain Service**  
Логика, которая не принадлежит конкретной сущности.  

```python
class PaymentService:
    def process_payment(self, order: Order, amount: float) -> bool:
        print(f"Processing payment of ${amount} for order {order.id}")
        return True  # Упрощенная реализация

class OrderService:
    def __init__(self, payment_service: PaymentService):
        self.payment_service = payment_service

    def place_order(self, order: Order) -> bool:
        if order.total_price <= 0:
            raise ValueError("Order total must be positive")
        
        if self.payment_service.process_payment(order, order.total_price):
            order.confirm()
            return True
        return False
```

---

### **4. Repository (Интерфейс для работы с данными)**  

```python
from abc import ABC, abstractmethod

class OrderRepository(ABC):
    @abstractmethod
    def save(self, order: Order) -> None:
        pass

    @abstractmethod
    def find_by_id(self, order_id: UUID) -> Order | None:
        pass

# Пример реализации (в реальности тут будет работа с БД)
class InMemoryOrderRepository(OrderRepository):
    def __init__(self):
        self.orders: dict[UUID, Order] = {}

    def save(self, order: Order) -> None:
        self.orders[order.id] = order

    def find_by_id(self, order_id: UUID) -> Order | None:
        return self.orders.get(order_id)
```

---

### **5. Пример использования**  

```python
# Создаем продукты и клиента
product1 = Product(uuid4(), "Laptop", 999.99)
product2 = Product(uuid4(), "Mouse", 25.50)
customer = Customer(uuid4(), "John Doe")

# Создаем заказ
order = Order(uuid4(), customer)
order.add_item(product1, 1)
order.add_item(product2, 2)

# Сервисы и репозиторий
payment_service = PaymentService()
order_service = OrderService(payment_service)
order_repo = InMemoryOrderRepository()

# Оформляем заказ
if order_service.place_order(order):
    order_repo.save(order)
    print(f"Order {order.id} confirmed! Total: ${order.total_price}")
else:
    print("Payment failed")
```
