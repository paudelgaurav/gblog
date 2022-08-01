+++
title = "Django optimization(Database)"
tags = [
    "Python",
    "django",
    "ORM",
    "optimization",
    "refactoring",
]
date = 2022-06-29T18:19:13Z

author = "Gaurav Paudel"
+++

![](https://miro.medium.com/max/1400/1*5cSylV22q9dghIEax3fqnA.jpeg)Photo by [Alex Blăjan](https://unsplash.com/@alexb?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText) on [Unsplash](https://unsplash.com/s/photos/slow?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText)

> The major bottleneck lies in our database portion so, we will be focusing on optimizing our database queries and lookups.

For the example purpose, I will be creating some simple E-commerce models

```
class Category(models.Model):  
    name = models.CharField(max_length=150)

class Product(models.Model):  
    name = models.CharField(max_length=150)  
    category = models.ForeignKey(Category, on_delete=models.CASCADE,   
                   related_name='products')  
    price = models.FloatField()  
class Order(models.Model:  
     products = models.ManyToManyField(Product)  
     total_price = models.FloatField()
```

**N+1 Problem:**  
Let’s look into the (N+1) query problem

```
products = Product.objects.all()  
for product in products:  
    print(product.category.name)  

```

The above example suffers from an (N+1) problem as it makes one more DB lookup to fetch the category for each product. Let's optimize it by using `select_related` to join the Category table as well.

```
products = Product.objects.all().select_related('category')  
for product in products:  
    print(product.category.name)
```

Now category is already available to us as we have performed a join operation while fetching our products.

Simple things like `select_related`and `prefetch_related` can make your query faster. An easy rule to remember when to use is when you’re traversing a single table with relations like `ForeignKey` and `OnetoOne` use `select_related` or `prefetch_related` for multi-table traversal like `ManyToMany` and reversed `ForeignKey` relation

**Laziness of QuerySet**:  
QuerySet in Django is lazy `Model.objects.all()` returns a lazy QuerySet. which means that just calling `all()` here will not perform a database query. The database query is not performed until you access the data, for example by iterating over it (can happen in template code) or calling update, delete, etc.  
So chaining many filters like

```
queryset = Products.objects.filter(price=500)  
queryset = queryset.filter(category__name='electronics')  
print(queryset)  

```

In the above example database query is performed only once when we invoke the print function, and multiple filters or order_by expression will be converted into a single SQL query by ORM. (awesome right ?)  
Keeping this in mind, we should not evaluate a QuerySet until we use it.  
Some common mistakes and bad practices will be, using `len()`, `bool()`, or `list()` with queryset which enforces evaluation of QuerySet.

**Writing optimized ORM queries:**  
_Case 1: When you need to update many rows of a table in the database_  
In this case, use`Queryset.update` rather than updating each instance.  
For eg: I need to update a certain field of my model in a certain condition

```
  
Order.objects.filter(paid=True).update(paid_at=timezone.now())
```

_Case 2: When you need to update a certain field by adding or substracting with its value_

```
  
product = Product.objects.get(foo=bar)  
product.quantity = F(‘quantity’) + 10  
product.save(update_fields=[‘quantity’])  

```

In the above example, I didn’t need to fetch the current value of quantity for a specific product hence saving a database lookup with the `F()` expression and I even use `update_fields` inside my `save()` method to particularly update that single field.

Case 3: When you need to count the total number of items or find whether it exists or not

```
products = Products.objects.all()  
print(f'There are {len(products)} in the Product table')if len(products):  
    print('Products exists in the database')
```

Here we use python to count the total number of products which is bad. Lets’s write a more optimized code with help of QuerySet functions.

```
print(f'There are {Products.objects.all().count()} in the Product table')if Products.objects.all().exists():  
    print('Product exists in the database')
```

**Proper use of Aggregate functions provided by SQL**:  
There are many cases where we need to perform some aggregation calculations in our table. Let SQL do that job rather than python as SQL is way faster for such cases.  
For eg, we need to calculate the average price of our products in our Product model

```
products = Product.objects.all()  
total_price = 0  
for product in products:  
    total_price += product.price  
avg_price = total_price / products.count()
```

The above example is slower as we’re doing our heavy lifting in python, a more optimized solution would be

```
**from** **django.db.models** **import** Avg  
products = Products.objects.all().aggregate(Avg('price'))  
-> {'price__avg': _averge_price_of_all_products_}
```

Django's official [documentation](https://docs.djangoproject.com/en/4.0/topics/db/aggregation/) provides a detailed explanation regarding aggregations

These are the basic database optimization guide for Django and I will be posting a bit more advanced optimization guides which will include (pythonic code, caching, and some scaling approaches) in the upcoming days.

Till then, happy coding and see you soon.
