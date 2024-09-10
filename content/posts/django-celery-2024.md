---
title: Using Celery with Django for Asynchronous Tasks

date: 2024-08-05
category: tech
tags:
    - Python
keywords:
    - Python
    - Django
    - Api
    - Celery
---

## Introduction

Celery is a powerful, distributed task queue that can be used with Django to handle asynchronous tasks and background processing. This guide will walk you through setting up Celery with Django and demonstrate how to use it effectively in your projects.

## Table of Contents

1. [Installation](#installation)
2. [Configuration](#configuration)
3. [Creating Tasks](#creating-tasks)
4. [Running Tasks](#running-tasks)
5. [Periodic Tasks](#periodic-tasks)
6. [Monitoring](#monitoring)
7. [Best Practices](#best-practices)

## Installation

First, you need to install Celery and its dependencies:

```bash
pip install celery redis django-celery-results
```

We'll use Redis as our message broker, but you can also use RabbitMQ or other brokers supported by Celery.

## Configuration

### Django Settings

Add the following to your Django `settings.py`:

```python
# Celery settings
CELERY_BROKER_URL = 'redis://localhost:6379'
CELERY_RESULT_BACKEND = 'django-db'
CELERY_ACCEPT_CONTENT = ['application/json']
CELERY_RESULT_SERIALIZER = 'json'
CELERY_TASK_SERIALIZER = 'json'
CELERY_TIMEZONE = 'UTC'

# Add django_celery_results to INSTALLED_APPS
INSTALLED_APPS = [
    ...
    'django_celery_results',
]
```

### Celery Configuration

Create a new file `celery.py` in your Django project directory (where `settings.py` is located):

```python
import os
from celery import Celery

# Set the default Django settings module for the 'celery' program.
os.environ.setdefault('DJANGO_SETTINGS_MODULE', 'your_project.settings')

app = Celery('your_project')

# Use a string here to make sure the worker doesn't serialize the object.
app.config_from_object('django.conf:settings', namespace='CELERY')

# Load task modules from all registered Django app configs.
app.autodiscover_tasks()
```

Update your project's `__init__.py`:

```python
from .celery import app as celery_app

__all__ = ('celery_app',)
```

## Creating Tasks

Create a `tasks.py` file in your Django app directory:

```python
from celery import shared_task

@shared_task
def add(x, y):
    return x + y

@shared_task
def multiply(x, y):
    return x * y

@shared_task(bind=True)
def debug_task(self):
    print(f'Request: {self.request!r}')
```

## Running Tasks

You can run tasks in your Django views or anywhere else in your Django code:

```python
from django.http import HttpResponse
from .tasks import add, multiply

def index(request):
    add.delay(7, 8)
    multiply.delay(7, 8)
    return HttpResponse("Tasks are running in the background!")
```

The `delay()` method is used to execute the task asynchronously.

## Periodic Tasks

To set up periodic tasks, you can use Celery Beat. First, install the Django Celery Beat package:

```bash
pip install django-celery-beat
```

Add it to your `INSTALLED_APPS`:

```python
INSTALLED_APPS = [
    ...
    'django_celery_beat',
]
```

Then, you can define periodic tasks in your Django admin interface or in your code:

```python
from celery.schedules import crontab
from django.conf import settings

settings.CELERY_BEAT_SCHEDULE = {
    'add-every-minute-contrab': {
        'task': 'your_app.tasks.add',
        'schedule': crontab(minute='*/1'),
        'args': (16, 16),
    },
}
```

## Monitoring

You can use Flower, a web-based tool for monitoring Celery:

```bash
pip install flower
celery -A your_project flower
```

Then visit `http://localhost:5555` to see the Flower dashboard.

## Best Practices

1. **Use `shared_task` decorator**: This allows your tasks to be used by multiple apps.

2. **Handle exceptions**: Wrap your task logic in try-except blocks to handle errors gracefully.

3. **Set timeouts**: Use the `soft_time_limit` and `time_limit` options to prevent tasks from running indefinitely.

4. **Use task states**: Update task states to track progress and provide feedback.

5. **Use task priority**: Set task priorities to ensure critical tasks are processed first.

6. **Optimize database queries**: Ensure your tasks are not causing N+1 query problems.

7. **Use task queues**: Separate your tasks into different queues based on their nature (e.g., email queue, data processing queue).

Example of a more robust task:

```python
from celery import shared_task
from celery.utils.log import get_task_logger
from django.core.mail import send_mail

logger = get_task_logger(__name__)

@shared_task(bind=True, max_retries=3, soft_time_limit=20)
def send_notification_email(self, user_id, subject, message):
    try:
        user = User.objects.get(id=user_id)
        send_mail(
            subject,
            message,
            'from@example.com',
            [user.email],
            fail_silently=False,
        )
    except User.DoesNotExist:
        logger.error(f"User with id {user_id} does not exist")
    except Exception as exc:
        logger.error(f"Error sending email to user {user_id}: {exc}")
        raise self.retry(exc=exc, countdown=60)  # Retry after 1 minute
```

