---
title: "Mental Notes About Python Asyncio"
date: 2016-06-26T09:34:53+02:00
draft: false
---

# Reflections on Python's New Asyncio Module

Today, I'd like to share some observations after spending a few hours experimenting with Python's new Asyncio module, which is based on the PEP 3156 specification.

### Multithreading War: The Final Battle

The introduction of the Asyncio module serves as definitive confirmation that the multithreaded war within Python has reached a conclusion. It suggests that unless some core developers have started implementing alternate strategies, such as Transactional Memory or Automatic Mutual Exclusion, the focus has shifted towards enhancing the Asynchronous I/O API to combat I/O bottlenecks rather than offering a genuine lock-free API of multithreading. Considering the scale of work required, this approach seems to be the more logical choice.

### Design of Asyncio API

The design of the Asyncio API appears to address the rampant spread of coroutine/event-based state machine libraries like Gevent, Tornado, Eventlet, and many others, which are attempting to solve the same issue: Event-based I/O Access and collaborative co-routines. It also seems that the API was not intended for direct use by developers, but rather, by those who develop libraries on top of it.

The Asyncio API provides a single event loop per process, which you can access via the `get_event_loop` function. You can begin adding event generator objects such as callbacks, coroutines, tasks, futures, pipes, etc. directly. However, keep in mind that all your code, along with the libraries you use, must be asynchronous. Otherwise, the scheduler could potentially timeout or block your callback queue.

### Scheduling Co-routines

The Asyncio API exposes methods like `call_soon()`, `call_later()`, and `call_at()` for scheduling co-routines. These cater perfectly to almost all common use cases and are well-documented and self-explanatory.

### Coroutines vs Tasks

Interestingly, the module differentiates between co-routines and tasks. I am still in the process of figuring out the distinction, as a task can be converted into a co-routine directly.

### Managing Internet Connections

The Asyncio module offers a simplified and effective suite for managing internet connections. It significantly alleviates the complexity typically associated with using Twisted, and this will be the focus of a future article.

### Use Case

My most common use case for a co-routine framework involves attaching a set of tasks and explicitly waiting for their completion. I also expect to have a mechanism to notify sibling co-routines about events, lock them, and if needed, block the entire loop. The Asyncio module caters to all these functionalities perfectly, making it an ideal standard replacement for my code that relies on Gevent and Greenlets.

### asyncio.Future

The API also presents a class called `asyncio.Future`, which bears a striking resemblance to the JavaScript jQuery `$.Deferred` object. I've found myself using `$.Deferred` quite frequently. An equivalent use case in JavaScript might look something like the following:

```javascript
var f = function() {  
    return $.Deferred(function(d) {  
        asynFunction().done(function(results) {
            d.resolve(results);
        });
    }).promise();
}

w = $.when(new f(), new f(), new f());  
w.done(function() {  
    console.log(arguments);  
});

w.fail(function() {  
    console.log('failed');  
});
```

In this example, a collection of `$.Deferred` objects is passed to the `$.when` function, which adds them to a callback queue and waits for them to reach a completed or failed state. A similar implementation using the new Asyncio module might look like this:

```python
#!/usr/bin/env python3.3
# -*- coding

: utf-8 -*-

import asyncio

chunk = lambda ulist, step: map(lambda i: ulist[i:i+step],  
range(0, len(ulist), step))

loop = asyncio.get_event_loop()  
chunks = chunk([x for x in range(0, 1000)], 8)

def sort_me(chunk, future):  
    future.set_result(sorted(chunk))

def create_future(chunk):  
    f = asyncio.Future()
    loop.call_soon(sort_me, chunk, f)
    return f

@asyncio.coroutine
def run():  
    for chunk in chunks:
        yield from create_future(chunk)

def main():  
    loop.run_until_complete(run())

main()
```

This code, although rudimentary, simply sorts chunks of integer sets. Notice that the coroutines are invoked using the new `yield from` operator, and the `loop` method `run_until_complete` functions similarly to the JavaScript `$.when` method.

### Closing Thoughts

In my opinion, the Asyncio module represents a significant step towards a more reliable Python, tackling the prevalent I/O bottlenecks of modern computers with an appealing API. It also brings much-needed standardization to the growing assortment of asynchronous orchestration libraries.

Personally, I would have preferred a more simplified API similar to jQuery's `$.Deferred` with fewer methods and cleaner interfaces. However, I do understand that the core Asyncio module needs to cater to a broad range of user scenarios.

In upcoming articles, I'll continue to explore and write more about this module.