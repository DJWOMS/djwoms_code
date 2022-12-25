# Как редактировать и манипулировать загруженными изображениями перед сохранением в Django

## Идея
Иногда нам нужно выполнить определенные операции с загруженными пользователем изображениями 
перед их сохранением в Django. Эти операции могут включать обрезку изображений, поворот, 
создание эскизов или сжатие.

## Задача
Создать миниатюру изображения которое загрузил пользователь.

## Решение
Нам понадобится библиотека Pillow. Установим её.

```
pip install Pillow
```

### Модель
Ниже приведен пример модели с двумя полями — `image` и `thumbnail`. В которых будет храниться
исходное изображение и миниатюра.

```py linenums="1" title="myapp/models.py"
from django.db import models


class Image(models.Model):
    image = models.ImageField()
    thumbnail = models.ImageField()
```


### Создание миниатюр
Создадим новую функцию `make_thumbnail()`, которая будет создавать миниатюры для данного изображения. 
Эту функцию можно написать в файле `models.py` или в вашем сервисе. Я создам файл `services.py`.

```py linenums="1" title="myapp/services.py"
from io import BytesIO
from django.core.files import File
from PIL import Image


def make_thumbnail(image, size=(100, 100)):
    """Создает миниатюры заданного размера"""
    im = Image.open(image)
    im.convert('RGB')
    im.thumbnail(size)
    thumb_io = BytesIO()
    im.save(thumb_io, 'JPEG', quality=85)
    thumbnail = File(thumb_io, name=image.name)
    return thumbnail
```

Пояснения к коду:

- **`im.convert(RGB)`** - служит для преобразования режима изображения в RGB. Потому что иногда Pillow 
выдает ошибку, когда мы пытаемся сохранить изображение в формате JPEG, если изображение представляет собой 
GIF с ограниченной палитрой.

- **`im.thumbnail(size)`** - это метод класса Image, который масштабирует изображение до заданного 
размера, сохраняя соотношение сторон.

- **`thumbnail = File(thumb_io, name=image.name)`** - создает понятный для Django объект File, который мы 
можем использовать в качестве значения для модели `Image`.


### Вызов функции из модели
Вызываем функцию в методе `save()` нашей модели. Полученную миниатюру присваиваем полю `thumbnail`.

```py linenums="1" hl_lines="7-9" title="myapp/models.py"
from services import make_thumbnail

class Image(models.Model):
    image = models.ImageField()
    thumbnail = models.ImageField()

    def save(self, *args, **kwargs):
        self.thumbnail = make_thumbnail(self.image, size=(100, 100))
        super().save(*args, **kwargs)
```

!!! note success ""

    Вы можете адаптировать и изменить этот код, чтобы выполнять любые операции с изображениями, 
    например, обрезать их.

