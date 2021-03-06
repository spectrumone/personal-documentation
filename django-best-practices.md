#Django Best Practices
<i>because I keep forgetting to use these.</i>

##settings.py
* always use environment variables; if it gets tedious always writing those, you can place it in the activate script of the virtualenv youre using.
* on creating **SECRET_KEY**
```python
import os
import random

SECRET_KEY = os.environ.get("SECRET_KEY", "".join(random.choice(string.printable) for i in range(40)))
```
* Handle missing secret key Exception using this:
```python
import os
from django.core.exceptions import ImproperlyConfigured

def get_env_variable(var_name):
    try:
        return os.environ[var_name]
    except KeyError:
        error_msg = "Set the {} environment variable.".format(var_name)
        raise ImproperlyConfigured(error_msg)
```
* On defining BASE_DIR like setting, use the package [Unipath](https://github.com/mikeorr/Unipath)
```python
from unipath import Path

BASE_DIR = Path(__file__).ancestor(3)
MEDIA_ROOT = BASE_DIR.child("media")
STATIC_ROOT = BASE_DIR.child("static")
STATICFILES_DIRS = (
    BASE_DIR.child("assets"),
)
```
* use package [dj_database_url](https://github.com/kennethreitz/dj-database-url) for creating db urls.
```python
DATABASES = {
    'default': dj_database_url.config(default="postgres:///channels-example", conn_max_age=500)
}
```

##models.py
* Create Generic TimeStampeModel to be inherited by almost all modules
```python
#core/models.py
from django.db import models

class TimeStampedModel(models.Model):
    """
    An abstract base class model that provides self-updating ``created`` and ``modified`` fields
    """
    created = models.DateTimeField(auto_now_add=True)
    modified = models.DateTimeFIeld(auto_now=True)
    
    class Meta:
        abstract = True
```
* Use model manager to create custom queries. The `use_for_related_fields` will make the Manager available on all relations that point to the model on which you defined this manager as the default manager.
```python
from django.db imporot models
from django.utils import timezone

class PublishedManager(models.Manager):

    use_for_related_fields = True
    
    def published(self, **kwargs):
        return self.filter(pub_date__lte=timezone.now(), **kwargs)
        
class FlavorReview(models.Model):
    review = models.CharField(max_length=255)
    pub_date = models.DateTimeField()
    
    # add our custom model manager
    objects = PublishedManager()

```
Usage:
```python
>>> from reviews.models import FlavorReview
>>> FlavorReview.objects.count()
3
>>> FlavorReview.objects.published().count()
1
```
* When you constantly call a foreignkey on your queries, it might be best to add to override `get_queryset` and add `select_related` to it. Count the number of queries if you want to check that it works. `select_related` only works for many-to-one and one-to-one relationships.
```python
class CountyManager(models.Manager):
    def get_queryset(self):
        return super(CountyManager, self).get_queryset().select_related('state')
```

##urls.py
* Don't reference views as strings in URLConfs. Adds magic methods and is hard to debug.
```python
from django.conf.urls import url

from . import views

urlpatterns = [
    url(r'^$', views.index, name='index'),
]
```

##forms.py
* On creating User Authentication forms, its better to check first `django.contrib.auth.forms`.

##views.py
* use **transanctions** whenever creating, updating, or deleting something from the database. This maintains database integrity.
```python
from django.db import transaction

with transaction.atomic():
    flavor.status = status
    flavor.latest_status = timezone.now()
    flavor.save()
```
* On create an object and saving it. Use the create method of the object manager instead.
```python
'''Standard way'''

e = Entry(**kwargs)
e.save
```
```python
'''Better way'''

Entry.objects.create(**kwargs)
```

##Queries

Sample Scenario:
```python
class Blog(models.Model):
    title = models.CharField(max_length=128)
    
class Entry(models.Model):
    blog = models.ForeignKey(Blog)
```

* On many-to-one relationships, query like its an attribute to get the object:
```python
e = Entry.objects.get(id=2)
e.blog = some_blog
e.save()
```
* On one-to-many relationships, you query the `many` Manager with the name `entry_set`.
* `_set` will be overriden if a `related_name` was used on by the dependent model.
```python
b = Blog.objects.get(id=1)
b.entry_set.all() # Returns all Entry objects related to Blog.
b.entry_set is a Manager that returns QuerySets.
b.entry_set.count()
```
* On many-to-many relationships, the model that defines the `ManytoManyField` uses attribute name to get the Manager but the reverse uses model name + `_set`.
```python
e = Entry.objects.get(id=3)
e.authors.all() # Returns all Author objects for this Entry.
e.authors.count()

a = Author.objects.get(id=5)
a.entry_set.all() # Returns all Entry objects for this Author.
```
* On one-to-one relationships, the model that defines the OneToOneField uses attribute_name to get the object but the other model will use the model name to get an object (on a Manager like the previous examples).
```python
ed = EntryDetail.objects.get(id=2)
ed.entry # Returns the related Entry object.

e = Entry.objects.get(id=2)
e.entrydetail # returns the related EntryDetail object
```
* Use `select_related` lookup to lessen database queries. `select_related` only works for many-to-one and one-to-one relationships.
```python
'''Standard Lookup'''

# Hits the database.
e = Entry.objects.get(id=5)

# Hits the database again to get the related Blog object.
b = e.blog
```
```python
'''Lookup with select_related'''

# Hits the database.
e = Entry.objects.select_related('blog').get(id=5)

# Doesn't hit the database, because e.blog has been prepopulated
# in the previous query.
b = e.blog
```
* For many-to-many and one-to-many relationships, use `prefetch_related` related instead.
```python
Pizza.objects.all().prefetch_related('toppings')
```
Any subsequent chained methods which imply a different database query will ignore previously cached results, and retrieve data using a fresh database query. As such, dont do this:
```python
>>> pizzas = Pizza.objects.prefetch_related('toppings')
>>> [list(pizza.toppings.filter(spicy=True)) for pizza in pizzas]
```
##Source
[http://www.twoscoopspress.com/](http://www.twoscoopspress.com/)<br/>
[https://docs.djangoproject.com/en/1.9/](https://docs.djangoproject.com/en/1.9/)<br/>
[https://medium.com/@raiderrobert](https://medium.com/@raiderrobert)
