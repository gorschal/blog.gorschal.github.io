---
layout: post
title: Шпаргалка по SQLAlchemy Core
excerpt_separator: <!--more-->
categories:
  - Разработка
tags:
  - architecture
  - sql
  - sqlalchemy_core
  - sqlalchemy
  - python
---

В прошлой статье про SQL так и не смог раскрыть то, что планировал. Исправляю в этой. Текст полностью посвящен SQLAlchemy Core, и постарался вообще без воды — только суть.

<!--more-->

SQLAlchemy Core - это система SQL-выражений, которая предоставляет схему-centric использование SQL. Это не ORM!

## 1. Подключение к базе данных

```python
from sqlalchemy import create_engine

# Создание движка для подключения к SQLite
engine = create_engine('sqlite:///example.db', echo=True)

# Для PostgreSQL
# engine = create_engine('postgresql://user:password@localhost/mydatabase')

# Для MySQL
# engine = create_engine('mysql+pymysql://user:password@localhost/mydatabase')
```

## 2. Определение метаданных и таблиц

```python
from sqlalchemy import MetaData, Table, Column, Integer, String, ForeignKey

metadata = MetaData()

# Определение таблицы users
users = Table('users', metadata,
    Column('id', Integer, primary_key=True),
    Column('name', String(50)),
    Column('fullname', String(50)),
)

# Определение таблицы addresses
addresses = Table('addresses', metadata,
    Column('id', Integer, primary_key=True),
    Column('user_id', None, ForeignKey('users.id')),
    Column('email_address', String(100), nullable=False)
)

# Создание всех таблиц
metadata.create_all(engine)
```

## 3. Вставка данных

```python
from sqlalchemy import insert

# Одиночная вставка
stmt = insert(users).values(name='john', fullname='John Doe')
with engine.connect() as conn:
    result = conn.execute(stmt)
    print(result.inserted_primary_key)  # ID вставленной записи

# Множественная вставка
with engine.connect() as conn:
    conn.execute(insert(users), [
        {'name': 'wendy', 'fullname': 'Wendy Williams'},
        {'name': 'mary', 'fullname': 'Mary Contrary'},
        {'name': 'fred', 'fullname': 'Fred Flintstone'}
    ])
```

## 4. Запросы (SELECT)

```python
from sqlalchemy import select

# Простой SELECT
stmt = select(users)
with engine.connect() as conn:
    result = conn.execute(stmt)
    for row in result:
        print(row)  # Доступ по индексу
        print(row.name)  # Доступ по имени столбца

# SELECT с условием
stmt = select(users).where(users.c.name == 'john')
with engine.connect() as conn:
    result = conn.execute(stmt)
    print(result.fetchone())

# JOIN запрос
stmt = select(users, addresses).where(users.c.id == addresses.c.user_id)
with engine.connect() as conn:
    for row in conn.execute(stmt):
        print(row)
```

## 5. Обновление данных (UPDATE)

```python
from sqlalchemy import update

stmt = update(users).where(users.c.name == 'john').values(fullname='John Smith')
with engine.connect() as conn:
    conn.execute(stmt)
    print(f"Updated {stmt.rowcount} rows")
```

## 6. Удаление данных (DELETE)

```python
from sqlalchemy import delete

stmt = delete(users).where(users.c.name == 'fred')
with engine.connect() as conn:
    conn.execute(stmt)
    print(f"Deleted {stmt.rowcount} rows")
```

## 7. Транзакции

```python
with engine.connect() as conn:
    trans = conn.begin()  # Начало транзакции
    try:
        conn.execute(insert(addresses), [
            {'user_id': 1, 'email_address': 'john@example.com'},
            {'user_id': 1, 'email_address': 'john@gmail.com'},
            {'user_id': 2, 'email_address': 'wendy@example.com'},
        ])
        trans.commit()  # Фиксация транзакции
    except:
        trans.rollback()  # Откат при ошибке
        raise
```

## 8. Агрегатные функции

```python
from sqlalchemy import func

# Подсчет количества пользователей
stmt = select(func.count()).select_from(users)
with engine.connect() as conn:
    result = conn.execute(stmt)
    print(f"Total users: {result.scalar()}")

# Группировка
stmt = select(
    addresses.c.user_id,
    func.count(addresses.c.id).label('num_addresses')
).group_by(addresses.c.user_id)
with engine.connect() as conn:
    for row in conn.execute(stmt):
        print(f"User {row.user_id} has {row.num_addresses} addresses")
```

## 9. Подзапросы

```python
# Создание подзапроса
subq = select(addresses.c.user_id, func.count('*').label('address_count')) \
    .group_by(addresses.c.user_id) \
    .subquery()

# Использование подзапроса в основном запросе
stmt = select(users.c.name, subq.c.address_count) \
    .join(subq, users.c.id == subq.c.user_id)
with engine.connect() as conn:
    for row in conn.execute(stmt):
        print(f"{row.name} has {row.address_count} addresses")
```

## 10. Сложные условия WHERE

```python
from sqlalchemy import and_, or_

# AND условие
stmt = select(users).where(
    and_(
        users.c.name.like('j%'),
        users.c.fullname != 'John Doe'
    )
)

# OR условие
stmt = select(users).where(
    or_(
        users.c.name == 'john',
        users.c.name == 'wendy'
    )
)

# Комбинированные условия
stmt = select(users).where(
    (users.c.name == 'john') | (users.c.name == 'wendy'),
    users.c.fullname != None
)
```

## 11. Работа с JSON (для PostgreSQL)

```python
from sqlalchemy import JSON

# Определение таблицы с JSON полем
data_table = Table('data', metadata,
    Column('id', Integer, primary_key=True),
    Column('data', JSON)
)

metadata.create_all(engine)

# Вставка JSON данных
with engine.connect() as conn:
    conn.execute(insert(data_table), [
        {'data': {'key1': 'value1', 'key2': 'value2'}},
        {'data': {'key1': 'value3', 'key3': 'value4'}}
    ])

# Запрос по JSON полю
stmt = select(data_table).where(
    data_table.c.data['key1'].astext == 'value1'
)
with engine.connect() as conn:
    for row in conn.execute(stmt):
        print(row.data)
```
