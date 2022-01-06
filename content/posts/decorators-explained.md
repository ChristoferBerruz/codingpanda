---
title: "Decorators - Explained"
date: 2022-01-01T20:40:16-08:00
draft: false
tags: [decorators, python3.6+]
---

**Decorators** are widely used in Python. These decorators are not be confused with the [Decorator Design Pattern](https://en.wikipedia.org/wiki/Decorator_pattern).

Decorators have one purpose: add extra functionality to the object they decorate. In Python, the objects that are commonly decorated are **functions**.

## Decorators

Using decorators consists of 3 things.
1. A decorating function, usually refered to as decorator.
2. An object to be decorated.
3. The use of the `@` symbol.

> In Python, `callable objects` can be treated as functions. If you want to make a class callable, you will have to implement the [`__call__`](https://docs.python.org/3/reference/datamodel.html#object.__call__) method.

The decorator **syntax** is simple, yet really powerful and popular. As someone who works in developing testing infrastructure using [Pytest](https://docs.pytest.org/en/6.2.x/contents.html), decorators are used all over the place.

Let's take Pytest's well known `fixture` decorator.

```python
import pytest


@pytest.fixture
def some_fixure():
    pass
```

Here, the use of decorators is

```python
@pytest.fixture
def some_fixure():
    pass
```

At this point, I bet you have seen this syntax somewhere in python code! Here, `some_fixture` is the object to be decorated, while `pytest.fixture` is the decorating function or **decorator**.

What sometimes is hard to remember, and you might not know, is that the decorator syntax is basically syntactic sugar for the following:
```python
def some_fixture():
    pass

some_fixture = pytest.fixture(some_fixture)
```


## Designing your own decorator - a time logger.

Suppose we want to record how long a function takes to execute and print to console. However, we are not allow to change the function body itself. How do we go about? Well, decorators!

Here is a general rule of thumb when designing your decorators.
1. What's the input of the decorator?
2. What's the output of the decorator?
3. What does the decorator do on top of what the original input does?

 > Step 2) can be become more complex once a decorator can return decorators. That is call a decorator factory!

Let's answer our questions:
1. **What's the input of the decorator?** A function, because it decorates functions.
2. **What's the output of the decorator?** A function.
3. **What does the decorator do on top of what the original object does?** It should calculate the time a function takes to execute and print to console.

With the requirements in mind, let's do this.

## Implementing the time logger

By 1) we know that the decorator should take functions.

```python
def time_it_decorator(func):
    pass
```

By 2), we know that the decorator should also return a function.

```python
def time_it_decorator(func):

    def wrapped():
        pass

    return wrapped
```

> We usually name the function returned by the decorator `wrapped`. The reason is that it is a `wrapped` version of the original function.

The `wrapped` function, should accept the same arguments as the original function. Because we don't know which are those, we are going to use the catch-all `*args` and `**kwargs`.

```python
def time_it_decorator(func):

    def wrapped(*args, **kwargs):
        pass

    return wrapped
```

Moreover, the `wrapped` function should do something on top of what the original function does. So, we still have to call the original function.
```python
def time_it_decorator(func):

    def wrapped(*args, **kwargs):
        return func(*args, **kwargs)

    return wrapped
```

> If this were to be our decorator, it will be rather boring as it does nothing **extra** on top of what the original object does. It is still a valid decorator tho.

Now, let's add the extra behavior specified in 3)

```python
import time

def time_it_decorator(func):
    def wrapped(*args, **kwargs):
        start_time = time.perf_counter()
        result = func(*args, **kwargs)
        end_time = time.perf_counter()
        time_taken = (end_time - start_time)
        print(f"{func.__name__!r} took {time_taken:.4f} seconds to complete")

        return result

    return wrapped
```
> Never heard of `time.perf_counter`? You can [check it out!](https://docs.python.org/3/library/time.html#time.perf_counter)

> Writing code in python3 and still using `.format()` in strings? Consider using [f-strings!](https://realpython.com/python-formatted-output/)

Now we can use our decorator!
```python
import time

def time_it_decorator(func):
    def wrapped(*args, **kwargs):
        start_time = time.perf_counter()
        result = func(*args, **kwargs)
        end_time = time.perf_counter()
        time_taken = (end_time - start_time)
        print(f"{func.__name__!r} took {time_taken:.4f} seconds to complete")

        return result

    return wrapped

@time_it_decorator
def connect_to_database():
    print("Connecting to database")
    time.sleep(10)

connect_to_database()
```

If we run this script, we should see something like this
```code
Connecting to database
'connect_to_database' took 10.0103 seconds to complete
```

## One small drawback

Functions have a `__doc__` (docstring information) and `__name__` properties. To see what happens to those properties in decorated functions
let's add some docstrings to both `connect_to_database` and `wrapped` functions.

**Our goal:** see what `connect_to_database.__name__` and `connect_to_database.__doc__` are *before* and *after* applying the decorator.

To do that, let's use the expanded version of the decorator syntax (without using `@`).
```python
import time

def time_it_decorator(func):
    def wrapped(*args, **kwargs):
        """
        Wrapped version of func that calculates the execution time
        of func and prints it to console.
        """
        start_time = time.perf_counter()
        result = func(*args, **kwargs)
        end_time = time.perf_counter()
        time_taken = (end_time - start_time)
        print(f"{func.__name__!r} took {time_taken:.4f} seconds to complete")

        return result

    return wrapped


def connect_to_database():
    """
    Connects to a database in the cloud.
    """
    print("Connecting to database")
    time.sleep(10)

# Before decorator
print("Before decorator")
print(connect_to_database.__name__)
print(connect_to_database.__doc__)

# Apply decorator
connect_to_database = time_it_decorator(connect_to_database)

# After decorator
print("After decorator")
print(connect_to_database.__name__)
print(connect_to_database.__doc__)
```

If we run this script, we see
```code
Before decorator
connect_to_database

    Connects to a database in the cloud.

After decorator
wrapped

        Wrapped version of func that calculates the time execution time of func and
        prints it to console.
```

This behavior is expected. However, it is not desired because the `__name__` and `__doc__` properties of the initial function are no longer accessible.

## Using `wraps` to keep `__name__` and `__doc__`

If you would like to preserve the original function's documentation and name and for them to become properties of the `wrapped` function, we can use the [`functools.wraps`](https://docs.python.org/3/library/functools.html#functools.wraps) decorator.

```python
import functools
import time

def time_it_decorator(func):
    @functools.wraps(func)
    def wrapped(*args, **kwargs):
        """
        Wrapped version of func that calculates the execution time
        of func and prints it to console.
        """
        start_time = time.perf_counter()
        result = func(*args, **kwargs)
        end_time = time.perf_counter()
        time_taken = (end_time - start_time)
        print(f"{func.__name__!r} took {time_taken:.4f} seconds to complete")

        return result

    return wrapped


def connect_to_database():
    """
    Connects to a database in the cloud.
    """
    print("Connecting to database")
    time.sleep(10)

# Before decorator
print("Before decorator")
print(connect_to_database.__name__)
print(connect_to_database.__doc__)

# Apply decorator
connect_to_database = time_it_decorator(connect_to_database)

# After decorator
print("After decorator")
print(connect_to_database.__name__)
print(connect_to_database.__doc__)
```

Now if we run this script
```code
Before decorator
connect_to_database

    Connects to a database in the cloud.

After decorator
connect_to_database

    Connects to a database in the cloud.
```

We kept the original `__name__` and `__doc__` properties! This is really useful to keep the identity of the original function exposed as we apply decorators.

## Conclusion
Let's recap what we learned today:
1. Decorators in Python are not the same as the [Decorator Design Pattern](https://en.wikipedia.org/wiki/Decorator_pattern)
2. There a 3 components of using a decorator:
   - The decorator
   - The object to be decorated
   - The `@` symbol (optional)
3. In designing decorators, remember to define
   - The input of the decorator
   - The output of the decorator
   - The extra work your decorator does on top of the original input
4. If decorating functions, use `functools.wraps` to preserve the `__name__` and `__doc__` properties of the original function


And... that's it! Hope you learned something useful today. Keep up in your journey in becoming an expert Python developer :)