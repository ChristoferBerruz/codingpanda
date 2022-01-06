---
title: "Decorator Overloading - JS async/await in Python"
date: 2021-10-24T20:58:12-07:00
draft: true
---

Recently, I have decided to dive deeper into the decorator syntax present in Python.

First, let's do a quick recap.

## Recap - What is a decorator?

Decorators can be divided in two parts: the decorating function and the decorator syntax.

The decorating function is a **function that take another function as an argument an return an object**. Generally, that object being
returned is also a function.

Based on our definition, this is a good (although rather boring) candidate of a decorating function.

```python
def decorating_function(some_function):
    return some_function
```

`decorating_function` simply returns back the function that you pass as an argument! This is indeed the simplest decorating function: an identity decorator.

Now, let's look at the **decorator syntax**. In python, the decorator syntax will be to **wrap** another function will these decorating function (or decorator).

For example, imagine you have the plain function:

```python
def say_hello():
    print('Say hello')
```

And you now want to decorate it with `decorating_function`. The syntax will just be
```python
@decorating_function
def say_hello():
    print('Say hello')
```

This is just syntatic sugar for the following:
```python
def say_hello():
    print('Say hello')

say_hello = decorating_function(say_hello)
```

Moreover, this happens during **interpretation time**! Not during run-time.

## Taking it really far - then/catch JS syntax using decorator

Here is a crazy idea: what if we can have some JavaScript async syntax, like then/catch
in python? Well, let's do that and along the way my goal is for us to understand the power of
decorators.

### The idea
JavaScript has promises, which are objects that encapsulate asynchronous tasks that will run in the future.
I really like the name: I promise I will do this; not now, but I will do it.

An important aspect of a  `promise` is that we can attach a `then` and `catch` handler.

Finally, I would like to use decorator to make promises on the fly, without knowing much
of the implementation.

### The Promise class

First, let's define the simplest Promise: no then and catch handler. We will save all these in a module
named `promises.py`

If we think about it; the main object of a promise is the task that we will eventually execute. Hence, a really
simple definition can be:

```python
from typing import Callable
class Promise:

    def __init__(self, task: Callable):
        self._task = task
```

Now, we want to use decorators, so let's create a decorator that can make a function a `Promise`

```python
def promise(task: Callable) -> Promise:
    return Promise(task)
```

Then, in another file name `main.py` I can do the following
```python
from promises import promise


@promise
def say_hello(name: str):
    print(f'Hi there, {name}')
```

**HOLD ON!** The `@promise` decorator does not return a function... It returns a `Promise` object!

Yes, that is correct. Here, I am using the `decorator syntax` to make my life easier and make promises
on the fly. Remember that the object a decorator returns is not always a function, it can be something
else. Given that we are discussing this already, let's make a `Promise` behave like a function :)

In python, the `__call__` is a special method that allows to make any object a `Callable`; something
that we can run by using function-call syntax `()`

```python
from typing import Callable
class Promise:

    def __init__(self, task: Callable):
        self._task = task

    def __call__(self, *args, **kwargs):
        return self._task(*args, **kwargs)


def promise(task: Callable) -> Promise:
    return Promise(task)
```

Let's try to run our `Promise`-d task!

```python
from promises import promise

@promise
def say_hello(name: str):
    print(f'Hi there, {name}')


def main():
    say_hello('person')

if __name__=='__main__':
    main()
```

We get the following:
```shell
python main.py 
Hi there, person
```

### Adding then/catch

Let's get spicy. Now that we have a base class, let's add a mechanism to add a `then` and `catch` function handlers.

The easiest part is to realize that these two function will be members of the `Promise` class. We also know they are functions,
so let's annotate their types.

```python
from typing import Callable
class Promise:

    def __init__(self, task: Callable, then: Callable = None, catch: Callable = None):
        self._task = task
        self._then = then
        self._catch = catch

    def __call__(self, *args, **kwargs):
        return self._task(*args, **kwargs)
```

If we take a step back and think about what will happen in the JavaScript world...

If the promise returns successfully, we execute the `then` function. If there is an error, we execute
the `catch` part.

So, let's modify the way our `__call__` method works.

```python
from typing import Callable
class Promise:

    def __init__(self, task: Callable, then: Callable = None, catch: Callable = None):
        self._task = task
        self._then = then
        self._catch = catch

    def __call__(self, *args, **kwargs):
        result = None
        try:
            result = self._task(*args, **kwargs)
        except Exception as e:
            self._catch(e)
            return

        self._then(result)
```

Before we move forward, let's just add some property fields for the class. Given that we are using
decorators in this lesson, we will use the `@property` decorator.

> On a side note, the `@property` decorator is quite neat. It returns a Property object,
> to which a `setter` or `deleter` can be registered using decorators as well!

```python
from typing import Callable
class Promise:

    def __init__(self, task: Callable, then: Callable = None, catch: Callable = None):
        self._task = task
        self._then = then
        self._catch = catch

    @property
    def task(self):
        return self._task

    @task.setter
    def task(self, some_task: Callable):
        self._task = some_task

    @property
    def then(self):
        return self._then

    @then.setter
    def then(self, some_then: Callable):
        self._then = some_then

    @property
    def catch(self):
        return self._catch

    @catch.setter
    def catch(self, some_catch: Callable):
        self._catch = some_catch

    def __call__(self, *args, **kwargs):
        result = None
        try:
            result = self._task(*args, **kwargs)
        except Exception as e:
            self._catch(e)
            return

        self._then(result)
```

Perfect! This is looking good. Now, let's provide two functions that will be used as the
decorators: `@then(<some_function>)` and `@catch(<some_function>)`. We need to pass
a function to these decorators because we want to register the functions as handlers.


```python
def then(some_then: Callable) -> Callable:

    def decorating_function(promise: Promise) -> Promise:
        promise.then = some_then
        return promise

    return decorating_function
```

Alright, this decorator is definitely more complicated than what we are used to. Let's walk step by
step to understand it better.

Let's recap. The decorator syntax is:

```python
@decorating_func
def decorate_me():
    pass
```

We can think about it in two separate entities: `{}` and `[]` as such:
```python
@{decorating_func}
[
def decorate_me():
    pass
]
```

By looking it at it in this way, we can realize that `[]` is the object we want to decorate; usually a function or `Callable`,
but it can be any object (like a `Promise` for example).

And, we also see that `{}` **eventually** has to evaluate to a function that takes whatever object is inside`[]`

If `then` is a function therefore `then()` is a **function call**. Hence, the function call can **return another function**! Which it
does by taking a look at the type hints. 

This function returned by `then()` *MUST* take as a parameter whatever it is trying to decorate. And what are we trying to decorate?
A promise! `then()` eventually returns `decorating_function` and if we take a look, it takes a `Promise` as an argument!

`decorating_function` returns the promise it decorates back for us to be able to append a `@catch()` following the `@then()`.

Similarly then, the `@catch(<some_functiom>)` is simply:

```python
def catch(some_catch: Callable) -> Callable:

    def decorating_function(promise: Promise) -> Promise:
        promise.catch = some_catch
        return promise

    return decorating_function
```

Now, our `promises.py` looks like this:

```python
from typing import Callable
class Promise:

    def __init__(self, task: Callable, then: Callable = None, catch: Callable = None):
        self._task = task
        self._then = then
        self._catch = catch

    @property
    def task(self):
        return self._task

    @task.setter
    def task(self, some_task: Callable):
        self._task = some_task

    @property
    def then(self):
        return self._then

    @then.setter
    def then(self, some_then: Callable):
        self._then = some_then

    @property
    def catch(self):
        return self._catch

    @catch.setter
    def catch(self, some_catch: Callable):
        self._catch = some_catch

    def __call__(self, *args, **kwargs):
        result = None
        try:
            result = self._task(*args, **kwargs)
        except Exception as e:
            self._catch(e)
            return

        self._then(result)


def promise(task: Callable) -> Promise:
    return Promise(task)


def then(some_then: Callable) -> Callable:

    def decorating_function(promise: Promise) -> Promise:
        promise.then = some_then
        return promise

    return decorating_function


def catch(some_catch: Callable) -> Callable:

    def decorating_function(promise: Promise) -> Promise:
        promise.catch = some_catch
        return promise

    return decorating_function
```

Looking at our `main.py` we can now use our new promise-style function!

```python
from promises import promise, then, catch


def handle_greeting(greeting: str):
    print(f"I received your greeting: {greeting}")


def handle_greeting_error(exc):
    print(f"There was a problem with your greeting. Error: {exc}")


@catch(handle_greeting_error)
@then(handle_greeting)
@promise
def say_hello(name: str) -> str:
    print(f'Hi there, {name}')

    greeting = 'Good morning'
    return greeting


def main():
    say_hello('person')

if __name__=='__main__':
    main()
```

Now, if we ran this we will see

```shell
python main.py
Hi there, person
I received your greeting: Good morning
```

We can also see that if we raise an exception while generating the greeting as such

```python
def say_hello(name: str) -> str:
    print(f'Hi there, {name}')

    greeting = 'Good morning'
    raise Exception('I am not in the mood to give greetings')
    return greeting
```

And running it again

```shell
python main.py
Hi there, person
There was a problem with your greeting. Error: I am not in the mood to give greetings
```