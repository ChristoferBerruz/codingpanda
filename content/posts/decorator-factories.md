---
title: "Decorator Factories"
date: 2022-01-12T12:40:42-08:00
draft: false
tags: [decorators, python3.6+]
---

Previously we explained the use of Decorators in Python. Today, I want to explore the concept of decorator factories and how they can be of great use when we want to **dynamically** modify the behavior of the decorator.

If you need a refresher on decorators, read my [previous post]({{< ref "posts/decorators-explained">}}).

## Decorator factories

The concept is simple: a decorator that returns a decorator. To make the examples simpler, we will always assume that the decorator will work on functions.

A simple code skeleton of a decorator factory is the following:

```python
import functools


def decorator_factory():

    def decorator(func):
        @functools.wraps(func)
        def wrapped(*args, **kwargs):
            return func(*args, **kwargs)
        return wrapped

    return decorator
```

To use a decorator factory, we actually have to make a *function call* `()` on the factory.

So, our functions will be decorated as follows:

```python

@decorator_factory()
def some_function():
    pass
```

Instead of

```python
@decorator_factory
def some_function():
    pass
```

## When do we use decorator factories?

The main reason to use decorator factories is when we want to modify the behavior of the inner decorator being returned.

For example, imagine to implement a `@sleep` decorator that pauses after the decorated function executes. Now, how will you go about allowing the user to specify **how long** to pause? That's where a factory comes in handy.

## Implementing the base `@sleep` decorator

Let's first develop a plain decorator and hard code the value of how long we should sleep. In this case, the value is 10 seconds.

```python
import time
import functools

def sleep(func):
    """
    Sleeps for 10 seconds after execution of func
    """
    @functools.wraps(func)
    def wrapped(*args, **kwargs):
        result = func(*args, **kwargs)
        time.sleep(10)
        return result

    return wrapped
```

We can use our decorator as we usually use decorators:

```python

@sleep
def some_func():
    pass
```

## Using a factory to change behavior of sleep

Let's make `sleep` a **factory** that takes the time the function should sleep as a keyword argument.

```python
import time
import functools


def sleep(sleep_time=10):
    """
    Returns a sleep decorator with given sleep_time.
    """
    def sleep_decorator(func):
        """
        Returns a wrapped version of func that sleeps for sleep_time after
        calling func.
        """
        @functools.wraps(func)
        def wrapped(*args, **kwargs):
            result = func(*args, **kwargs)
            time.sleep(sleep_time)
            return result

        return wrapped

    return sleep_decorator
```

We see that the `sleep_decorator` now uses the `sleep_time` value to know how much to sleep, without us hard coding that value anymore.

Given that `sleep` is now a decorator factory, we have to make a function call on it.

```python

@sleep(sleep_time=20)
def foo():
    pass

@sleep(sleep_time=30)
def bar():
    pass

@sleep()
def baz():
    pass
```

To show that we indeed sleep a different amount of time in each function, I will use the `time_it_decorator` we [previously]({{< ref "posts/decorators-explained">}}) developed to *time* execution time.

```python
import time
import functools


def sleep(sleep_time=10):
    """
    Returns a sleep decorator with given sleep_time.
    """
    def sleep_decorator(func):
        """
        Returns a wrapped version of func that sleeps for sleep_time after
        calling func.
        """
        @functools.wraps(func)
        def wrapped(*args, **kwargs):
            result = func(*args, **kwargs)
            time.sleep(sleep_time)
            return result

        return wrapped

    return sleep_decorator


def time_it_decorator(func):
    """
    Returns a wrapped function that times the execution time of func.
    """
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


@time_it_decorator
@sleep(sleep_time=20)
def foo():
    pass


@time_it_decorator
@sleep(sleep_time=30)
def bar():
    pass


@time_it_decorator
@sleep()
def baz():
    pass


foo()
bar()
baz()
```

Once we run this script, we see that each of the functions sleep for a different amount of time.

```
'foo' took 20.0204 seconds to complete
'bar' took 30.0305 seconds to complete
'baz' took 10.0104 seconds to complete
```

## Conclusion

* Use decorator factories when you want to give the user of your decorator the ability to modify the behavior of the decorator. Use arguments and keyword arguments to achieve that.
* To use a decorator factory in your functions, you have to make a function call `()` on them. So, if the function named `decorator` is a factory you should use `@decorator()` instead of `@decorator`.

