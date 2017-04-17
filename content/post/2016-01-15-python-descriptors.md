---
title: "python descriptors"
categories: ["tech"]
date: 2016-01-15
tags: ["python", "Django"]
---

In Django's `View` class, it has a `as_view` method which returns a request handler function. Interestingly this method is decorated with
a `classonlymethod` decorator, making it only can be invoked from class.

The definition as `classonlymethod` is like:
```python
class classonlymethod(classmethod):
    def __get__(self, instance, owner):
        if instance is not None:
            raise AttributeError("This method is available only on the class, not on instances.")
       return super(classonlymethod, self).__get__(instance, owner)
```
This is a class decorator, which convert the decorated method into a new object with `__get__` method. Everytime when this object is accessed,
`__get__` will be invoked and check if caller is a class.

The mechanism of `__get__` get called everytime is python's descriptor.

## What makes a descriptor
```python
__get__(self, object, type=None)
__set__(self, obj, value)
__delete__(self, obj)
```
An object defines any of the above methods makes it a descriptor.

An object defineds both `__get__` and `__set__` is a data descriptor. An object defines only `__get__` is non-data descriptor;
Data descriptor takes precedence over an instance attribue with the same name, while non-data descriptor has less precedence than instance attribute.

### How descriptor is accessed
For an object `a`, its descriptor `x` is called with the order: `object.__getattribute__(x)` --> `type(b).__dict__['x'].__get__(b, type(b))`.
For a class `A`, `x` is called with `type.__getattribute__(x)` --> `A.__dict__['x'].__get__(None, A)`.

So accessing `x` is actually accessing `__get__` of `x`. This explains why `classonlymethod` descriptor works: every time the descripted method is called, the result of `__get__` is returned.


### Application
In python, every function is a non-data descriptor. In Django, there's a `method_decorator` that makes use of this.
Here's the simplied code:
```python
def method_decorator(decorator):
    """
    Converts a function decorator into a method decorator
    """
    def _dec(func):
        def _wrapper(self, *args, **kwargs):
            @decorator
            def bound_func(*args2, **kwargs2):
                return func.__get__(self, type(self))(*args2, **kwargs2)
            return bound_func(*args, **kwargs)
        
        return _wrapper
    return _dec
```
Here `func` is the decorated method, the code simulates how instance access its method by invoking func with `__get__`. This way, `self` will be bind to the first parameter of `func`, and `func` will becomes a "bound method".

The relatioship between function and ubound/bound function can be illustrated as:
```python
def test(*args):
    pass
class A(object):
    pass
a = A()
test #<function __main__.test>
test.__get__(a, A) #<bound method A.test of <__main__.A object at 0x7fd1a82638d0>>
test.__get__(None, A)  #<unbound method A.test>
```
