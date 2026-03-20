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

 <!--more-->

## 1. Архитектура: Слоистая структура (Layered Architecture)

FastAPI не навязывает структуру, поэтому важно самостоятельно разделять ответственности. Не размещайте бизнес-логику в обработчиках (endpoint functions).

| Слой                   | Ответственность                                                             |
| ---------------------- | --------------------------------------------------------------------------- |
| **Router**             | Маршрутизация, валидация HTTP параметров, вызов сервисов, возврат Response. |
| **Service**            | Бизнес-логика, транзакции, вызов внешних API, работа с БД через Repository. |
| **Repository/DAO**     | Запросы к БД (SQLAlchemy, Tortoise ORM), построение запросов.               |
| **Schemas (Pydantic)** | Контракт данных (DTO), валидация полей, сериализация.                       |

### Инъекция зависимостей через DI систему

FastAPI имеет встроенную мощную систему DI. Используйте её для управления всеми зависимостями: БД, сервисы, конфиги. Это делает код тестируемым и избавляет от глобальных состояний.

### Управление транзакциями в асинхронной среде

В асинхронных приложениях управление транзакциями критически важно. Используйте контекстные менеджеры или явные коммиты/роллбэки на уровне сервисов. Для сложных операций с несколькими репозиториями используйте шаблон Unit of Work.

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
from sqlalchemy.ext.asyncio import AsyncSession
from sqlalchemy.exc import IntegrityError

class UserService:
    def __init__(self, db: AsyncSession, cache: Redis, uow: UnitOfWork):
        self.db = db
        self.cache = cache
        self.uow = uow  # Unit of Work для управления транзакциями

    async def create_user(self, payload: UserCreate) -> User:
        # Начинаем транзакцию через UOW
        async with self.uow:
            # Проверка существования через репозиторий
            if await self.uow.users.exists(email=payload.email):
                raise EmailAlreadyExistsError()

            # Создание пользователя
            user = User(email=payload.email, hashed_password=self._hash_password(payload.password))
            self.uow.users.add(user)

            # Создание профиля в рамках той же транзакции
            profile = Profile(user_id=user.id, name=payload.name)
            self.uow.profiles.add(profile)

            # Логирование в БД
            await self.uow.audit.log("user_created", user_id=user.id)

            # UOW автоматически закоммитит всё при успешном выходе из контекста
            # или откатит при исключении

        # Побочные эффекты ПОСЛЕ успешной транзакции
        await self.cache.set(f"user:{user.id}", "active", expire=3600)
        await self._send_welcome_email.delay(user.email)  # Фоновая задача

        return user

# core/uow.py
class UnitOfWork:
    def __init__(self, session_factory: Callable[[], AsyncSession]):
        self.session_factory = session_factory
        self.session: Optional[AsyncSession] = None
        self.users: Optional[UserRepository] = None
        self.profiles: Optional[ProfileRepository] = None

    async def __aenter__(self):
        self.session = self.session_factory()
        self.users = UserRepository(self.session)
        self.profiles = ProfileRepository(self.session)
        return self

    async def __aexit__(self, exc_type, exc_val, exc_tb):
        if exc_type:
            await self.session.rollback()
        else:
            await self.session.commit()
        await self.session.close()
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

    # Если здесь произойдет ошибка, пользователь уже в БД!
    await send_welcome_email(payload.email)

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

# deps/uow.py
async def get_uow(
    session: AsyncSession = Depends(get_db)
) -> UnitOfWork:
    uow = UnitOfWork(lambda: session)
    async with uow:
        yield uow

# deps/services.py
def get_user_service(
    uow: UnitOfWork = Depends(get_uow),
    cache: Redis = Depends(get_redis),
) -> UserService:
    return UserService(uow, cache)

# router — чистое использование
@router.post("/users")
async def create_user(
    payload: UserCreate,
    service: UserService = Depends(get_user_service)
):
    return await service.create_user(payload)
```

### Анти-паттерн

- Создавать сессию БД глобально.
- Забывать про `yield` и закрывать соединения вручную в `except` блоках.
- Использовать глобальные переменные для хранения состояния запроса.
- Создавать зависимости внутри функции (например, `UserService()`), а не через `Depends`.

---

## 3. Асинхронность: Запрещенные блокирующие вызовы

FastAPI работает в одном потоке event-loop. Блокирующий вызов останавливает всё приложение.

**Правило:** В `async def` эндпоинтах никогда не вызывайте синхронные блокирующие функции.

| Операция         | Блокирующее           | Неблокирующее                                |
| ---------------- | --------------------- | -------------------------------------------- |
| HTTP запрос      | `requests.get()`      | `httpx.AsyncClient` / `aiohttp`              |
| БД (Postgres)    | `psycopg2`, `sqlite3` | `asyncpg`, `SQLAlchemy (async)`, `databases` |
| Время            | `time.sleep()`        | `asyncio.sleep()`                            |
| Файловая система | `open()`, `os.path`   | `aiofiles`, `anyio`                          |
| Хеширование      | `bcrypt.hashpw()`     | `passlib[bcrypt]` с `async` обёрткой         |

### Как быть, если нужна синхронная библиотека?

Если библиотеку нельзя заменить на асинхронную, используйте `run_in_threadpool`.

```python
from fastapi.concurrency import run_in_threadpool
import bcrypt

@router.get("/sync-op")
async def do_sync():
    # Выполняется в отдельном пуле потоков, не блочит event-loop
    hashed = await run_in_threadpool(bcrypt.hashpw, password, salt)
    return {"hash": hashed}
```

---

## 4. Pydantic: Валидация и Сериализация

Pydantic v2 (используется в современных версиях) требует явного разделения моделей.

### Группы моделей (Schema Design)

Не используйте одну модель `User` из ORM везде. Создавайте отдельные схемы:

1.  `UserCreate` — входные данные (пароль, email).
2.  `UserOut` — выходные данные (id, email, но БЕЗ пароля).
3.  `UserUpdate` — частичное обновление (все поля Optional).

### Использование `response_model`

Всегда указывайте `response_model`. Это фильтрует выходные данные (например, скрывает `password_hash`) и генерирует OpenAPI схему.

### Правильно

```python
# schemas.py
from pydantic import BaseModel, ConfigDict, EmailStr

class UserCreate(BaseModel):
    email: EmailStr
    password: str
    name: str

class UserOut(BaseModel):
    id: int
    email: EmailStr
    name: str
    is_active: bool = True

    model_config = ConfigDict(from_attributes=True)  # Для маппинга из ORM

# router.py
@router.post("/users", response_model=UserOut)
async def create_user(data: UserCreate):
    ...
```

---

## 5. Обработка ошибок

Не используйте `HTTPException` внутри слоя сервисов. Сервисы должны выбрасывать доменные исключения, которые перехватываются глобальным обработчиком.

### Глобальная обработка исключений

Централизованный обработчик трансформирует доменные ошибки в понятные HTTP-ответы с единым форматом.

### Правильно

```python
# core/exceptions.py
class DomainError(Exception):
    """Базовое исключение для всех бизнес-ошибок"""
    def __init__(self, message: str, code: str = None):
        self.message = message
        self.code = code or self.__class__.__name__
        super().__init__(message)

class UserNotFound(DomainError):
    pass

class EmailAlreadyExistsError(DomainError):
    pass

class InsufficientPermissionsError(DomainError):
    pass

# main.py
from fastapi import FastAPI, Request
from fastapi.responses import JSONResponse

app = FastAPI()

@app.exception_handler(DomainError)
async def domain_error_handler(request: Request, exc: DomainError):
    status_map = {
        UserNotFound: 404,
        EmailAlreadyExistsError: 409,
        InsufficientPermissionsError: 403,
    }
    status_code = status_map.get(type(exc), 400)

    return JSONResponse(
        status_code=status_code,
        content={
            "error": exc.message,
            "code": exc.code,
            "type": exc.__class__.__name__
        }
    )

@app.exception_handler(IntegrityError)
async def integrity_error_handler(request: Request, exc: IntegrityError):
    # Обработка специфичных ошибок БД
    return JSONResponse(
        status_code=409,
        content={"error": "Database constraint violation", "code": "db_constraint"}
    )

# service.py
async def get_user(user_id: int):
    user = await repo.get(user_id)
    if not user:
        raise UserNotFound(f"User with id {user_id} not found")  # Чистое исключение
    return user
```

### Неправильно

```python
# service.py — плохо: HTTPException в сервисе
async def get_user(user_id: int):
    user = await repo.get(user_id)
    if not user:
        raise HTTPException(404, "User not found")  # Сервис знает про HTTP!

# router.py — плохо: try/except в каждом роутере
@router.get("/users/{user_id}")
async def get_user(user_id: int, service: UserService = Depends()):
    try:
        return await service.get_user(user_id)
    except UserNotFound:
        raise HTTPException(404, "User not found")  # Бойлерплейт
```

---

## 6. Настройки (Configuration)

Используйте `pydantic-settings` для управления конфигурацией. Не используйте `.env` файлы напрямую через `os.getenv` в разных местах.

### Правильно

```python
from pydantic_settings import BaseSettings, SettingsConfigDict
from typing import Optional

class Settings(BaseSettings):
    app_name: str = "Awesome API"
    admin_email: str
    debug: bool = False

    database_url: str
    redis_url: str
    redis_password: Optional[str] = None

    secret_key: str
    algorithm: str = "HS256"
    access_token_expire_minutes: int = 30

    model_config = SettingsConfigDict(
        env_file=".env",
        env_file_encoding="utf-8",
        case_sensitive=False
    )

# Зависимость для настроек (синглтон)
_settings = None

def get_settings() -> Settings:
    global _settings
    if _settings is None:
        _settings = Settings()
    return _settings

@router.get("/info")
async def info(settings: Settings = Depends(get_settings)):
    return {"app_name": settings.app_name}
```

---

## Чек-лист для Code Review (FastAPI)

Перед мержем проверьте:

- [ ] **Разделение ответственности:** В роутерах нет SQL-запросов и бизнес-логики (только вызов сервисов).
- [ ] **Инъекция зависимостей:** Все зависимости (БД, сервисы, конфиги, UOW) внедряются через `Depends`, а не создаются внутри функции.
- [ ] **Управление транзакциями:** Используется UOW или явные транзакции с атомарностью. Побочные эффекты (кэш, сообщения) выполняются ПОСЛЕ коммита.
- [ ] **Асинхронность:** Отсутствуют блокирующие вызовы (`requests`, `time.sleep`, `psycopg2`) внутри `async def`. Синхронный код обёрнут в `run_in_threadpool`.
- [ ] **Pydantic v2:** Используется `model_config = ConfigDict(...)` вместо устаревшего `class Config`. Разделены схемы ввода/вывода.
- [ ] **Безопасность:** `response_model` исключает утечку чувствительных полей (пароли, токены). Пароли хешируются асинхронно.
- [ ] **Обработка ошибок:** Используются кастомные доменные исключения и глобальный `exception_handler`. `HTTPException` отсутствует в слоях сервисов и репозиториев.
- [ ] **Типизация:** Функции имеют полные возвращаемые типы (return type hints). Используется `mypy` для проверки.
- [ ] **Тестирование:** Используется `httpx.AsyncClient` для асинхронных тестов. БД мокируется или использует тестовую in-memory базу. Зависимости легко подменяются через DI.
