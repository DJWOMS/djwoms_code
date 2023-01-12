# Как сократить количество запросов в базу при связи Foreign Key в Django

## Идея
Нужно забрать из БД все товары и категории на которые ссылаются товары.

## Задача
Сократить количество запросов в базу данных при получении данные из связанных 
таблиц.

Есть две модели, категории и товары. Товары связаны с категориями Foreign Key.
```py linenums="1" title="myapp/models.py"
from django.db import models


class Category(models.Model):
   name = models.CharField(max_length=20)


class Product(models.Model):
   title = models.CharField(max_length=50)
   description = models.TextField()
   price = models.DecimalField(max_digits=6, decimal_places=2)
   category = models.ForeignKey(Category, on_delete=models.CASCADE)
```

Предположим что в базе данных 10 категорий и 10 товаров.

При выводе товаров нужно отображать `id` и `name` категории. Для этого в serializer
товара добавлен serializer категории.
```py linenums="1" title="myapp/serializers.py"
from rest_framework import serializers

from .models import Category, Product


class CategorySerializer(serializers.ModelSerializer):
   class Meta:
       model = Category
       fields = '__all__'


class ProductSerializer(serializers.ModelSerializer):
   category = CategorySerializer()

   class Meta:
       model = Product
       fields = '__all__'
```

Из базы данных получаем все товары.

```py linenums="1" title="myapp/views.py"
from rest_framework import viewsets

from .models import Product
from .serializers import ProductSerializer


class ProductView(viewsets.ReadOnlyModelViewSet):
   queryset = Product.objects.all()
   serializer_class = ProductSerializer
```

## Проблема
Выполниться 11 запросов в базу данных. Один для того, чтобы забрать все продукты и еще 10 для каждого 
продукта, чтобы получить информацию о категории.
Получается чем больше будет записей в таблице продуктов, тем больше будет запросов в БД.
Сколько товаров столько запросов.

SQL для получения данных категории для каждого продукта. 
```sql
SELECT "shop_category"."id",
       "shop_category"."name"
FROM "shop_category"
WHERE "shop_category"."id" = 1

SELECT "shop_category"."id",
       "shop_category"."name"
FROM "shop_category"
WHERE "shop_category"."id" = 2

...

SELECT "shop_category"."id",
       "shop_category"."name"
FROM "shop_category"
WHERE "shop_category"."id" = 10
```

## Решение
Используем методом `select_related` указав модель категорий. В таком случае django orm выполнить 
JOINs нужной таблицы. Все нужные данные из связанных таблиц подтянуться сразу.
В базу будет сделан один запрос.
```py linenums="1" hl_lines="8" title="myapp/views.py"
from rest_framework import viewsets

from .models import Product
from .serializers import ProductSerializer


class ProductView(viewsets.ReadOnlyModelViewSet):
   queryset = Product.objects.select_related('category').all()
   serializer_class = ProductSerializer
```
В SQL теперь явно указан JOIN таблицы категорий.

```sql hl_lines="9"
SELECT "shop_product"."id",
       "shop_product"."title",
       "shop_product"."description",
       "shop_product"."price",
       "shop_product"."category_id",
       "shop_category"."id",
       "shop_category"."name"
FROM "shop_product"
INNER JOIN "shop_category" ON ("shop_product"."category_id" = "shop_category"."id")
```


!!! note success "select_related"

    Чтобы получить данные из связанных таблиц используйте метод `select_related`, в котором перечисли, 
    те модели которые нужно джойнить.



