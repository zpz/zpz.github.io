---
layout: post
title: Guaranteed Finalization Without Context Manager
excerpt_separator: <!--excerpt-->
tags: [Python]
---

In the [last post](https://zpz.github.io/blog/multiprocessing-with-conveniences/), I said
"I'm a little nervous about daemon threads," so I used context manager to ensure a
background thread is properly closed. However, in the same post, there was indeed a situation where context manager is not a good tool.<!--excerpt-->
Specifically, I needed to use object `a` of class `A` in class `B`; I ended up entering the context for `a` by calling
`a.__enter__()` in `B.__init__`, and exiting the context by calling `a.__exit__()`
in `B.__del__` and in a few other methods that may terminate `B`.
To be fair, this is not a proper use of context manager. This calls `__enter__` and `__exit__` in a brute-force way without the benefits of context manager.

The point of context manager is to "ensure a finalization function is called, even if exceptions happen before that," and in some sense without explicitly calling the function.
If such a need of finalization is there, yet context manager is not usable, then what is the alternative?

In the case of a background thread, we can make it a "daemon thread". The official doc states that

> A thread can be flagged as a “daemon thread”. The significance of this flag is that the entire Python program exits when only daemon threads are left.

The doc is quick to warn:

> **Note:** Daemon threads are abruptly stopped at shutdown. Their resources (such as open files, database transactions, etc.) may not be released properly. 

So, a daemon thread should still trigger proper finalization, if needed, when it is demolished.
This is only not done in the "context manager" fashion.

As it happens, the `Queue` class in `multiprocessing` uses a daemonic thread and has some mechanism to "finalize" the thread. In this post, I'll walk through this mechanism and see whether it can be useful in my own code.

## The multiprocessing Queue

The class `Queue` is defined in `multiprocessing.queues`. The relevant code is as follows:

```python
from .util import Finalize


class Queue(object):

    def put(self, obj, block=True, timeout=None):
        if self._closed:
            raise ValueError(f"Queue {self!r} is closed")
        if not self._sem.acquire(block, timeout):
            raise Full

        with self._notempty:
            if self._thread is None:
                self._start_thread()
            self._buffer.append(obj)
            self._notempty.notify()

    def close(self):
        self._closed = True
        close = self._close
        if close:
            self._close = None
            close()

    def _start_thread(self):
        # Start thread which transfers data from buffer to pipe
        self._buffer.clear()
        self._thread = threading.Thread(
            target=Queue._feed,
            args=(self._buffer, self._notempty, self._send_bytes,
                  self._wlock, self._reader.close, self._writer.close,
                  self._ignore_epipe, self._on_queue_feeder_error,
                  self._sem),
            name='QueueFeederThread'
        )
        self._thread.daemon = True

        self._thread.start()

        if not self._joincancelled:
            self._jointhread = Finalize(
                self._thread, Queue._finalize_join,
                [weakref.ref(self._thread)],
                exitpriority=-5
                )

        # Send sentinel to the thread queue object when garbage collected
        self._close = Finalize(
            self, Queue._finalize_close,
            [self._buffer, self._notempty],
            exitpriority=10
            )

    @staticmethod
    def _finalize_join(twr):
        thread = twr()
        if thread is not None:
            thread.join()

    @staticmethod
    def _finalize_close(buffer, notempty):
        with notempty:
            buffer.append(_sentinel)
            notempty.notify()

    @staticmethod
    def _feed(buffer, notempty, send_bytes, writelock, reader_close,
              writer_close, ignore_epipe, onerror, queue_sem):
        ...
        while 1:
            try:
                ...
                try:
                    while 1:
                        obj = bpopleft()
                        if obj is sentinel:
                            debug('feeder thread got sentinel -- exiting')
                            reader_close()
                            writer_close()
                            return

                        ...
                except IndexError:
                    pass
            except Exception as e:
                if ignore_epipe and getattr(e, 'errno', 0) == errno.EPIPE:
                    return
                if is_exiting():
                    return
                else:
                    queue_sem.release()
                    onerror(e, obj)

_sentinel = object()
```

A summary on the high level is this:

1. The first time `put` is called, `_start_thread` is called to start a background thread. The thread is daemonic, and it runs `_feed`.
2. Two finalizers are set up. One is about the thread and involves the function `_finalize_join`; the other is about the queue object itself and involves the function `_finalize_close`.

Obviously, the class `Finalize` is the main power here; I'll turn to it in a moment.
For now, a little more about how things fit together during tear-down.

First, how does the thread terminate? The function `_feed` shows that this happens either during certain error conditions, or if the buffer presents the sentinel value.

Second, we see `_finalize_close` puts the sentinel value in the buffer.

Third, `_finalize_join` simply waits for the thread to exit. When will the wait be over? Of course, once the sentinel has been put in the buffer.

Incidentally, the method `close` calls the `Finalize` that is associated with `_finalize_close`, which would cause the background thread to terminate. We know that a call to `close` is not mandatory. Often we don't call it. Hence, the method `close` *proactively* calls the finalizer. If `close` is not called, the finalizer runs automatically somewhen, somehow.

## The `Finalize` utility class

The class `Finalize` is defined in `multiprocessing.util`. The main parts of the code is listed below:

```python
import itertools
import weakref
import atexit

_finalizer_registry = {}
_finalizer_counter = itertools.count()

class Finalize(object):
    '''
    Class which supports object finalization using weakrefs
    '''
    def __init__(self, obj, callback, args=(), kwargs=None, exitpriority=None):
        if (exitpriority is not None) and not isinstance(exitpriority,int):
            raise TypeError(
                "Exitpriority ({0!r}) must be None or int, not {1!s}".format(
                    exitpriority, type(exitpriority)))

        if obj is not None:
            self._weakref = weakref.ref(obj, self)
        elif exitpriority is None:
            raise ValueError("Without object, exitpriority cannot be None")

        self._callback = callback
        self._args = args
        self._kwargs = kwargs or {}
        self._key = (exitpriority, next(_finalizer_counter))
        self._pid = os.getpid()

        _finalizer_registry[self._key] = self

    def __call__(self, wr=None,
                 # Need to bind these locally because the globals can have
                 # been cleared at shutdown
                 _finalizer_registry=_finalizer_registry,
                 sub_debug=sub_debug, getpid=os.getpid):
        '''
        Run the callback unless it has already been called or cancelled
        '''
        try:
            del _finalizer_registry[self._key]
        except KeyError:
            pass
        else:
            if self._pid != getpid():
                res = None
            else:
                res = self._callback(*self._args, **self._kwargs)
            self._weakref = self._callback = self._args = \
                            self._kwargs = self._key = None
            return res


def _run_finalizers(minpriority=None):
    '''
    Run all finalizers whose exit priority is not None and at least minpriority
    Finalizers with highest priority are called first; finalizers with
    the same priority will be called in reverse order of creation.
    '''
    ...  # run entries in `_finalizer_retristry`


def _exit_function(...):
    ...  # call `_run_finalizers`
    

atexit.register(_exit_function)
```

Among the first things to notice is that a `Finalize` object is callable,
hence the call to `self._close` in `Queue.close`.

What `Finalize.__call__` does is to execute the callback function (line 47) that has been passed to `Finalize.__init__`, along with any parameters.

`Finalize.__init__` registers the object itself in the global registry (line 29). Items in the registry are executed at normal interpreter termination, as guaranteed by the use of `atexit`.

So, we've arrived at this comforting guarantee: the callback function that is passed to `Finalize` will be executed when the program terminates.

Now, note another thing: besides a callback function, an "object" may be passed to `Finalize`.
If so, the object is wrapped in a weak reference:

```python
        if obj is not None:
            self._weakref = weakref.ref(obj, self)
```

What's the role of this `self._weakref`? Actually we cannot find any use of it in `Finalize`. Unused variable! A bug in Python's standard library. Yay!

Think about this in the `multiprocessing.Queue` use case: the background thread in `Queue` is guaranteed to be cleaned up at program termination. Good. But wait! What if the program runs for a long time, and multiprocessing queues are used liberally here and there, now and then... Do all the daemon threads live on till the end of the program? There could be too much garbage buildup!

Although cleanup at program termination is good, we'd prefer that to be only the last resort. If possible, we'd like the cleanup to happen as soon as possible, for example, when the `Queue` object ends its life. Notice that although the thread is started by the queue, it will not be terminated when the queue terminates (neither will it interfere with the queue's termination), because the thread is daemonic.

So, a new question is: can we trigger the callback sooner, not `atexit`, but, say, when the queue object is finished?

Yes, we can! (Barack Obama, 2008).

Or rather, the weakref can.


## Weakref

The official doc for `class weakref.ref(object[, callback])` has this to say:

> Return a weak reference to `object`. The original object can be retrieved by calling the reference object if the referent is still alive; if the referent is no longer alive, calling the reference object will cause `None` to be returned. If `callback` is provided and not `None`, and the returned weakref object is still alive, the callback will be called when the object is about to be finalized; the weak reference object will be passed as the only parameter to the callback; the referent will no longer be available.

This sounds very serious and precise, but also a little mysterious at more than one place, for example

- "...will be called when the object is about to be finalized": What is "to be finalized"?
- "the weak reference object will be passed...; the referent will not longer be available": What? "No longer available"? What kind of situation is that?

Please be sure to distinguish "the weak reference object" and the "referent". The former is the instance of the class `weakref.ref`, whereas the latter is the `object` passed to it.
(Yes, `weakref.ref` is a class, not a function.)

To shed some light on this passage, we could dig into the source code. However, the `weakref` code is in `C`, which we would rather avoid. Let's make some empirical observations by code.

```python
import weakref


class My:
    def __init__(self, name):
        self.name = name

    def __del__(self):
        print('    -- del', self.name)


def cleanup(wr):
    print('    cleanup')
    obj = wr()
    if obj is None:
        print('    obj is gone')
        return
    print('    name:', obj.name)


def worker(refs):
    me = My('yes')
    refs[me.name] = weakref.ref(me, cleanup)
    print('  leaving worker')


def main():
    print('to enter worker')
    refs = {}
    worker(refs)
    print('left worker')
    print(refs)


if __name__ == '__main__':
    main()
```

Running this script (in Python 3.10.4), I got

```
$ python3 wr.py 
to enter worker
  leaving worker
    -- del yes
    cleanup
    obj is gone
left worker
{'yes': <weakref at 0x7fcc87bdc6d0; dead>}
```

This suggests,

1. When the function `worker` exits, its local variable `me` gets garbage collected, which triggers `me.__del__`.
2. The callback function (`cleanup`) in the weakref object is triggered after `__del__`.
3. The "when the object is about to be finalized" is referring to a step *after* `__del__`. That might be the actual destruction of the object `me`.
4. In the callback, `me` is no longer usable. The weakref object says the referent is already dead and gone. It might be correct to think: after the call to `__del__`, the object is no longer usable.

Now, I can summarize these findings as instructions for user of `weakref.ref(object, callback)`:

1. The callback is something we want to trigger once `object` is garbage collected.
2. Conceptually, this imposes no requirement on the relationship between `object` and `callback`. For example, `callback` does not need to be about something that is created by `object`.
3. `callback` takes a single parameter which is a weakref object; but the function should not use this parameter---it's just a placeholder (unless we get creative).
4. We need to keep the weakref object alive in order for `callback` to be executed. For example, we may put the weakref object in a global container to achieve this.

Now look back at the code of `Queue`. On line 48, the `Finalize` object is attached to the object as `self._close`. In order to invoke the callback, the finalizer needs to be alive when the object has been garbage collected, yet the finalizer is an attribute of the object? Sounds like a deadlock.

To break this deadlock, the finalizer object has to be referenced in a scope larger than `Queue`. In that case, `self._close` is but one reference to the finalizer, hence termination of the queue, thereby `self._close`, won't kill the finalizer. This is not hard to do. In he code of `Finalize`, on line 29, the `Finalize` object is placed in the global container `_finalizer_registry`. With this, we are free to create a finalizer attribute within an object to finalize that object!

In fact, in `Queue`, we could have written

```python
        Finalize(
            self, Queue._finalize_close,
            [self._buffer, self._notempty],
            exitpriority=10
            )
```

This looks dangling and ephemeral, but it's OK. The `Finalize` object once created is kept globally. It is assigned to `self._close` only because the method `Queue.close` wants to use this finalizer.

Another thing to note is that the callback function `Queue._finalize_close` is a static method. This is to make sure it does not hold a reference to the `Queue` instance. Look at `Finalize.__init__` again. Because the `Finalize` object will be kept globally, it can not contain any (non-weak) reference to `obj`; otherwise there will be a deadlock---the finalizer waits to act upon `obj`'s termination, but the finalizer itself references `obj` hence prevents the latter's termination. To summarize the requirements on the parameter `callback` of `Finalize.__init__`:

1. If `callback` is a method of the class of `obj`, it must be a static method or class method.
2. `callback` can not take `obj` in its arguments `args` or `kwargs`.

One more thing: the weakref doc says the callback function (the parameter to `weakref.ref`) takes a single parameter, which is the (useless) weak reference object. In contrast, the callback passed to `Finalize` is more flexible. It can take arguments via `args` and `kwargs`. The `Finalize` code re-packages things in its method `__call__`. The `Finalize` instance itself is the callback parameter to `weakref.ref`.

Now the instructions for `weakref.ref` can be revised into instructions for `Finalize` as follows. To ensure finalization of the instance of a custom class:

1. Define the finalization operation in a `classmethod` or `staticmethod`.
2. Create an instance attribute as a `Finalize` object, taking `self` and that class/static method as arguments, along with additional arguments via `args` and `kwargs` as long as they don't reference `self`.

I have phrased this in terms of a custom class; that is just for convenience. `Finalize` is more general, and its `__init__` parameter `obj` is optional. For example, `Finalize` can be used to register a regular function to do cleanup when the program terminates.

## Finale

Given these new understandings, let's upgrade the test script:

```python
from multiprocessing.util import Finalize


class My:
    def __init__(self, name):
        self.name = name
        self._fin = Finalize(self, type(self)._finalize, args=(name, ))

    def __del__(self):
        print('    -- del', self.name)

    @classmethod
    def _finalize(cls, name):
        print('    -- fin', name)
        print("    I'm doing all kinds of cleanup")


def worker():
    me = My('yes')
    print('  leaving worker')


def main():
    print('to enter worker')
    worker()
    print('left worker')


if __name__ == '__main__':
    main()
```

Running it:

```
$ python3 wr.py 
to enter worker
  leaving worker
    -- del yes
    -- fin yes
    I'm doing all kinds of cleanup
left worker
```

This is very generic, very clean, very powerful. I'm no longer nervous about daemon threads.

Just don't overuse it, though.

Yes, we can! (Barack Obama, 2008)