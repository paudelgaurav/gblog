+++
title = "Tracking user activity in Django(DRF)"
tags = [
    "python",
    "django",
    "DRF",
    "tracking",
    "mixins",
]
date = 2022-06-28T18:19:13Z
author = "Gaurav Paudel"
+++


![](https://miro.medium.com/max/1280/1*rqR2HOLKFlR-o6BQagT0QA.jpeg)Photo by [Parker Coffman](https://unsplash.com/@lackingnothing?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText) on [Unsplash](https://unsplash.com/s/photos/spy?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText)

Here I will be demonstrating a simple way to track API-endpoints hits in Django.

First, let’s build our `ActivityLog` model
```python

from django.db import models
from django.contrib.auth import get_user_model
from django.contrib.contenttypes.models import ContentType
from django.contrib.contenttypes.fields import GenericForeignKey

User = get_user_model()

CREATE, READ, UPDATE, DELETE = "Create", "Read", "Update", "Delete"
LOGIN, LOGOUT, LOGIN_FAILED = "Login", "Logout", "Login Failed"
ACTION_TYPES = [
    (CREATE, CREATE),
    (READ, READ),
    (UPDATE, UPDATE),
    (DELETE, DELETE),
    (LOGIN, LOGIN),
    (LOGOUT, LOGOUT),
    (LOGIN_FAILED, LOGIN_FAILED),
]

SUCCESS, FAILED = "Success", "Failed"
ACTION_STATUS = [(SUCCESS, SUCCESS), (FAILED, FAILED)]


class ActivityLog(models.Model):
    actor = models.ForeignKey(User, on_delete=models.CASCADE, null=True)
    action_type = models.CharField(choices=ACTION_TYPES, max_length=15)
    action_time = models.DateTimeField(auto_now_add=True)
    remarks = models.TextField(blank=True, null=True)
    status = models.CharField(choices=ACTION_STATUS, max_length=7, default=SUCCESS)
    data = models.JSONField(default=dict)

    # for generic relations
    content_type = models.ForeignKey(
        ContentType, models.SET_NULL, blank=True, null=True
    )
    object_id = models.PositiveIntegerField(blank=True, null=True)
    content_object = GenericForeignKey()

    def __str__(self) -> str:
        return f"{self.action_type} by {self.actor} on {self.action_time}"

```
As the model is self-explanatory, here actor refers to the user performing the action of different operations like create, update, login, etc  
And for the “generic” relationships between our ActivityLog model and instances of any other model in our system, we will be using `ContentType` and `GenericForeignKey`. (Explained well in [Django docs](https://docs.djangoproject.com/en/4.0/ref/contrib/contenttypes/))

Now, we need to create a mechanism so that whenever an API endpoint is hit an instance of this model is created.
```python
import logging

from django.conf import settings
from django.contrib.contenttypes.models import ContentType

from rest_framework.exceptions import ValidationError

from .models import ActivityLog, READ, CREATE, UPDATE, DELETE, SUCCESS, FAILED


class ActivityLogMixin:
    """
    Mixin to track user actions
    :cvar log_message:
        Log message to populate remarks in LogAction
        type --> str
        set this value or override get_log_message
        If not set then, default log message is generated
    """

    log_message = None

    def _get_action_type(self, request) -> str:
        return self.action_type_mapper().get(f"{request.method.upper()}")

    def _build_log_message(self, request) -> str:
        return f"User: {self._get_user(request)} -- Action Type: {self._get_action_type(request)} -- Path: {request.path} -- Path Name: {request.resolver_match.url_name}"

    def get_log_message(self, request) -> str:
        return self.log_message or self._build_log_message(request)

    @staticmethod
    def action_type_mapper():
        return {
            "GET": READ,
            "POST": CREATE,
            "PUT": UPDATE,
            "PATCH": UPDATE,
            "DELETE": DELETE,
        }

    @staticmethod
    def _get_user(request):
        return request.user if request.user.is_authenticated else None

    def _write_log(self, request, response):
        status = SUCCESS if response.status_code < 400 else FAILED
        actor = self._get_user(request)

        if actor and not getattr(settings, "TESTING", False):
            logging.info("Started Log Entry")

            data = {
                "actor": actor,
                "action_type": self._get_action_type(request),
                "status": status,
                "remarks": self.get_log_message(request),
            }
            try:
                data["content_type"] = ContentType.objects.get_for_model(
                    self.get_queryset().model
                )
                data["content_object"] = self.get_object()
            except (AttributeError, ValidationError):
                data["content_type"] = None
            except AssertionError:
                pass

            ActivityLog.objects.create(**data)

    def finalize_response(self, request, *args, **kwargs):
        response = super().finalize_response(request, *args, **kwargs)
        self._write_log(request, response)
        return response
```
Above `ActivityLogMixin`override. `finalize_response`which is provided by rest\_framework’s APIView which means we can use this mixin with APIView and also with viewsets.

Wait, we are tracking every API hit but the successful logins and failed attempts are yet to be tracked. Let’s add a `signals.py` file to catch `user_logged_in` , `user_login_failed`signals.


```python
from django.contrib.auth.signals import user_logged_in, user_login_failed
from django.dispatch import receiver

from .models import ActivityLog,  LOGIN, LOGIN_FAILED


def get_client_ip(request):
    x_forwarded_for = request.META.get("HTTP_X_FORWARDED_FOR")
    return (
        x_forwarded_for.split(",")[0]
        if x_forwarded_for
        else request.META.get("REMOTE_ADDR")
    )


@receiver(user_logged_in)
def log_user_login(sender, request, user, **kwargs):
    message = f"{user.full_name} is logged in with ip:{get_client_ip(request)}"
    ActivityLog.objects.create(actor=user, action_type=LOGIN, remarks=message)


@receiver(user_login_failed)
def log_user_login_failed(sender, credentials, request, **kwargs):
    message = f"Login Attempt Failed for email {credentials.get('email')} with ip: {get_client_ip(request)}"
    ActivityLog.objects.create(action_type=LOGIN_FAILED, remarks=message)
```

Now, we are ready to use the above things in our existing codebase let me show you an example.

```python
from rest_framework.views import APIView
from rest_framework.decorators import api_view
from rest_framework.response import Response
from rest_framework.viewsets import ReadOnlyModelViewSet

from .mixins import ActivityLogMixin

from .models import Post, Article
from .serializers import PostSerializer


class ArticleListView(ActivityLogMixin, APIView):
    def get(self, request, *args, **kwargs):
        return Response({"articles": Article.objects.values()})


class PostReadOnlyViewSet(ActivityLogMixin, ReadOnlyModelViewSet):
    queryset = Post.objects.all()
    serializer_class = PostSerializer

    def get_log_message(self, request) -> str:
        return f"{request.user} is reading blog posts"
```

Here, `ArticleListView`and `PostReadOnlyViewSet`are using `ActivityLogMixin`hence their endpoints hit will be tracked.

Did you notice an extra method `get_log_message`in `PostReadOnlyViewSet`to build its own custom message?

You can provide `log_message`or override `get_log_message` to populate remarks in `LogAction`. If not set then, a default log message is generated

Here is the working example in this [_repo_](https://github.com/paudelgaurav/django-user-activity), feel free to provide feedback and improvements. In the near future, we will be moving logic from mixins to middleware, will use a separate log-based database to store activity logs, etc but for now, let’s Keep It Simple Stupid

Happy coding
