---
title: "Python Function Serialization"
date: 2016-01-10T09:39:30+02:00
draft: false
---

# Function Serialization with Python and JavaScript

If for any specialized purpose you find yourself needing to call Python code from JavaScript, and vice versa, you may want to look into my experimental project, 
[Slurpy](https://github.com/niedbalski/slurpy)

One challenging aspect during the development was the implementation of the function serialization mechanism. In this post, I'll share the Python class responsible for function serialization, which produces a JSON representation.

Here is my `FuncMarshal` class:

```python
#!/usr/bin/env python
# -*- coding: utf-8 -*-

from hashlib import md5
import marshal  
import json  
import types


class FuncMarshal:

    @classmethod
    def md5sum(cls, serialized):
        hasher = md5()
        hasher.update(serialized)
        return hasher.hexdigest()

    @classmethod
    def serialize(cls, function):
        if callable(function):
            serialized = json.dumps((
                            marshal.dumps(function.func_code),
                            function.func_name,
                            function.func_defaults)
                        )
            return (cls.md5sum(serialized), serialized)

    @classmethod
    def deserialize(cls, encoded):
        (code, name, defaults) = json.loads(encoded)
        code = marshal.loads(code)

        return (cls.md5sum(encoded), \
                types.FunctionType(code, globals=globals(), name=str(name), \
                                                            argdefs=defaults))
```

This class is composed of three class methods:

- `md5sum`: This computes the MD5 hash of the serialized function for data integrity purposes.
- `serialize`: This serializes a callable function to a JSON string using Python's `marshal` and `json` modules.
- `deserialize`: This method reverses the serialization process, converting the JSON string back to the original function.

To maintain data integrity, I've also incorporated an MD5 checksum of the resulting serialized object. However, this step is optional and not particularly crucial to the functionality of the code.

If you've encountered a similar task or have a better approach for this type of serialization, please do share. Any suggestions for improvement are always welcome.