# Django: Monolith -> MicroServices

## Как перенести Монолит Django в Микросервисы.

**Содержание**:  
· Шаг 1: Начиная с модульного монолита  
∘ Рефакторинг в понятные приложения Django  
∘ Перенести бизнес-логику в сервисы  
∘ Устранить жесткие зависимости с помощью сигналов  
· Шаг 2: Выбрать правильный первый микросервис  
· Шаг 3: Превратить его в микросервис Django  
· Шаг 4: Общая база данных или отдельные базы данных  
∘ Вариант 1: Сохранить одну базу данных (временно)  
∘ Вариант 2: Разделить на отдельные базы данных  
∘ Вариант 3: Синхронизация на основе событий  
· Шаг 5: Связь между микросервисами Django  
∘ REST API (синхронный, простой в запуске)  
∘ Асинхронный обмен сообщениями (масштабируемый, несвязанный)  
∘ Архитектура на основе событий (следующий уровень несвязанности)  
· Шаг 6: Развертывание микросервисов Django правильным способом  
· Шаг 7: Централизованная Аутентификация и безопасность  
· Шаг 8: Ведение журнала, мониторинг и наблюдаемость  
· Шаг 9: Ожидаемые реальные проблемы  
· Заключение  

Шаг 1: Начиная с модульного монолита
Не переходить к микросервисам, пока монолит не станет "здоровым".
Попытка разделить запутанную, неструктурированную кодовую базу только умножит проблемы.

Рефакторинг в чистые приложения Django
Каждое приложение Django должно быть четко привязано к бизнес-домену.

Пример структуры:
```html
/project
    /users
    /orders
    /payments
    /notifications
```

"Плохой" знак: если все модели находятся в ```models.py``` одном приложении, вы еще не готовы.

Перенесте бизнес-логику в службы
Представления не предназначены для хранения бизнес-логики. Модели тоже.

Переместить настоящую логику в ```services.py```. Это сделает ее пригодной для повторного использования в разных проектах и, в конечном итоге, в разных сервисах.
```python
# users/services.py 
из django.contrib.auth.models import User 

def  create_user ( data ): 
    return User.objects.create(**data)
```

Теперь логика больше не привязана к системе представлений Django — это просто Python. И это именно то, что понадобится, когда это станет отдельным микросервисом.

Устранение жестких зависимостей с помощью сигналов
Монолиты часто страдают от глубоко связанного кода.
Например, размещение заказа может вызвать отправку письма по электронной почте. Но это не значит, что  ```orders``` приложение должно зависеть от ```notifications```.

Заменить прямые вызовы функций сигналами.

До:
```python
# orders/views.py 

def  place_order ( request ): 
    order = Order.objects.create(user=request.user) 
    send_order_email(order)
```
После:
```python
# orders/signals.py 

@receiver( post_save, sender=Order ) 
def  handle_order_created ( sender, instance, **kwargs ): 
    send_order_email(instance)
```
Теперь, если уведомления позже перейдут в микросервис, нужно будет только опубликовать событие — логика уже разделена.

Шаг 2: Выбрать правильный первый микросервис
Не все должно быть микросервисом. Начать с чего-то, что:

* Имеет четкие границы
* Не делится слишком большим объемом данных
* Не блокирует остальную часть системы в случае сбоя
* 
Вот три замечательных кандидата:

**AuthService**
Управляет регистрацией пользователей, входом в систему, управлением токенами. Django уже сохраняет аутентификацию четко разделенной — что делает это безопасной отправной точкой.

**NotificationService**
Электронные письма и SMS не обязательно должны быть синхронными. Переместить их в фоновый обработчик или микросервис на базе API.

**PaymentService**
Платежи часто интегрируются со сторонними API, требуют более строгой безопасности и имеют собственные рабочие процессы. Идеально для изоляции.

Шаг 3: Превратить в микросервис Django
Мы не меняем фреймворки. Django отлично подходит для микросервисов — особенно если вы поддерживаете чистоту.

Начать с перемещения приложения в новый проект Django.
```
/auth_service 
    /users 
        models.py 
        views.py 
        services.py 
    settings.py 
    urls.py
```
Затем предоставить доступ к службе с помощью Django REST Framework .
```python
# users/views.py

from rest_framework.decorators import api_view
from rest_framework.response import Response
from .services import create_user

@api_view(["POST"])
def register_user(request):
    user = create_user(request.data)
    return Response({"id": user.id, "email": user.email})
```
В вашем монолите замените внутренний вызов HTTP-запросом:
```python
# monolith/users/services.py

import requests

def create_user(data):
    response = requests.post("http://auth-service/api/register/", json=data)
    return response.json()
```
Монолит больше не привязан к пользовательской модели — он просто вызывает службу.

Шаг 4: Общая база данных и отдельные базы данных
Разделение баз данных — это то, из-за чего большинство переходов терпят неудачу.

Существует три способа обработки данных:

Вариант 1: сохранить одну базу данных (временно)
Да, это противоречит догме микросервисов. Но это помогает при переходе.

Можно разделить логику и продолжать использовать базу данных монолита — при условии, что каждая служба будет обращаться только к своим собственным таблицам.

В Django это можно сделать, назначив приложения разным маршрутизаторам баз данных.

Вариант 2: Разделить на отдельные базы данных
Каждая служба владеет своими данными. Если другой службе нужен доступ, она запрашивает его через API.

Никаких кросс-сервисных запросов. Никаких общих миграций.

Вариант 3: Синхронизация на основе событий
Сервисы публикуют и потребляют события.

Пример: Когда создается заказ, он публикует ```order_created``` событие.

Служба уведомлений отслеживает это событие и отправляет электронное письмо, даже не затрагивая базу данных заказов.

Для этого можно использовать Redis Pub/Sub , Kafka или RabbitMQ .

Шаг 5: Взаимодействие между микросервисами Django
Когда все жило внутри одного проекта, вызывались функции или импортировали классы. С микросервисами это ушло. Теперь сервисам нужно общаться друг с другом по сети.

Но не все коммуникации одинаковы.

REST API (синхронный, простой в запуске)
Django REST Framework (DRF) делает это простым. Сервисы предоставляют конечные точки, а другие потребляют их через requests.

Пример:
```python
# from monolith -> auth service

def fetch_user_info(user_id):
    response = requests.get(f"http://auth-service/api/users/{user_id}/")
    return response.json()
```
Просто, но : если служба аутентификации выйдет из строя, этот вызов не будет выполнен. Так что…

* Используйте повторные попытки и тайм-ауты
* Потерпеть неудачу достойно
* Используйте автоматические выключатели в более сложных системах

Асинхронный обмен сообщениями (масштабируемый, несвязанный)
Для фоновых процессов, таких как отправка электронных писем или обновление аналитики, используйте Celery + Redis/RabbitMQ .

Вместо того чтобы вызывать другую службу напрямую, опубликуйте событие или поставьте задачу в очередь.

В монолите:
```python
from notifications.tasks import send_email_notification

send_email_notification.delay(order_id)
```
В микросервисе:
```python
@shared_task
def send_email_notification(order_id):
    order = Order.objects.get(id=order_id)
    # logic to send email
```

Не нужно ждать завершения задачи. Система остается быстрой и развязанной .

Архитектура, управляемая событиями (развязка следующего уровня)
Рассмотреть возможность использования таких инструментов, как Kafka или Redis Streams, для создания полностью событийно-управляемой системы.

Сервисы могут публиковать такие события, как order_created, а другие могут подписываться на них, даже не зная, кто это.

Шаг 6: Правильное развертывание микросервисов Django
Каждый микросервис теперь является собственным проектом Django. Так как же их развернуть?

Использовать Docker с первого дня
Контейнеризация обеспечивает согласованность между средами и изолирует зависимости между службами.

Пример Dockerfile:
```
FROM python:3.10
WORKDIR /app
COPY . .
RUN pip install -r requirements.txt
CMD ["gunicorn", "project.wsgi:application", "-b", "0.0.0.0:8000"]
```

Каждая услуга получает свой собственный имидж.

Простой, воспроизводимый и масштабируемый.

Docker Compose для локальной разработки
Развернуть всю архитектуру микросервисов локально:
```
services:
  auth-service:
    build: ./auth_service
    ports:
      - "8001:8000"

notification-service:
    build: ./notification_service
    ports:
      - "8002:8000"
```

Kubernetes для производства
Как только все масштабируется, переходить на Kubernetes . Он обрабатывает:

* Обнаружение услуг / Service discovery
* Автомасштабирование / Auto-scaling
* Проверки здоровья / Health checks
* Последовательные развертывания / Rolling deployments
  
Django хорошо работает в Kubernetes с Gunicorn , PostgreSQL , Redis и Celery, все они контейнеризированы.

Шаг 7: Централизованная аутентификация и безопасность
При наличии нескольких сервисов уже нельзя полагаться на сессии Django. Нужна аутентификация на основе токенов .

Использовать JWT (веб-токены JSON)
JWT идеально подходят для микросервисов:

Токен включает в себя идентификационные данные пользователя.
Каждая служба может проверить токен без обращения к базе данных.
Пример использования djangorestframework-simplejwt:
```python
pip install djangorestframework-simplejwt
# settings.py
 REST_FRAMEWORK = { 
    'DEFAULT_AUTHENTICATION_CLASSES' : ( 
        'rest_framework_simplejwt.authentication.JWTAuthentication' , 
    ) 
}
```
Теперь ваш AuthService выпускает токены, а другие службы проверяют их, используя общие секреты или открытые ключи .

Шаг 8: Ведение журнала, мониторинг и наблюдаемость
Отладка одного приложения Django — это просто. Отладка между службами? Не так уж и сложно.

Необходимо:

* Централизованное ведение журнала (ELK Stack, Loki + Grafana)
* Структурированные журналы (журналы JSON с именами служб, идентификаторами запросов)
* Мониторинг и оповещения (Prometheus + Grafana)
* Распределенная трассировка (OpenTelemetry, Jaeger)

Установите идентификатор корреляции для каждого запроса и передайте его каждой службе. Таким образом, когда что-то сломается, вы сможете отследить весь поток по службам.

Шаг 9: Ожидаемые реальные проблемы

Вот с чем можно столкнутся:

❗ Отладка станет сложнее
Понадобятся журналы из нескольких служб, чтобы понять одну ошибку. Используйте идентификаторы трассировки и централизованный поиск по журналам .

❗ Задержка в сети становится реальностью
Вызов служб по HTTP не бесплатный. Поддерживайте экономность служб , агрессивно кэшируйте и пакетируйте запросы, где это возможно.

❗ Инфраструктура будет расти
Вместо одного приложения теперь надо поддерживать:

* Несколько кодовых баз
* Несколько конвейеров CI/CD
* Больше баз данных
* Больше облачных ресурсов

Переходите на микросервисы только в том случае, если готовы нести эксплуатационные расходы .

❗ Тестирование больше не является простым
Тестов модулей будет недостаточно. Добавить:

* Тесты контрактов (для проверки API между сервисами)
* Интеграционные тесты (за пределами границ сервиса)
* Имитация внешних вызовов обслуживания

Заключение: стоит ли ломать монолит?
Вот правда:

Цель — не микросервисы, а удобство обслуживания. Цель — масштабируемость. Цель — скорость.

Если монолит Django работает хорошо, не разделяйте его только для того, чтобы следовать тенденциям.
Но если команда растет, код запутан, а приложение замедляется — это может быть изменение архитектуры, которое вас спасет.

Начать с малого. Сохранять модульность. Двигаться пошагово. И прежде всего — убедиться, что это стоит сложности.

Удачного кодирования! 💻
