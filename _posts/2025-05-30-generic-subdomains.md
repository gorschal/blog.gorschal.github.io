---
layout: post
title: Использование концепции общих поддоменов (Generic Subdomains) в (DDD)
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

Общие поддомены (**Generic Subdomains**) - это концепция из Domain-Driven Design (DDD), которая относится к частям домена, которые являются общими для многих систем и не являются уникальными или конкурентными преимуществами бизнеса.

<!--more-->

## Что такое Generic Subdomains?

Generic Subdomains - это:

- Части системы, которые решают стандартные, хорошо известные проблемы
- Не являются ключевыми для бизнес-логики
- Могут быть реализованы с использованием готовых решений или библиотек
- Часто требуют интеграции, а не разработки с нуля

Примеры: аутентификация, логирование, отправка email, генерация отчетов в стандартных форматах.

## Отличие от Core и Supporting Subdomains

- **Core Subdomain** - уникальная бизнес-логика, конкурентное преимущество (например, алгоритм рекомендаций для Netflix)
- **Supporting Subdomain** - специфичная для бизнеса логика, но не ключевая (например, система управления контентом для медиа-компании)
- **Generic Subdomain** - общие функции, встречающиеся во многих системах

## Примеры Generic Subdomains в Python

### 1. Аутентификация и авторизация

```python
from passlib.context import CryptContext
from datetime import datetime, timedelta, timezone
import jwt

# Конфигурация хеширования паролей
pwd_context = CryptContext(schemes=["bcrypt"], deprecated="auto")

# Generic Subdomain: Аутентификация
class AuthService:
    @staticmethod
    def verify_password(plain_password: str, hashed_password: str) -> bool:
        return pwd_context.verify(plain_password, hashed_password)

    @staticmethod
    def get_password_hash(password: str) -> str:
        return pwd_context.hash(password)

    @staticmethod
    def create_access_token(data: dict, secret_key: str, expires_delta: timedelta) -> str:
        to_encode = data.copy()
        expire = datetime.now(timezone.utc) + expires_delta
        to_encode.update({"exp": expire})
        encoded_jwt = jwt.encode(to_encode, secret_key, algorithm="HS256")
        return encoded_jwt
```

> **На заметку**: библиотека `passlib` больше не поддерживается авторами. Для новых проектов лучше использовать `pwdlib` или `argon2-cffi`, а готовые решения вроде `Auth0`/`Keycloak` — для полноценного Identity Provider.

### 2. Логирование (Logging)

```python
import logging
from logging.handlers import RotatingFileHandler
import sys

# Generic Subdomain: Логирование
class LoggingService:
    @staticmethod
    def setup_logging(log_file: str = "app.log"):
        logger = logging.getLogger()
        logger.setLevel(logging.INFO)

        # Формат логов
        formatter = logging.Formatter('%(asctime)s - %(name)s - %(levelname)s - %(message)s')

        # Логи в файл с ротацией
        file_handler = RotatingFileHandler(
            log_file, maxBytes=1024*1024, backupCount=5
        )
        file_handler.setFormatter(formatter)
        logger.addHandler(file_handler)

        # Логи в консоль
        console_handler = logging.StreamHandler(sys.stdout)
        console_handler.setFormatter(formatter)
        logger.addHandler(console_handler)

        return logger
```

### 3. Отправка email

```python
import smtplib
from email.mime.text import MIMEText
from email.mime.multipart import MIMEMultipart

# Generic Subdomain: Отправка email
class EmailService:
    def __init__(self, smtp_server: str, smtp_port: int, username: str, password: str):
        self.smtp_server = smtp_server
        self.smtp_port = smtp_port
        self.username = username
        self.password = password

    def send_email(self, to_email: str, subject: str, body: str, is_html: bool = False):
        msg = MIMEMultipart()
        msg['From'] = self.username
        msg['To'] = to_email
        msg['Subject'] = subject

        if is_html:
            msg.attach(MIMEText(body, 'html'))
        else:
            msg.attach(MIMEText(body, 'plain'))

        with smtplib.SMTP(self.smtp_server, self.smtp_port) as server:
            server.starttls()
            server.login(self.username, self.password)
            server.send_message(msg)
```

### 4. Генерация PDF отчетов

```python
from fpdf import FPDF

# Generic Subdomain: Генерация PDF
class PdfReportService:
    @staticmethod
    def generate_simple_report(data: list[dict], output_file: str):
        pdf = FPDF()
        pdf.add_page()
        pdf.set_font("Arial", size=12)

        # Заголовок
        pdf.cell(200, 10, txt="Отчет", ln=1, align='C')
        pdf.ln(10)

        # Данные
        for item in data:
            for key, value in item.items():
                pdf.cell(50, 10, txt=f"{key}:", border=1)
                pdf.cell(140, 10, txt=str(value), border=1, ln=1)
            pdf.ln(5)

        pdf.output(output_file)
```

> **На заметку**: ставьте `fpdf2` (активно поддерживаемый форк), а не старую библиотеку `fpdf` — она давно не обновляется. Установка: `pip install fpdf2`, импорт при этом остаётся `from fpdf import FPDF`.

## Интеграция Generic Subdomains в DDD-систему

Рассмотрим пример интеграции Generic Subdomain (EmailService) в Core Domain (OrderProcessing):

```python
from datetime import datetime
from typing import List

# Core Subdomain: Обработка заказов
class Order:
    def __init__(self, order_id: str, customer_email: str, items: List[str], total: float):
        self.order_id = order_id
        self.customer_email = customer_email
        self.items = items
        self.total = total
        self.created_at = datetime.now()
        self.status = "created"

class OrderProcessor:
    def __init__(self, email_service: EmailService):
        self.email_service = email_service

    def process_order(self, order: Order):
        # Бизнес-логика обработки заказа (Core Domain)
        order.status = "processed"

        # Использование Generic Subdomain для отправки уведомления
        subject = f"Ваш заказ #{order.order_id} принят в обработку"
        body = f"""
        Детали заказа:
        Номер: {order.order_id}
        Сумма: {order.total}
        Статус: {order.status}
        """

        self.email_service.send_email(
            to_email=order.customer_email,
            subject=subject,
            body=body
        )

        return order

# Использование
email_service = EmailService(
    smtp_server="smtp.example.com",
    smtp_port=587,
    username="user@example.com",
    password="password"
)

order_processor = OrderProcessor(email_service)

order = Order(
    order_id="12345",
    customer_email="customer@example.com",
    items=["Товар 1", "Товар 2"],
    total=100.50
)

processed_order = order_processor.process_order(order)
```

## Стратегии реализации Generic Subdomains

- **Использование готовых библиотек** (как в примерах выше)
- **Интеграция внешних сервисов** (например, Auth0 для аутентификации)
- **Собственная реализация** (если требуется особая кастомизация)

## Преимущества выделения Generic Subdomains

- **Сосредоточение на Core Domain** - команда может фокусироваться на уникальной бизнес-логике
- **Снижение сложности** - стандартные компоненты не нужно разрабатывать с нуля
- **Упрощение поддержки** - обновление Generic Subdomains независимо от основной системы
- **Повторное использование** - одни и те же компоненты могут использоваться в разных проектах

## Заключение

Generic Subdomains в DDD позволяют эффективно организовать архитектуру системы, выделив стандартные, повторно используемые компоненты. В Python такие поддомены часто реализуются с помощью существующих библиотек (как `passlib` для аутентификации или `fpdf` для генерации PDF), что ускоряет разработку и повышает надежность системы.

Ключевой принцип - не изобретать велосипед для задач, которые уже имеют стандартные решения, и фокусироваться на разработке уникальной бизнес-логики, которая действительно обеспечивает конкурентное преимущество.
