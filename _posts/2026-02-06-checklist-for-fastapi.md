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

Документ описывает стандарты разработки API на FastAPI. Основной фокус — производительность (async), чистая архитектура (Layered Architecture) и безопасность типов.

title: Чек-лист для Fastapi

## 1. Архитектура: Слоистая структура (Layered Architecture)

FastAPI не навязывает структуру, поэтому важно самостоятельно разделять ответственности. Не размещайте бизнес-логику в обработчиках (endpoint functions).

| Слой                   | Ответственность                                                             |
| ---------------------- | --------------------------------------------------------------------------- |
| **Router**             | Маршрутизация, валидация HTTP параметров, вызов сервисов, возврат Response. |
| **Service**            | Бизнес-логика, транзакции, вызов внешних API, работа с БД через Repository. |
| **Repository/DAO**     | Запросы к БД (SQLAlchemy, Tortoise ORM), построение запросов.               |
| **Schemas (Pydantic)** | Контракт данных (DTO), валидация полей, сериализация.                       |

### Правильно

```python
# users/router.py
@router.post("/users", response_model=UserOut, status_code=201)
async def create_user(
    payload: UserCreate,
    service: UserService = Depends(get_user_service)
):
    return await service.create_user(payload)

# users/service.py
class UserService:
    def __init__(self, db: AsyncSession, cache: Redis):
        self.db = db
        self.cache = cache

    async def create_user(self, payload: UserCreate) -> UserOut:
        # Логика здесь, а не в роутере
        if await self.check_email_exists(payload.email):
            raise EmailAlreadyExistsError()

        user = User(email=payload.email, hashed_password=hash(payload.password))
        self.db.add(user)
        await self.db.commit()
        await self.db.refresh(user)

        # Явный маппинг на границе слоя сервиса (безопасность + избежание lazy loading)
        return UserOut.model_validate(user)
```

### Неправильно

```python
# users/router.py
@router.post("/users")
async def create_user(payload: UserCreate, db: AsyncSession = Depends(get_db)):
    # Вся логика в роутере — плохо
    if await db.execute(select(User).where(User.email == payload.email)):
        raise HTTPException(400, "Exists")

    user = User(**payload.dict())
    db.add(user)
    await db.commit()
    return user
```

---

## 2. Dependency Injection: Система зависимостей

DI в FastAPI — мощный инструмент для управления ресурсами (БД, кэш, сервисы).

### Управление ресурсами (Context Managers)

Используйте `yield` для зависимостей, требующих закрытия (сессии БД, файлы).

### Правильно

```python
# deps/db.py
async def get_db():
    async with async_session_maker() as session:
        try:
            yield session
        finally:
            # Закрытие происходит автоматически
            pass

# deps/services.py
def get_user_service(db: AsyncSession = Depends(get_db)) -> UserService:
    return UserService(db)
```

### Важно: Жизненный цикл

- `Depends()` по умолчанию **кеширует** результат в рамках одного HTTP запроса. Это оптимально и позволяет безопасно вызывать одну и ту же зависимость несколько раз.
- Используйте `yield` **только** для ресурсов, требующих закрытия (БД, HTTP-клиенты, файловые дескрипторы).
- **Анти-паттерн:** Создание сервиса через `yield` без `finally` — это избыточно и может удерживать память дольше необходимого.

---

## 3. Асинхронность: Запрещенные блокирующие вызовы

FastAPI работает в одном потоке event-loop. Блокирующий вызов останавливает всё приложение.

### Выбор сигнатуры: `def` vs `async def`

- Используйте `async def`, если внутри функции есть хотя бы один `await`.
- Используйте `def` (синхронную), если внутри нет асинхронных вызовов. FastAPI автоматически запустит её в threadpool, не нагружая event-loop лишними обертками.

### Таблица блокирующих операций

| Операция         | Блокирующее           | Неблокирующее                                |
| ---------------- | --------------------- | -------------------------------------------- |
| HTTP запрос      | `requests.get()`      | `httpx.get()` / `aiohttp`                    |
| БД (Postgres)    | `psycopg2`, `sqlite3` | `asyncpg`, `SQLAlchemy (async)`, `databases` |
| Время            | `time.sleep()`        | `asyncio.sleep()`                            |
| Файловая система | `open()`, `os.path`   | `aiofiles`, `anyio`                          |

### Как быть, если нужна синхронная библиотека?

Если библиотеку нельзя заменить на асинхронную, используйте `run_in_threadpool`.

```python
from fastapi.concurrency import run_in_threadpool

@router.get("/sync-op")
async def do_sync():
    # Выполняется в отдельном пуле потоков, не блочит event-loop
    result = await run_in_threadpool(heavy_sync_function, arg1, arg2)
    return result
```

---

## 4. Pydantic: Валидация и Сериализация

Pydantic v2 требует явного разделения моделей и поддерживает современные подходы к типизации.

### Группы моделей (Schema Design)

Не используйте одну модель `User` из ORM везде. Создавайте отдельные схемы:

1.  `UserCreate` — входные данные (пароль, email).
2.  `UserOut` — выходные данные (id, email, но БЕЗ пароля).
3.  `UserUpdate` — частичное обновление (все поля Optional).

### Использование `response_model`

Всегда указывайте `response_model`. Это фильтрует выходные данные (например, скрывает `password_hash`) и генерирует OpenAPI схему.

### Правильно (с использованием `Annotated` для переиспользования)

```python
from typing import Annotated
from pydantic import BaseModel, ConfigDict, EmailStr, Field

class UserCreate(BaseModel):
    email: Annotated[EmailStr, Field(description="User email address")]
    password: Annotated[str, Field(min_length=8, description="User password")]

class UserOut(BaseModel):
    id: int
    email: EmailStr
    is_active: bool = True

    model_config = ConfigDict(from_attributes=True) # Для маппинга из ORM

# router.py
@router.post("/users", response_model=UserOut)
async def create_user(data: UserCreate):
    ...
```

---

## 5. Обработка ошибок

Не используйте `HTTPException` внутри слоя сервисов. Сервисы должны выбрасывать доменные исключения, которые перехватываются глобальным обработчиком.

### Правильно (Глобальный обработчик)

```python
# exceptions.py
class DomainError(Exception):
    pass

class UserNotFound(DomainError):
    pass

# main.py
@app.exception_handler(UserNotFound)
async def user_not_found_handler(request: Request, exc: UserNotFound):
    return JSONResponse(
        status_code=404,
        content={"detail": "User not found", "code": "user_not_found"},
    )

# service.py
async def get_user(pk: int):
    user = await repo.get(pk)
    if not user:
        raise UserNotFound(pk)  # Чистое исключение
    return user
```

---

## 6. Настройки (Configuration)

Используйте `pydantic-settings` для управления конфигурацией. Не используйте `.env` файлы напрямую через `os.getenv` в разных местах.

### Правильно

```python
from pydantic_settings import BaseSettings, SettingsConfigDict

class Settings(BaseSettings):
    app_name: str = "Awesome API"
    admin_email: str

    database_url: str
    redis_url: str

    model_config = SettingsConfigDict(env_file=".env", env_file_encoding="utf-8")

# Зависимость для настроек
def get_settings() -> Settings:
    return Settings()

@router.get("/info")
async def info(settings: Settings = Depends(get_settings)):
    return {"app_name": settings.app_name}
```

---

## Чек-лист для Code Review (FastAPI)

Перед мержем проверьте:

- [ ] **Разделение ответственности:** В роутерах нет SQL-запросов и бизнес-логики (только вызов сервисов).
- [ ] **Асинхронность:** Отсутствуют блокирующие вызовы (`requests`, `time.sleep`, `psycopg2`) внутри `async def`.
- [ ] **DI:** Зависимости (БД, конфиг, сервисы) внедряются через `Depends`, а не создаются внутри функции.
- [ ] **Pydantic v2:** Используется `model_config = ConfigDict(...)` вместо устаревшего `class Config`, применяется `Annotated` для переиспользуемых полей.
- [ ] **Безопасность:** `response_model` исключает утечку чувствительных полей (пароли, токены), а сервис возвращает Pydantic-схему, а не ORM-объект.
- [ ] **Обработка ошибок:** Используются кастомные исключения и `exception_handler`, а не множество `try/except` в роутерах.
- [ ] **Типизация:** Функции имеют возвращаемые типы (return type hints).
- [ ] **Тестирование:** Используется `TestClient` (для синхронных тестов) или `httpx.AsyncClient` (для асинхронных), БД мокируется или использует тестовую in-memory базу.
