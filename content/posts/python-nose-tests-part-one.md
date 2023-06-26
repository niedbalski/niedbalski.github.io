---
title: "Python Nose Tests Part One"
date: 2013-05-26T09:45:33+02:00
draft: false
---

# An Introduction to Python Testing with Nose

This article delves into the rudimentary aspects of executing a Python test suite using Nose. It focuses on practical aspects for quickly developing a test suite and reviewing its results.

Back when I first started writing Python code, the common method for writing and running a series of unit tests was to utilize the standard `Unittest` module for making assertions and developing your custom runner. The runner was tasked with tracking assertions and generating a human-readable report, albeit without standardization. Due to this, many older open-source Python projects still use these types of runners, which are harder to maintain. These runners have evolved over time to generate better reports, add coverage, inspect various file types, and more.

However, the Python testing landscape has significantly transformed in recent years. The standard Python module has grown robust, with numerous assertion cases being added and a vast collection of utilities continuously expanding.

During this time, Nose has emerged as the de-facto standard for running unit tests. It includes an extensive base library for assertions/utils to extend your unit test classes, and report generation has now become standardized, with Xunit reigning supreme in output formats.

I typically run my tests on a pre-commit basis. This way, before pushing a change into the repo, my test base runs to check if any modifications might break existing code. Now, I generally use a development branch, and before migrating changes to the master branch, I utilize a Jenkins or Travis server to continually run my test suites and verify their success.

## Utilizing Nose

As I mentioned earlier, `nosetests` is the default standard in every project I work on. Here, I'll share my standard configuration and the methods of integrating these components.

Here's a simple example of a testable class:

```python
class Math:

    def sum(self, a, b):  
        return a + b

    def subs(self, a, b):
        return a - b

    def mul(self, a, b):  
        return a * b
```

In every test suite I've written, I've used the `nose.tools` collection because it provides a set of convenience functions that make writing tests easier and does not depend on the `unittest` module.

Here's a basic test suite for the above class:

```python
import code  
import operator

m = code.Math()

def test_sum():  
    """Check if sum method is equivalent to operator"""
    eq_(m.sum(1, 1), operator.add(1, 1))


def test_sub():  
    """Check if sub method is equivalent to operator"""
    eq_(m.sub(2, 1), operator.sub(2, 1))


def test_mul():  
    """Check if mul method is equivalent to operator"""
    eq_(m.mul(1, 1), operator.mul(1, 1))
```

To run this test suite using Nose, execute the `nosetests` command from the command line. My usual command is:

```bash
$ nosetests --verbosity=3 -x . 
```

This command provides verbose output and will halt the test runner if any test fails. You can review other command-line options [here](https://nose.readthedocs.io/en/latest/man.html).

In the example case, Nose's output will be something like:

```bash
Check if sum method is equivalent to operator ... ok  
Check if sub method is equivalent to operator ... ok  
Check if mul method is equivalent to operator ... ok

----------------------------------------------------------------------
Ran 3 tests in 0.001s

OK  
```

Two crucial aspects of running Nose over any other runner are the output format and coverage options. By default, you can specify the `--with-coverage

` option, and Nose will output the missing and covered statements in your code.

To produce the standard XUnit output file format (for compatibility with CI environments like Jenkins or Travis), pass the following command:

```bash
$ nosetests --verbosity=3 -x --with-xunit 
```

This command generates a `nosetests.xml` file in the current working directory.

To produce a human-readable report from this XUnit output, I've developed an XSLT template with filters and other useful features, available on [Github Nosetests-XUnit-XSLT](https://github.com/niedbalski/nosetest-xunit-xslt).

To format the report, run the following commands:

```bash
$ sudo apt-get install xlstproc
$ xsltproc nosetests.xslt nosetests.xml > tests.html
```

By default, `nosetests` tries to load a configuration file (`nose.cfg`) from the current working directory. Here's my default configuration file:

```ini
[nosetests]
verbosity=3  
with-xunit=1  
xunit-file=xunit.xml  
with-xcoverage=1  
xcoverage-file=coverage.xml  
```

To integrate `nosetests` into a Jenkins setup, you will need to add support for XUnit, which can be achieved using the [Jenkins XUnit plugin](https://plugins.jenkins.io/xunit/).

## Auto-generating tests

Another great feature of Nose is the test generator support. Test generators let you enhance your test base by yielding test cases with different parameters, thus increasing both the number and quality of tests.

Here's a good example for our basic class using generators:

```python
def tests_sum():  
    """Generate multiple tests for sum method"""
    for i in range(1, 10):
        yield check_sum, i, i ** 2

def check_sum(n, nn):  
    """Check if sum method is equivalent to operator"""
    eq_(m.sum(n, nn), operator.add(n, nn))
```

This generates ten new test cases, contributing to a more robust code base.

In this article, we've covered some basic aspects of the standard Python test suite. My hope is that you'll dive deeper into Nose and start writing your own enhancements.

In the next article in this series, I will explain how to extend Nose to create your own test runner, how to run tests in parallel to accelerate the process, and also how to write useful Nose plugins to take full advantage of this fantastic Python testing tool. Stay tuned!
