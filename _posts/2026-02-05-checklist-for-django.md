---
layout: post
title: Чек-лист для Django
excerpt_separator: <!--more-->
categories:
  - Разработка
tags:
  - python
  - testing
---

Документ описывает единые стандарты написания кода для Django-проектов. Цель — создание поддерживаемого, безопасного и производительного приложения.

<!--more-->

## 1. Архитектура: "Тонкие Views, Толстые Сервисы"

Бизнес-логика не должна находиться во `views` или `models`. Django MTV (Model-Template-View) паттерн должен расширяться до Service Layer для сложной логики.

| Слой          | Ответственность                                                                            |
| ------------- | ------------------------------------------------------------------------------------------ |
| **Views**     | Принять запрос, валидация входных данных (Serializer/Form), вызов Сервиса, возврат ответа. |
| **Services**  | Бизнес-логика, транзакции, оркестрация моделей. Не знают про HTTP (request/response).      |
| **Models**    | Структура данных, методы уровня инстанса, простые запросы (QuerySets).                     |
| **Selectors** | Сложные запросы на чтение (Getters/Fetchers), не меняют состояние БД.                      |

### Инъекция зависимостей

Сервисы не должны создаваться внутри вьюхи напрямую. Это делает код жестким и усложняет тестирование. Используйте явную передачу зависимостей.

### Управление транзакциями

Операции, изменяющие несколько моделей или имеющие внешние побочные эффекты (API, кэш, письма), должны оборачиваться в `transaction.atomic()`. Письма и задачи отправляются _после_ успешного коммита транзакции через `transaction.on_commit()`.

### Правильно (Service Layer)

```python
# users/services.py
from typing import Optional
from django.db import transaction
from django.core.mail import send_mail
import structlog

class UserService:
    def create_user(self, email: str, password: str) -> User:
        with transaction.atomic():
            # Проверка существования
            if User.objects.filter(email=email).exists():
                raise UserAlreadyExistsError(email)

            user = User(email=email)
            user.set_password(password)
            user.save()

            # Дополнительная бизнес-логика в рамках транзакции
            AuditLog.objects.create(action="user_created", user=user)

            # Отправка письма только после успешного коммита
            transaction.on_commit(
                lambda: send_welcome_email.delay(user.id)
            )

            structlog.get_logger().info("user_created", user_id=user.id)
            return user

# users/views.py
class SignupView(APIView):
    def __init__(self, user_service: Optional[UserService] = None, **kwargs):
        super().__init__(**kwargs)
        self.user_service = user_service or UserService()  # fallback для простоты

    def post(self, request):
        serializer = SignupSerializer(data=request.data)
        serializer.is_valid(raise_exception=True)

        user = self.user_service.create_user(
            email=serializer.validated_data['email'],
            password=serializer.validated_data['password']
        )
        return Response({'user_id': user.id}, status=201)
```

### Неправильно (Fat View)

```python
# users/views.py
def signup_view(request):
    if request.method == 'POST':
        # Вся логика свалена во view
        email = request.POST.get('email')
        if User.objects.filter(email=email).exists():
            return JsonResponse({'error': 'Exists'}, status=400)

        user = User(email=email)
        user.set_password(request.POST.get('password'))
        user.save()

        send_welcome_email.delay(email)  # Прямой вызов здесь
        return JsonResponse({'status': 'ok'})
```

---

## 2. ORM и Производительность

Главная проблема Django ORM — ленивая загрузка и проблема N+1 запросов.

### Правило: Всегда используйте `select_related` и `prefetch_related`

Если вы обращаетесь к связанному полю в цикле или шаблоне, оно должно быть загружено заранее.

### Правильно

```python
# views.py
def order_list(request):
    # select_related для ForeignKey (OneToOne) - JOIN
    # prefetch_related для M2M или обратных связей - отдельный запрос
    orders = Order.objects.select_related(
        'customer'
    ).prefetch_related(
        'items__product'
    ).all()

    return render(request, 'orders.html', {'orders': orders})
```

### Неправильно

```python
def order_list(request):
    orders = Order.objects.all()  # N+1 проблема!
    for order in orders:
        print(order.customer.name)  # Запрос к БД на каждой итерации
```

### Использование Custom QuerySets

Выносите часто используемые фильтры в методы менеджера.

```python
# models.py
class OrderQuerySet(models.QuerySet):
    def paid(self):
        return self.filter(status='paid')

    def recent(self):
        return self.filter(created_at__gte=timezone.now() - timedelta(days=7))

class Order(models.Model):
    objects = OrderQuerySet.as_manager()

# Использование
paid_orders = Order.objects.paid().recent()
```

### Пагинация

Никогда не возвращайте `.all()` без пагинации для эндпоинтов API. Используйте `django.core.paginator.Paginator` или пагинацию DRF.

```python
from django.core.paginator import Paginator

def order_list(request):
    orders = Order.objects.select_related('customer').all()
    paginator = Paginator(orders, 20)  # 20 заказов на страницу
    page_obj = paginator.get_page(request.GET.get('page'))
    return render(request, 'orders.html', {'orders': page_obj})
```

---

## 3. Настройки и Окружение

### Разделение настроек

Не вали все настройки в одну кучу. Разделяй их на осмысленные группы.

### Работа с временем

Всегда держите `USE_TZ = True`. Используйте `from django.utils import timezone` и `timezone.now()` для получения текущего времени. Храните все даты в UTC.

**Никогда не хардкодьте секреты!**

### Правильно

```python
# settings/production.py
import os
from decouple import config

SECRET_KEY = config('DJANGO_SECRET_KEY')
DEBUG = False
USE_TZ = True  # Обязательно для корректной работы с часовыми поясами
ALLOWED_HOSTS = config('ALLOWED_HOSTS', cast=lambda v: [s.strip() for s in v.split(',')])
DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.postgresql',
        'NAME': config('DB_NAME'),
        'USER': config('DB_USER'),
        'PASSWORD': config('DB_PASSWORD'),
        'HOST': config('DB_HOST'),
        'PORT': config('DB_PORT'),
        'CONN_MAX_AGE': 600,  # Пуллинг соединений
    }
}
```

### Неправильно

```python
# settings.py
SECRET_KEY = 'django-insecure-#x@z...'
DEBUG = True  # В проде это дыра в безопасности
```

---

## 4. Миграции

Миграции — это версия вашей схемы БД.

1.  **Не меняйте данные в миграциях**, если это возможно (данные теряются при откате). Используйте `RunPython` с осторожностью.
2.  **Называйте миграции осмысленно**, но лучше полагаться на стандартные имена Django.
3.  **Проверяйте SQL** перед применением: `python manage.py sqlmigrate app_name 0004`.

### Опасные операции

Будьте осторожны с удалением полей и таблицами в продакшене. Django не делает `CASCADE` по умолчанию для некоторых БД, но может заблокировать таблицу на время изменения.

---

## 5. Формы и Валидация (DRF / Django Forms)

Валидация должна происходить на уровне Serializer/Form.

### Правильно

```python
# serializers.py
class UserSerializer(serializers.ModelSerializer):
    class Meta:
        model = User
        fields = ['email', 'age']

    def validate_age(self, value):
        if value < 18:
            raise serializers.ValidationError("Too young")
        return value

    def validate(self, attrs):
        # Кросс-полевая валидация
        if attrs['email'].endswith('@spam.com') and attrs['age'] < 21:
            raise serializers.ValidationError("Restricted domain for minors")
        return attrs
```

### Неправильно

```python
# Валидация внутри View или save()
def create(self, request):
    age = request.data.get('age')
    if age < 18:
        return Response({'error': 'Too young'}, status=400) # Плохо, не структурировано
```

---

## 6. Безопасность (Security Checklist)

- [ ] `DEBUG = False` в проде.
- [ ] `SECURE_SSL_REDIRECT = True` (перенаправление на HTTPS).
- [ ] `SESSION_COOKIE_SECURE = True` и `CSRF_COOKIE_SECURE = True`.
- [ ] `SECURE_HSTS_SECONDS = 31536000` (HTTP Strict Transport Security).
- [ ] Используйте `CORS` через `django-cors-headers` только для разрешенных доменов.
- [ ] Никогда не передавайте ID сущностей в URL, если пользователю не положено их видеть. Используйте `UUID` или проверку прав доступа.

### Глобальная обработка исключений

Используйте централизованный обработчик для трансляции бизнес-исключений в HTTP-ответы. Никогда не показывайте клиенту traceback в production.

```python
# utils/exceptions.py
from rest_framework.views import exception_handler

def custom_exception_handler(exc, context):
    response = exception_handler(exc, context)

    if isinstance(exc, UserAlreadyExistsError):
        return Response(
            {'error': str(exc), 'code': 'user_already_exists'},
            status=status.HTTP_409_CONFLICT
        )

    return response
```

---

## 7. Асинхронность (Django 4.1+)

Если вы используете async views, убедитесь, что драйвер БД поддерживает это (asyncpg для PostgreSQL) или используйте `sync_to_async` правильно.

### Правильно

```python
# settings.py
# Для async views нужна отдельная настройка или использование asgiref.sync
import asgiref.sync

# views.py
async def my_async_view(request):
    # Безопасный вызов синхронного ORM
    users = await sync_to_async(list)(User.objects.all())
    return JsonResponse({'count': len(users)})
```

---

## Чек-лист для Code Review (Django)

Перед мержем проверьте:

- [ ] **Бизнес-логика** вынесена из Views в Services/Selectors.
- [ ] **Инъекция зависимостей** используется для тестируемости сервисов.
- [ ] **Транзакции** обернуты в `atomic()` с `on_commit()` для побочных эффектов.
- [ ] **N+1 запросы** отсутствуют (использованы `select_related` / `prefetch_related`).
- [ ] **Секретные данные** берутся из `.env` или Variables (не захардкожены).
- [ ] **Модели**: поля имеют `verbose_name`, `help_text` (для админки и документации).
- [ ] **Валидация** находится в Serializer/Form, а не разбросана по коду.
- [ ] **Миграции**: нет "черных дыр" (удаление данных без бэкапа, блокирующие операции).
- [ ] **Admin**: кастомизирован для удобства менеджеров (list_display, search_fields), но нет прямой бизнес-логики.
- [ ] **Security**: `SECRET_KEY` скрыт, `DEBUG=False` в проде конфиге.
- [ ] **Обработка ошибок**: бизнес-исключения маппятся на HTTP-коды централизованно.
- [ ] **Тесты**: используют `TestCase` или `pytest-django` с правильной изоляцией БД.
