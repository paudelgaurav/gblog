+++
title = "Asynchronous and Scheduled tasks in Django with django_q and Redis"
tags = [
    "Python",
    "django",
    "ORM",
    "optimization",
    "refactoring",
]
date = 2022-08-02T08:32:10Z

author = "Gaurav Paudel"
+++


![](https://miro.medium.com/max/1400/1*lKovpT1kBDQdv6xsJu8eUQ.jpeg)Photy by [Roman Odintsov](https://www.pexels.com/photo/original-painting-of-gautama-buddha-on-wall-of-shabby-house-4552137/)

> Some long-running tasks may affect the whole experience of software, so it’s better to offload those tasks and move on to perform other tasks.

A bit about the django\_q package. It is indeed one of my favorite third-party packages built for Django. Using this package, we can queue our tasks, run tasks in another cluster and even perform some scheduled cron jobs.

**So let’s get started.**

---

First, we need to install django\_q:  
`pip install django_q`

```python
# settings.py example

INSTALLED_APPS = (  
# other apps
    'django_q',  
)

Q_CLUSTER = {
    'name': 'myproject',
    'workers': 8,
    'recycle': 500,
    'timeout': 60,
    'compress': True,
    'save_limit': 250,
    'queue_limit': 500,
    'cpu_affinity': 1,
    'label': 'Django Q',
    'redis': {
        'host': '127.0.0.1',
        'port': 6379,
        'db': 0, 
    }
}
```

Configure your `settings.py` as above and perform a quick migration with `python manage.py migrate` and finally, run `python manage.py qcluster`.  
But before this, make sure Redis is up and running.  
_For more details about Redis commands visit this_ [_link_](https://www.tutorialspoint.com/redis/redis_commands.htm)_._

---

_Our setup is complete. Let’s move toward some common problems and their solutions._

---

**First problem: _We need to set up a corn job that runs daily and performs a specific task._**

Let’s consider this:  
`Event(title=’PYCON-2077’, remaining_days=20075)`

Now I need to subtract a day from the `remaining_days` right?  
So, let's create a task and schedule it on a daily basis.

```python
from django.db.models import Case, F, When
from django_q.models import Schedule

from my_app.models import Event


def decrease_remaining_days_from_event():
    Event.objects.update(
        remaining_days=Case(
            When(remaining_days__gt=0, then=F("remaining_days") - 1), default=0
        )
    )


Schedule.objects.create(
    func="decrease_remaining_days_from_event",
    schedule_type=Schedule.DAILY,
)
```


But what I really love to do is create a management command that will create a scheduler.

```python

from django.core.management.base import BaseCommand
from django_q.models import Schedule
from django_q.tasks import schedule


class Command(BaseCommand):
    help = "Create CRON scheduler tasks"

    def handle(self, *args, **options):
        schedule(
            name="Decrease Remaining Days From Event",
            func="my_app.tasks.decrease_remaining_days_from_event",
            schedule_type=Schedule.DAILY,
        )
        self.stdout.write(self.style.SUCCESS("Successfully created CRON tasks"))
```

---

**Second Problem: _After a task is completed, I need to send an email (or perform any heavy task), which takes a bit of time and my end-user has to suffer._**

So what we can do in this case is quickly offload those tasks to the `django_q cluster` .

```python

from django.http import HttpResponse
from django_q.tasks import async_task

from my_app.models import Event


def create_event(request, *args, **kwargs):
    Event.objects.create(title="my cool event", remaining_days=10)

    msg = "A new event has been created"
    async_task(
        "django.core.mail.send_mail",
        "New Event",
        msg,
        "from@example.com",
        ["to@example.com"],
    )
    async_task("time.sleep", 10)

    return HttpResponse(status=201)
```

Although the above view `create_event` does time-consuming tasks like sending mail and sleeping for 10 seconds :), the response will be instant as these tasks are offloaded to the cluster and are handled asynchronously.

---

Here is a working example in this [repo](https://github.com/paudelgaurav/django_q_example), feel free to provide feedback and improvements.

> This is just the tip of the iceberg of what you can do with the django\_q package. For more information, do visit its [official site.](https://django-q.readthedocs.io/en/latest/)

--- 

Till then, hasta la vista, Happy coding.
