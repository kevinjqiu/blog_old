---
layout: post
title: "Use Python's sys.settrace() for fun and for profit"
date: 2012-04-17 16:58
comments: true
categories: python
---

## The itch to scratch

Everyone in the software industry knows Kent Beck, the pioneers of extreme programming and test-driven development and the co-author of JUnit. One of his lesser known project was [JUnitMax](http://junitmax.com), which aims to reduce the time developers have to wait while tests are running. One of the ideas behind that is that when code changes, only the test cases that exercise the code need to be run, instead of running the entire suite. The idea makes a lot of sense to me, but at the time, I (and the development shop I was in) wasn't practising enough TDD, so unit test time wasn't a big problem for me back then.

Fast-forward a few years, now as the project in my current company gets bigger, the time it takes to run tests is slowly becoming an impeding factor of my productivity. I remembered JUnitMax and say to myself, wouldn't it be neat if something like JUnitMax were available? As the name suggests, JUnitMax is for Java while my project is in Python. Java, being a statically-typed language, has the blessings of statical analysis, which means a tool like JUnitMax can figure out which test cases cover which lines of code simply by type analysis. Python, however, being a dynamic language, doesn't have this ability.

A few days ago, while I was running unit tests with coverage, it dawned on me that if the coverage tool knows which lines of the source code is covered by unit tests, couldn't the same technique be used to figure out which lines are covered by which test cases?

So, I started looking into [coveragepy](http://nedbatchelder.com/code/coverage/)'s source code, and watching its author [Ned Batchelder](http://nedbatchelder.com/blog/)'s excellent PyCon2011 [video](http://blip.tv/pycon-us-videos-2009-2010-2011/pycon-2011-python-aware-python-4896752) on `sys.settrace`. I wanted to build a proof-of-concept tool that integrates with the de-facto Python unit-test tool [nose](https://github.com/nose-devs/nose), that, when run, gathers the information about which lines in the files in the source folder are covered by which test cases, and hence [nostrils](https://github.com/kevinjqiu/nostrils) is born.

## Here comes `sys.settrace()`

Python's motto is "batteries included". This is manifested in many Python's stanndard library modules, such as ast (source code parsing) and dis (bytecode disassembly). One of which is the ability to make the Python interpreter call an external function whenever a line of code is being executed. You can do a lot of fun stuff with it, for example, Coverage.py uses this to build code coverage data; pdb uses it to insert breakpoints into a running application and change the way a Python program is executed.

## How can it be used?

For *nostrils*, we need to write a nose plugin that installs the trace function when a test is encountered. The trace function records the line numbers and the current test case name. After all tests are run, we have our map.

## A simple use case

To start, we need a simple use case:

```python
# worker.py
# this is the code-under-test
def add(x, y):
    z = x + y
    return z

def subtract(x, y):
    z = x - y
    return z
```

```python
# test_worker.py
# test cases

import worker

def test_add():
    assert 1 == worker.add(1, 0)

def test_add___negative():
    assert 0 == worker.add(-1, 1)

def test_subtract():
    assert 0 == worker.subtract(0, 0)

class TestFoo(object):

    def test_add(self):
        assert 5 == worker.add(5, 0)

```

As you can see, we have 4 tests and 2 methods-under-test. Our goal is that when running `nosetests --with-nostrils` (`--with-nostrils` is the switch to turn on the nostrils plugin), we get the following mappings:

```python worker.py
def add(x, y):
  z = x + y # test_add, test_add_negative, TestFoo.test_add
  return z  # test_add, test_add_negative, TestFoo.test_add

def subtract(x, y):
  z = x - y # test_subtract
  return z  # test_subtract
```

## Nose plugin

I won't go into the details about how to create a plugin for nose. You can read it [here](http://readthedocs.org/docs/nose/en/latest/plugins/writing.html, and you can take a look at my sample setup [here](https://raw.github.com/kevinjqiu/nostrils/master/setup.py). In a nutshell, every plugin has a name, and when nose is supplied with --with-*plugin_name*, your plugin is activated. Nose provides a test lifecycle "hooks" that plugins can implement. For example, `startTest` is called when a test case is discovered and adapted into a nose [TestCase](http://readthedocs.org/docs/nose/en/latest/api/test_cases.html). `addSuccess` is called when a test case succeeded. `finalize` is called when all tests are finished.

Here's how my plugin looks like:

```python
class Nostrils(Plugin):
    name = 'nostrils'

    def addError(self, test, err, *args):
        self._restore_tracefn()

    def addFailure(self, test, err, *args):
        self._restore_tracefn()

    def addSkip(self, test, err):
        self._restore_tracefn()

    def addSuccess(self, test, err):
        self._restore_tracefn()

    def startTest(self, test):
        self._current_test = test
        self._install_tracefn()

    def finalize(self, result):
        self._print()

    def _install_tracefn(self):
        self._orig_tracefn = sys.gettrace()
        sys.settrace(self._trace) # See below

    def _restore_tracefn(self):
        sys.settrace(self._orig_tracefn)
```

The idea is that we install the trace function when test starts, and restore the trace function back to what it was. We also keeps track of what's the current test in `self._current_test`.

## Trace function

Now let's have a look at the trace function:

```python
class Nostrils(Plugin):
  # ...
  def _trace(self, frame, event, arg):
    if event == 'line':
      self._trace_down(frame)
    return self._trace

  def _trace_down(self, frame):
    while frame is not None:
      if frame.f_code == test.__call__.func_code:
        break

      self._collect(frame)
      frame = frame.f_back
```

A trace function should take 3 parameters:
* frame: the current [frame](http://docs.python.org/reference/datamodel.html#types) object
* event: what type of event that triggered the trace function? See [here](http://docs.python.org/library/sys.html#sys.settrace)
* `*args`: any additional arguments

Here, I'm only interested in the `line` event, which is triggered when a new line of code is being executed. When this happens, we invoke `_trace_down`, which walks the frame stack by recursing on `frame.f_back`. When it's `None`, we're at the bottom of the stack. Because we're tracing the execution of tests, we can probably stop traversing when the code object of the frame is the entry point of the test case (`if frame.f_code == test.__call__.func_code`). This way, we save ourselves some unnecessary traversals.

## Data Collection

There's are few things we need to collect: filename, line number of the code being executed and the test case name that covers the code.

```python
class Nostrils(Plugin):
  def __init__(self):
    super(Nostrils, self).__init__()
    self._data = defaultdict(
      lambda : defaultdict(
        lambda : set([])
      )
    )

  def _collect(self, frame):
    filename, lineno = frame.f_code.co_filename, frame.f_lineno
    self._data[filename][lineno].add("%s:%s.%s" % self._current_test.address())
```

The data structure we use here is a dictionary of dictionary. At the top level, the keys are filenames, and the values are dictionaries of with keys the line numbers and the values the set of test case names. The data structure looks like this:

```
{
  'foo.py':{
      1 : set(['test_foo.py:test_foo_case1', 'test_foo.py:test_foo_case2']),
      2 : set(['test_foo.py:test_foo_case1', 'test_foo.py:test_foo_case2']),
      3 : set(['test_foo.py:test_foo_case2'])
  }
}
```

There we have it! We have a prototype of what could become a PyUnitMax ;)

## Potential Problems
* Scale: Now I'm only running nostrils on trivial code base. Profiling and optimization is needed if nostrils were to be used in real-world cases.</li>
* Multi-threading: No consideration was given to multi-threading at this stage.</li>

## Collaborators welcome!
I have since refactored the code, revised the data structure and published it on [github](https://github.com/kevinjqiu/nostrils). Please provide me with feedbacks and suggestions.

