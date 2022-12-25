# Как переопределить сохранение формы Django admin Inline?
## Идея
Пользователь на сайте может отправить сообщение и видеть всю переписку с менеджером.
Менеджер видит переписку и отправляет сообщения, используя Django админку. Для отображения переписки 
используется класс `StackedInline`.

## Задача
Когда менеджер отправляет сообщение, то текущий пользователь, должен автоматически подставиться при 
сохранении. У менеджера нет возможности выбрать пользователя который отправил сообщение.

## Модели
Переписка ведется в некой группе.

Есть две модели `Group` и `Message`. Модель `Message` связана с `Group`.
```py linenums="1" title="myapp/models.py"
from django.contrib.auth.models import User
from django.db import models


class Group(models.Model):
    name = models.CharField(max_length=100)
    create_at = models.DateTimeField(auto_now_add=True)

    def __str__(self):
        return self.name


class Message(models.Model):
    text = models.TextField()
    group = models.ForeignKey(Group, on_delete=models.CASCADE, related_name='messages')
    user = models.ForeignKey(User, on_delete=models.CASCADE)
```

## Админка

В админке зарегистрирована только одна модель `Group`. Для отображения сообщений в группе, используем 
класс `MessageInline` который указан в классе `GroupAdmin` код:

```py linenums="1" title="myapp/admin.py"
from django.contrib import admin
from .models import Group, Message


class MessageInline(admin.StackedInline):
    model = Message
    extra = 1


@admin.register(Group)
class GroupAdmin(admin.ModelAdmin):
    list_display = ('id', 'name')
    inlines = [MessageInline]
```
![admin](/assets/django/admin/custom_saving_inline/get_user.jpg)

Видим что у менеджера есть возможность выбрать пользователя от имени которого отправить сообщение. 

Лешим его такой возможности, добавив `readonly_fields`.

```py linenums="1" hl_lines="8" title="myapp/admin.py"
from django.contrib import admin
from .models import Group, Message


class MessageInline(admin.StackedInline):
    model = Message
    extra = 1
    readonly_fields = ('user',)


@admin.register(Group)
class GroupAdmin(admin.ModelAdmin):
    list_display = ('id', 'name')
    inlines = [MessageInline]
```
![admin](/assets/django/admin/custom_saving_inline/no_user.jpg)


Но теперь при попытке сохранить (отправить) сообщение, мы получим ошибку:
```
IntegrityError
NOT NULL constraint failed: myapp_message.user_id
```

Мы не указали к какому юзеру принадлежит данное сообщение. 

## Решение
Решим данную проблему, нужно переопределить метод `save_formset` и добавить текущего пользователя.

```py linenums="1" hl_lines="16-21" title="myapp/admin.py"
from django.contrib import admin
from .models import Group, Message


class MessageInline(admin.StackedInline):
    model = Message
    extra = 1
    readonly_fields = ('user',)


@admin.register(Group)
class GroupAdmin(admin.ModelAdmin):
    list_display = ('id', 'name')
    inlines = [MessageInline]

    def save_formset(self, request, form, formset, change):
        instances = formset.save(commit=False)
        for instance in instances:
            instance.user = request.user
            instance.save()
        formset.save_m2m()
```

!!! note warning "Обратите внимание"

    При изменении сообщения пользователя, автоматически присвоится пользователь менеджера.
