---
layout: post
title: UUIDv7 в Django
excerpt_separator: <!--more-->
categories:
  - Разработка
tags:
  - python
  - uuid
---

_Перевод!!! [Первоисточник здесь.](https://abenezer.ca/blog/uuidv7-in-django)_

<!--more-->

Если вы хотите воспользоваться уникальными преимуществами `UUIDv7` в своих проектах на `Django` — вы попали по адресу. `UUIDv7` выделяется возможностью сортировки по времени создания, чего нет у `UUIDv4`. Это особенно полезно для приложений, которым требуется эффективная сортировка и индексация. В этом руководстве мы рассмотрим, что делает `UUIDv7` особенным, и как его реализовать в Django. Более подробное сравнение различных версий UUID можно найти в [разделе ресурсов](https://abenezer.ca/blog/#resources) в конце статьи.

### Что такое UUIDv7?

Как и другие `UUID`, `UUIDv7` — это 128-битный уникальный идентификатор, используемый для маркировки данных. Вы, вероятно, знакомы с `UUIDv4` — стандартным выбором для генерации случайных идентификаторов. Поскольку [UUIDv4](https://everyuuid.com/) полностью случайны, вероятность дублирования крайне мала — это отлично. Однако у `UUIDv4` есть недостаток: они не поддерживают естественную сортировку, что может замедлить работу, если использовать их в качестве индекса.

`UUIDv7`, напротив, представляет собой лексикографически сортируемый `UUID` со встроенной меткой времени. Это позволяет сортировать `UUID` по времени их создания — идеально подходит для ускорения поиска и повышения производительности индексации в базах данных.

### Структура UUIDv7

```
0190163d-8694-739b-aea5-966c26f8ad91
└─timestamp─┘ │└─┤ │└───rand_b─────┘
             ver │var
              rand_a
```

|                   | Bit Size | Description                                                             |
| ----------------- | -------- | ----------------------------------------------------------------------- |
| **Timestamp**     | 48       | Unix timestamp in milliseconds (sortable by generation time)            |
| **Version**       | 4        | Version (set to `7` for UUIDv7)                                         |
| **Random Part A** | 12       | Random component (`rand_a`), for uniqueness within the same millisecond |
| **Variant**       | 2        | Defines the UUID layout as RFC 4122 compliant                           |
| **Random Part B** | 62       | Additional random component (`rand_b`), providing further uniqueness    |

## Структура UUIDv7

```python
import os
import time
import uuid

def uuidv7() -> uuid.UUID:
    """
    Генерирует UUIDv7.
    """
    # Генерируем 16 случайных байт в качестве основы UUID
    value = bytearray(os.urandom(16))

    # Получаем текущую временную метку в миллисекундах с Unix-эпохи
    timestamp = int(time.time() * 1000)

    # Вставляем временную метку (48 бит) в UUID
    value[0] = (timestamp >> 40) & 0xFF
    value[1] = (timestamp >> 32) & 0xFF
    value[2] = (timestamp >> 24) & 0xFF
    value[3] = (timestamp >> 16) & 0xFF
    value[4] = (timestamp >> 8) & 0xFF
    value[5] = timestamp & 0xFF

    # Устанавливаем версию и вариант
    value[6] = (value[6] & 0x0F) | 0x70  # версия = 7
    value[8] = (value[8] & 0x3F) | 0x80  # вариант RFC 4122

    return uuid.UUID(bytes=bytes(value))
```

## Использование UUIDv7 в Django

Чтобы использовать `UUIDv7` в `Django`, можно создать собственное поле, расширяющее встроенный `UUIDField`, и генерировать UUIDv7 с помощью функции выше.

### Шаг 1: Создайте собственное поле UUID

app/fields.py

```python
import os
import time
import uuid
from django.core.exceptions import ValidationError
from django.db import models

def uuidv7() -> uuid.UUID:
    """
    Генерирует UUIDv7.
    """
    value = bytearray(os.urandom(16))
    timestamp = int(time.time() * 1000)

    value[0] = (timestamp >> 40) & 0xFF
    value[1] = (timestamp >> 32) & 0xFF
    value[2] = (timestamp >> 24) & 0xFF
    value[3] = (timestamp >> 16) & 0xFF
    value[4] = (timestamp >> 8) & 0xFF
    value[5] = timestamp & 0xFF

    value[6] = (value[6] & 0x0F) | 0x70
    value[8] = (value[8] & 0x3F) | 0x80

    return uuid.UUID(bytes=bytes(value))

class UUIDField(models.UUIDField):
    """
    Пользовательское поле UUID, поддерживающее разные версии UUID.
    """

    def __init__(self, primary_key: bool = True, version: int | None = None, editable: bool = False, *args, **kwargs):
        if version:
            if version == 2:
                raise ValidationError("UUID версии 2 не поддерживается.")
            if version < 1 or version > 7:
                raise ValidationError("Версия UUID должна быть от 1 до 7.")

            version_map = {
                1: uuid.uuid1,
                3: uuid.uuid3,
                4: uuid.uuid4,
                5: uuid.uuid5,
                7: uuidv7,
            }
            kwargs.setdefault("default", version_map[version])
        else:
            kwargs.setdefault("default", uuid.uuid4)

        kwargs.setdefault("editable", editable)
        kwargs.setdefault("primary_key", primary_key)

        super().__init__(*args, **kwargs)
```

Это пользовательское поле `UUIDField` принимает необязательный аргумент `version`. Если передать `version=7`, оно будет использовать функцию `uuidv7`; в противном случае по умолчанию используется `uuid.uuid4`. При необходимости можно настроить версию по умолчанию или расширить словарь для поддержки других версий.

### Шаг 2: Используйте пользовательское поле в моделях

app/models.py

```python
from django.db import models
from app.fields import UUIDField

class MyModel(models.Model):
    id = UUIDField(primary_key=True, version=7)
    name = models.CharField(max_length=100)
    created_at = models.DateTimeField(auto_now_add=True)
    updated_at = models.DateTimeField(auto_now=True)
```

Всё готово! Теперь у вас есть модель Django с UUIDv7 в качестве первичного ключа. В зависимости от ваших задач, вы можете использовать его как индекс для ускорения поиска или сортировки по времени создания.

### Тестирование UUIDv7 в оболочке Django

После выполнения миграций и создания нескольких экземпляров `MyModel` вы можете проверить сгенерированные `UUID` в оболочке `Django`:

```shell
>>> from app.models import MyModel
>>> ids = MyModel.objects.all().values_list('id', flat=True)
>>> for id in ids:
...     print(id)
...
UUID('01927d8c-1645-7ab9-86b6-d351a982a38f')
UUID('0192628e-ef00-7c4f-bdd6-811ff48f0e9b')
UUID('01927d8b-de00-7e2a-9cf7-b86dcb6d7c44')
UUID('01927db9-1b51-7844-94a7-349f372bdc00')
UUID('01927dee-f946-79dc-aac0-ec2ebbd740c4')
UUID('01927e08-c241-7b1b-bf0a-a63413abbdf2')
UUID('0192628a-de38-75f8-8b18-c43496533ad8')
```
