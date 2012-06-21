---
layout: post
title: "Use Python decorator to curry functions"
date: 2010-10-22 20:06
comments: true
categories: python, functional programming
---

It's been a while since the last time I wrote about Python. This morning, I was listening to a podcast on my way to work. They were discussing functional programming and dynamic languages...I learned Python before I went into Computer Science, and then I learned about functional programming and through learning of Scala and Clojure, my functional programming concepts have been enriched. As I was listening, it suddenly appeared to me that there isn't a way in Python to [curry](http://en.wikipedia.org/wiki/Currying) a function. Not that it's critical to everyday development, but wouldn't it be neat if I can curry a function in Python?

Then the hosts of the podcast discussed how dynamic languages are so flexible that you can pretty much do anything to it. "You can take a function as parameter, return a function from a function, and so on." Hey, isn't that what Python's [decorator](http://www.artima.com/weblogs/viewpost.jsp?thread=240808) can do? I learned decorators before, but I haven't used it beyond the scope of creating properties and certainly haven't written any decorators. I thought this would be a good exercise for learning decorators.

Here's a simple example of what function currying: suppose you have a method

```python
def add(x,y):
  return x+y
```

Then calling `add(1,2)` should be the same as `add(1)(2)`. `add(1)` is what they call a partially applied function. It's a function that takes one parameter.

Our goal here is to write a decorator "curried" that takes a function with n parameters and transform it in a way that can be applied n times and get the final result.

We'll start with unit tests first:

```python
import unittest

class CurryTest(unittest.TestCase):

    def test_with_no_args(self):
        @curried
        def do_nothing():
            return ""
        self.assertEquals("", do_nothing())

    def test_with_int_args(self):
        @curried
        def add_int(x,y):
            return x+y
        self.assertEquals(3, add_int(1)(2))
    def test_with_str_args(self):
        @curried
        def add_str(x,y):
            return "%s%s"%(x,y)
        self.assertEquals("ab", add_str("a")("b"))
```

So we make sure that a currying on a function takes no parameter is valid but should be a pass through, and also the "curried" decorator can be applied to any function with arguments (excluding positional arguments and keyword arguments)

A decorator is simply a function that takes a function as parameter:
```python
def curried(fn):
  pass
```
and `@curried` is simply a syntactic sugar for:

```python
def fn(...): ...
fn=curried(fn)
```

So, now we can write `curried` decorator. 
To make the test for function with no argument pass, in `curried()` function, we can test to see if fn has arguments. Python's standard library provides `inspect.getargspec` method:

```python
def curried(fn):
  argspec = inspect.getargspec(fn)
  if len(argspec.args)==0:
    return fn
  else:
    # later
```

Now the first test passes.

For the other two cases, here's the strategy. In Python, when a class defines `__call__` method, the instance of that class is said to be "callable". For instance:

```python
class A(object):
  def __call__(self, arg):
    return arg

f=A()
f("echo")  # this gives you "echo"
```

This is very similar to Scala's `apply()` function. Now that we have this in our inventory, we can define a `PartialFunction` class, take all the required parameters of the original function, and allow them to be applied one at a time. So the `__call__` method of PartialFunc will look like this:

```python
def __call__(self, value):
  # Xxx
```

If all the required parameters are passed in, `PartialFunc` should evaluate the original function with the complete argument list. Otherwise, `PartialFunc` stores the parameter in an instance variable, and returns itself.

Here's the complete code:
```python
class PartialFunc(object):
    def __init__(self, fn, argspec):
        self.fn = fn
        self.argspec = argspec 
        self.args = []

    def __call__(self, value):
        self.args.append(value) 
        if len(self.args) == len(self.argspec.args):
            arglist = ",".join(["self.args[%d]"%i for i in range(0, len(self.args))])
            return eval("self.fn(" + arglist + ")")
        else:
            return self
```

and the curried decorator:
```python
def curried(fn):
    argspec = inspect.getargspec(fn)
    if len(argspec.args) == 0:
        return fn
    else:
        return PartialFunc(fn, argspec)
```

It's pretty straightforward. When the parameters are complete, I construct a python statement that calls the original function with the complete argument list, and then pass the statement into an eval statement. I know evals are evil, but I can't find a way in Python to dynamically change the signature of the original method and make it accept a variable length argument (varargs).

So this is it. It's quite simple. Python methods can have varargs and keyword args, the situation gets a little more complicated. The thing is, both varargs and keyword args are not mandatory, so it's hard for the curried function to know whether the argument list has been completed...Also, if you take default values into account, it could get even more complicated. 

