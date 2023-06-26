---
title: "Python Nose Tests Runner Speedup"
date: 2013-05-28T09:55:59+02:00
draft: false
---

# Python Nose: Speed Up the Runner

As your test base expands, the time spent running tests inevitably increases. However, the Python testing framework, Nose, provides mechanisms to manage your run plan effectively and speed up the runtime. This article, a continuation of the "Starting with Nose" and "Nose: Extending and customizing" series, will delve into these mechanisms.

## Test Attributes

The use of attributes can be handy to split and accelerate tests. The `attr` decorator from `nose.plugins.attrib` can be used on any testable function. Below is an example that declares the `speed` attribute.

```python
from nose.plugins.attrib import attr

@attr(speed='slow')
def test_slow():  
    # slow test
```
Once you've marked all your slow tests, you can filter this attribute, and Nose will only run the test cases flagged as slow.

```bash
$ nosetest -x -v -a speed=slow
```

If you want to specify multiple attribute filters, you can use a comma-separated list.

**Reference:** [Nose attributes](https://nose.readthedocs.io/en/latest/plugins/attrib.html)

## Parallel Testing

Nose offers the `nose.plugins.multiprocess` plugin, enabling you to run tests concurrently. Please ensure that your test suite is ready to run in parallel and that your tests don't depend on re-entrant context variables or globally allocated resources. Otherwise, your tests may produce unexpected behaviors.

If you have a setup method that can be executed separately for each test, you can use the `_multiprocess_can_split_ = True` option.

```python
class TestClass:  
    _multiprocess_can_split_ = True

    @classmethod
    def setup_class(cls):
        # ...
```

This means that the fixtures will execute multiple times, typically once per test, and concurrently.

If you have a setup method that must be executed once and can be shared among other processes, you can use the `_multiprocess_shared_ = True` option.

```python
class TestClass:  
    _multiprocess_shared_ = True

    @classmethod
    def setup_class(cls):
        # ...
```

You can then run this suite from the command line as follows:

```bash
$ nosetests --processes=NUMBER_OF_PROCESSORS
```

This will create separate processes for each test suite, depending on the sharing type you selected. Each test case will have a context for fixtures and shared resources.

**Reference:** [Nose in parallel](https://nose.readthedocs.io/en/latest/plugins/multiprocess.html)

## Problems with Parallel Running

The primary issue with parallel test suites is related to non-self-contained pure unit tests that depend on external context resources. These dependencies can lead to race conditions and other unexpected behaviors.

Remember:

> "But the biggest issue you will face is probably concurrency. Unless you have kept your tests as religiously pure unit tests, with no side-effects, no ordering issues, and no external dependencies, chances are you will experience odd, intermittent and unexplainable failures and errors when using this plugin. This doesn't necessarily mean the plugin is broken; it may mean that your test suite is not safe for concurrency."

Also, because tests are split among processes, the results are not natively merged for coverage and reporting.

To merge the coverage reports on parallel tests, you can use the `nose-cov` plugin:

```bash
$ nosetests --with-cov --processes=4 tests/
$ coverage combine
$ coverage report
```

For XUnit reports, the problem is the same. Some developers have started working on a multiprocess-compatible XUnit plugin, and there's a beta plugin called `n

ose_xunitmp`. You can use this plugin as follows:

```bash
$ nosetests --with-xunitmp
$ nosetests --xunitmp-file results.xml
```

## Next Steps

This article concludes our series on nosetests. After reading these articles, you should be able to write a simple test suite, add coverage and unit reporting capabilities, create your own filters and plugins, add attributes to filter for, and run them in parallel! It's now your turn to start exploring the Python unit testing universe.

Good luck and have fun!

