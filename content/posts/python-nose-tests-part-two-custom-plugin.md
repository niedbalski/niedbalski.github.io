---
title: "Python Nose Tests Part Two Custom Plugin"
date: 2016-05-27T09:51:03+02:00
draft: false
---
# Writing a Basic Nosetests Plugin

In this article, we continue our exploration of the Python testing framework Nose, building upon the foundations established in the previous article, "Starting with Nose". We'll delve into creating a basic nosetests plugin.

I've applied this approach in a project for RedHat called Autotest, to which I contributed. The project used non-standard nomenclatures and ran its own in-house built-in tests runner. The code for this contribution can be found in this [pull request](https://github.com/autotest/autotest/pull/701/files), and could be useful for customizing your runner. Official reference for this topic can also be reviewed [here](https://nose.readthedocs.io/en/latest/writing_plugins.html).

## Selectors

By default, Nose uses a `nose.selector.Selector` instance to decide what is and is not a test. The default selector is fairly simple: if an object's name matches the `testMatch` regular expression defined in the active `nose.config.Config` instance, the object is selected as a test.

When you run the `nosetests` command, all the files matching the `test_*` filename are marked as potential test candidates. This filter regex parameter is extensible:

```bash
$ nosetests --match='^Foo[\b_\./-])[Tt]est'
```

This pattern is fine for many projects, but may not suffice for projects using non-default filenames or projects where test case levels are split based on the filename. For these cases, you can create custom selectors. Here's a simple usage example:

```python
#!/usr/bin/env python
# -*- coding: utf-8 -*-

from nose.selector import Selector
import logging
import os

logger = logging.getLogger(__name__)

class CustomTestSelector(Selector):

    def wantDirectory(self, dirname):
        parts = dirname.split(os.path.sep)
        return 'tests' in parts

    def wantModule(self, module):
        return True

    def wantFile(self, filename):
        if not filename.endswith('_unittest.py'):
            return False
        return True
```

In this example, we're checking if the file is in the `tests/` directory and if the filename ends with the `_unittest.py` postfix. The Selector API is pretty clear: every time a new filename is received, the directory, module, and filenames are evaluated. If all the functions return `True`, then the filename is added to the test list to be executed.

A selector is pointless without a Plugin, so the next step is to build a basic plugin.

**Reference:** [Nose Selectors](https://nose.readthedocs.io/en/latest/usage.html#selecting-tests)

## Plugins

Plugins extend the default behavior of a Nose runner to support test collection, selection, observation, and reporting.

In this example, we need to extend the default test selection mechanism. The plugin for this selector looks like this:

```python
from nose.plugins import Plugin
import sys
import nose

class CustomSelectorPlugin(Plugin):

    enabled = True
    name = 'custom_selector'

    def configure(self, options, config):
        self.result_stream = sys.stdout

        config.logStream = self.result_stream
        self.testrunner = nose.core.TextTestRunner(stream=self.result_stream,
                                                   descriptions=True,
                                                   verbosity=2,
                                                   config=config)

    def options(self, parser, env):
        parser.add_option("--skip-tests",
                          dest="skip_tests",
                          default=[],
                          help='A space separated list of tests to skip')

    def prepareTestLoader(self, loader):
        loader.selector = CustomTestSelector(loader.config)

    def finalize(self, result):
       

 log.info('Plugin finalized!')
```

Here, we're overloading three methods from the base Plugin class; `configure`, `options`, and `prepareTestLoader`. The `configure` method sets the `testrunner` argument and specifies some stdout streaming parameters. The `options` method receives a parser, similar to an ArgParse object, to which you can add options. The `prepareTestLoader` function receives a loader instance, which will be replaced by our custom `CustomTestSelector` to filter our desired test cases.

Once this is done, you need to register this plugin into the nose runner.

For other attributes that can be extended from the base Plugin class, you can review the [official documentation](https://nose.readthedocs.io/en/latest/writing_plugins.html) about how to write and extend nose.

**Reference:** [Nose Plugins](https://nose.readthedocs.io/en/latest/plugins/index.html)

## Registering the plugin in the nose runner

If you are using setuptools, the plugin must be included in the entry points of your package setup file. Your plugin will become available to your `nosetests` command once you run `install` or `develop`.

```python
setup(name='CustomSelectorPlugin',  
    entry_points = {
        'nose.plugins.0.10': [
            'custom_selector = custom_selector:CustomSelectorPlugin'
            ]
        },
    )
```

You can also do this programmatically without setuptools.

```python
from .plugin import CustomSelectorPlugin  
import nose

def run_test():  
    nose.main(addplugins=[CustomSelectorPlugin()])

def main():  
    run_test()

if __name__ == '__main__':  
    main()
```

Then you can execute this file directly and all the default `nosetests` options will be available, along with your newly created plugin options.

## Conclusion

Nose offers a robust API for extending the default behavior of collection, selection, and reporting. The provided nose API's are clear and concise to understand. The available official documentation is self-explanatory. It's highly recommended to use the default patterns on your tests schemas , but if this doesn't fit your case, feel free to extend nose using a similar approach.

## Next steps

In the next article in this Nose series, we will cover how to run tests in parallel and how to use test attributes to improve the time spent running tests. Stay tuned!
