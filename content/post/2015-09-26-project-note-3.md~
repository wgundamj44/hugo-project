---
layout: post
title: Some project notes(3)
category: [tech]
tags: [Django]
---
## Transaction in Django

### atomic
{% highlight python %}
@atomic
def transaction_func():
    pass
{% endhighlight %}

We know that a decorator will return a wrapper function/class of the decorated function. In Django, the an instance of Atomic class is returned. Atomic is a subclass of ContextDecorator, it is callable because ContextDecorator has a `__call__` function.

In ContextDecorator's `__call__`, we can see something like:
{% highlight python %}
@wraps(func, assigned=available_attrs(func))
def inner(*args, **kwargs):
    with self:
        return func(*args, **kwargs)
return inner
{% endhighlight %}
So, the wrapped function will be put into a context `self`. In its context control function,
`__enter__` will open a new transaction if not already in one or create a new savepoint if otherwise. `__exit__` is rather complex. Generally, if atomic is wrapped in another atomic, then the inner one will be treated as a savepoint if not be told otherwise. when the inner one exits, Django will try to commit the inner savepoint, and rollback the savepoint and set `needs_rollback` flag if error occurs. If everything goes well, then in the outermost atomic exits, Django will commit the transaction.

### database connections
In Django settings, we wrote DATABASES like:
{% highlight python %}
DATABASES = {
    'default': {
        'ENGINE': 'Django.db.backends.mysql', # Add 'postgresql_psycopg2', 'mysql', 'sqlite3' or 'oracle'.
        'NAME': '',                      # Or path to database file if using sqlite3.
        # The following settings are not used with sqlite3:
        'USER': '',
        'PASSWORD': '',
        'HOST': '',                      # Empty for localhost through domain sockets or '127.0.0.1' for localhost through TCP.
        'PORT': '',                      # Set to empty string for default.
    }
}
{% endhighlight %}
So how this info is used? Django has a class `ConnectionHandler`. This class holds the `DATABSES` in its `_database` dict. Also, it has a `_connection` dict, which stores the connections to databases. For example, for our DATABASES above, `_connection` will have a `default` attribute, with its value the connection instance which talks to the database. All the db manipulations, like transaction, rollback etc. will be carried out through this connection instance.

## Behind model class of Django
{% highlight python %}
class Question(models.Model):
    question_text = models.CharField(max_length=200)
    pub_date = models.DateTimeField('date published')
{% endhighlight %}
When we use the above code to create a model in Django, we get a class `Question`. If we try to invoke `Question.pub_date`, we get an attribute error saying that pub_date isn't there.
Instead, in `Question._meta.fields` we find these fields. So, models.Model does more than it appears to do.

In the definition of models.Model, I find it has a base class `six.with_metaclass(ModelBase)`. Here its base class is not the simple ModelBase, `with_metaclass` made a dummy class like this:
{% highlight python %}
def with_metaclass(meta, *bases):
    class metaclass(meta):
        def __new__(cls, name, this_bases, d):
            return meta(name, bases, d)
    return type.__new__(metaclass, 'temporary_class', (), {})
{% endhighlight %}
This is a python2/3 compatible way of setting metaclass.
About metaclass, there's an excellent article in [stackoverflow](http://stackoverflow.com/questions/100003/what-is-a-metaclass-in-python).

So how this with_metaclass works? \\
When we write `six.with_metaclass(ModelBase)`, we create a class whose __class__ is `metaclass`. Now `Model` will have a base class whose __class__ will be `metaclass`. According to [python doc](https://docs.python.org/2/reference/datamodel.html#__metaclass__):

> if there is at least one base class, its metaclass is used
> (this looks for a __class__ attribute first and if not found, uses its type)

`Model`'s metaclass will be `metaclass`, so will all its sub classes. As a result, when `Question` is defined, `__new__` of `metaclass` will be called, and it returns an instance of `ModelBase`, this instance becomes the content of `Question` class we saw.

How does `ModelBase` create the content of `Question`? \\
First thing I can see is that, it attach `_meta` object to model class. _meta is an object of class `Options`. This `Options` holds many information about model, including primary key, app_label, fields, etc. Attributes in `Meta` Class of Model will be stored here. This can answer our problem of where has the fields gone? They were added to _meta in the `__new__` of `ModelBase`. Interestingly, fields is actually a method in `Options`, when called return a list on the fly.

Second, install `Manager` of model. This is VERY complex, the behavior depends heavily on if the model is abstract, proxy or its parents. Anyway, at least `ensure_default_manager` will be called, so that we can call `Question.objects`.

Third, add `DoesNotExist` and `MultipleObjectsReturned` exception for model.

## The unusual part of Manager
We often use something like `Question.objects.filter()`, actually this method comes from class `QuerySet`. The definition of `Manager` is:
{% highlight python %}
class Manager(BaseManager.from_queryset(QuerySet)):
    pass
{% endhighlight %}
`from_queryset` is a classmethod, it will 'copy' necessary methods from QuerySet into BaseManager. This 'copy' is a delegation, when we call `filter` on `Manager`, what happens is that the Manager instance will call the same function on the `queryset` obtained by calling `get_queryset`. That's why when customizing Managers, we often override the `get_queryset` method, so that any following query functions will all carried out based on the customized `queryset`.

## Fields in Django ORM


## How query is generated in Django ORM
