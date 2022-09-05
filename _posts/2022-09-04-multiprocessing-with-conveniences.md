---
layout: post
title: A few Convenience Utilities for Python Multiprocessing
excerpt_separator: <!--excerpt-->
tags: [Python, concurrency]
---

In my use of Python's `multiprocessing` module, from time to time I'm bothered
by the lack of two convenience features, one concerning exception visibility,
and the other concerning logging. I've created a subclass of `Process` to add these features.<!--excerpt-->


## Subclassing Process to return worker results and exceptions

Usually, we use `multiprocessing` like this:

```python
import multiprocessing as mp

def main():
    p = mp.get_context('spawn').Process(target=func, args=..., kwargs=...)
    p.start()
    ...
    p.join()
```

If the target function (or "worker") crashes due to an exception, the main process would proceed as if nothing bad has happened. We have to deliberately check `p.exitcode`; even if that tells us something is wrong, we won't get the usual traceback printout. For crashes in the worker to be visible, we need to be sure things get printed out in the target function. That, of course, still does not stop the flow of the main process. We still need to check `p.exitcode`.

In this regard, `concurrent.futures.ProcessPoolExecutor.submit` is more pleasant to use.
The returned `Future` object will raise the exception that the worker has crashed with, if any, when we call `Future.result()` or `Future.exception()`.

By the way, another convenience provided by `concurrent.futures` is the method `result()`. The `Process` object does not have this. If the worker wants to return a simple value as the result, we have to pass in a `Queue` or something else to receive it in the child process and fetch it back to the main process.

In this article, I'll walk through a custom process class that addresses these inconveniences.

To set the stage, I only care about a "spawned" process, as [that is almost always what we should be using](https://pythonspeed.com/articles/python-multiprocessing/). So I'll create a subclass of `multiprocessing.context.SpawnProcess`, which in turn is a very thin wrapper around `multiprocessing.process.BaseProcess`.

After reading parts of the `multiprocessing` code (especially the "spawn" part, which I did not dive into), I decided it's black magic to me and I should not enhance it by hacking its internals. Instead, I should rely on its public API as far as it can get the enhancements done.

To subclass `BaseProcess`, three of its methods, namely `__init__`, `run`, and `start`, are most relevant. Here's the `cpython` source code:

```python
class BaseProcess(object):

    def __init__(self, group=None, target=None, name=None, args=(), kwargs={},
                 *, daemon=None):
        ...

    def run(self):
        '''
        Method to be run in sub-process; can be overridden in sub-class
        '''
        if self._target:
            self._target(*self._args, **self._kwargs)

    def start(self):
        '''
        Start child process
        '''
        self._check_closed()
        assert self._popen is None, 'cannot start a process twice'
        assert self._parent_pid == os.getpid(), \
               'can only start a process object created by current process'
        assert not _current_process._config.get('daemon'), \
               'daemonic processes are not allowed to have children'
        _cleanup()
        self._popen = self._Popen(self)
        self._sentinel = self._popen.sentinel
        # Avoid a refcycle if the target function holds an indirect
        # reference to the process object (see bpo-30775)
        del self._target, self._args, self._kwargs
        _children.add(self)
```

The basic idea is, well, basic: I shall override `run` to capture the result or exception, make them available
on the `self` object, and access them via a few new methods.

A critical fact to realize is that the `self` in `run` is **not** the `self` in `__init__`. The code of `run` executes in the child process. If I simply add attributes to `self` in `run`, they won't be reflected on the `self` in the main process. Similarly, if I add custom attributes to `self` in `__init__`, they won't be available in `run`.

As for `start`, it's not obvious at all how it arranges `run` to be run in the child process. There are quite a few hoops in between.

However, there's at least one connection between the main and the child processes, and that is the worker parameters.
For example, We often use queues as parameters to share things between the processes.

With this, the strategy is becoming clear. We can use a queue to collect things in the child process and make them available in the main process.

How should we do that? What about adding a parameter to `run`, or something?

No, we don't know how `run` is invoked and we certainly don't want to mess with that. Instead, we'll take a ride in `self._kwargs`---we just need to add some elements with unusual names so that they won't collide with user code, and use them on the back of the user. The code starts like this:


```python
import multiprocessing.connection
import multiprocessing.context
from concurrent.futures.process import _ExceptionWithTraceback


class TimeoutError(RuntimeError):
    pass


class SpawnProcess(multiprocessing.context.SpawnProcess):
    def __init__(self, *args, kwargs=None, **moreargs):
        if kwargs is None:
            kwargs = {}

        assert '__result_and_error__' not in kwargs
        reader, writer = multiprocessing.connection.Pipe(duplex=False)
        kwargs['__result_and_error__'] = writer

        super().__init__(*args, kwargs=kwargs, **moreargs)

        assert not hasattr(self, '__result_and_error__')
        self.__result_and_error__ = reader

    def run(self):
        result_and_error = self._kwargs.pop('__result_and_error__')
        # Upon completion, `result_and_error` will contain `result` and `exception`
        # in this order; both may be `None`.
        if self._target:
            try:
                z = self._target(*self._args, **self._kwargs)
            except BaseException as e:
                result_and_error.send(None)
                result_and_error.send(_ExceptionWithTraceback(e, e.__traceback__))
            else:
                result_and_error.send(z)
                result_and_error.send(None)
        else:
            result_and_error.send(None)
            result_and_error.send(None)

    def done(self):
        return self.exitcode is not None

    def result(self, timeout=None):
        self.join(timeout)
        if not self.done():
            raise TimeoutError
        if not hasattr(self, '__result__'):
            self.__result__ = self.__result_and_error__.recv()
            self.__error__ = self.__result_and_error__.recv()
        if self.__error__ is not None:
            raise self.__error__
        return self.__result__

    def exception(self, timeout=None):
        self.join(timeout)
        if not self.done():
            raise TimeoutError
        if not hasattr(self, '__result__'):
            self.__result__ = self.__result_and_error__.recv()
            self.__error__ = self.__result_and_error__.recv()
        return self.__error__
```

A few things to note. First, it does not use a queue. Instead, a `Pipe` is used for this simple case.
It will always contain two values upon completion of `run`, namely the result and the exception.

Second, in `run`, after capturing the exception and put it in the pipe, the exception is *not* `raise`d, hence the parent behavior in case of crashes is suppressed. (The parent behavior is to print out the exception info and traceback. But I was bitten by total silence more than once. Things are tricky.) This is consistent with the behavior of `concurrent.futures.ProcessPoolExecutor`. One should either call `result()` or check `exception()` in the main process. If the exception is printed in the child process, it's a duplicate of the printout in the main process, leading to some mess, if not confusion.

Third, new methods `done`, `result`, and `exception` have been provided to mimic the usage of `concurrent.futures.Future`. Both `result` and `exception` wait for completion with optional timeout, which can be quite handy.

The use of `_ExceptionWithTraceback` from `concurrent.futures` is also interesting.

We can design some tests for this class.

```python
def delay_double(x, delay=2):
    sleep(delay)
    if x < 10:
        return x * 2
    raise ValueError(x)


def test_process():
    t = SpawnProcess(target=delay_double, args=(3,))
    t.start()
    sleep(0.1)
    assert not t.done()
    assert t.is_alive()
    with pytest.raises(TimeoutError):
        y = t.result(0.1)
    with pytest.raises(TimeoutError):
        y = t.exception(0.1)
    assert t.result() == 6
    assert t.exception() is None
    t.join()

    t = cls(target=delay_double, args=(12,))
    t.start()
    with pytest.raises(TimeoutError):
        y = t.result(0.2)

    with pytest.raises(ValueError):
        y = t.result()

    e = t.exception()
    assert type(e) is ValueError

    t.join()
```

This does the job decently. Let's move on to another enhancement.


## Handling child process' logging in the main process

As a rule, library code should not configure log handlers regarding formatting, level, destination, etc.
That is the responsibility of the end-user's "entrypoint" script.

Similarly, when we start a new process, we should **not** configure logging
in the process, as that is the end-user's job and we don't know what exact settings they want.
However, the user's logging setup in the main process will not carry over to a child process that is created by the "spawn" method.
We can verify this behavior by a small script:

```python
import logging
import multiprocessing as mp


logger = logging.getLogger('mytest')


def worker():
    logger.error('worker error')
    logger.warning('worker warning')
    logger.info('worker info')
    logger.debug('worker debug')


def main():
    logger.error('main error')
    logger.info('main info')
    p = mp.get_context('spawn').Process(target=worker)
    p.start()
    p.join()
    logger.warning('main warning')
    logger.debug('main debug')


if __name__ == '__main__':
    logging.basicConfig(
        format='[%(asctime)s.%(msecs)02d; %(levelname)s; %(name)s; %(funcName)s, %(lineno)d] [%(processName)s]  %(message)s',
        level=logging.DEBUG,
    )
    main()
```

Running this script gets

```
[2022-09-04 21:51:14,607.607; ERROR; mytest; main, 16] [MainProcess]  main error
[2022-09-04 21:51:14,607.607; INFO; mytest; main, 17] [MainProcess]  main info
worker error
worker warning
[2022-09-04 21:51:14,664.664; WARNING; mytest; main, 21] [MainProcess]  main warning
[2022-09-04 21:51:14,664.664; DEBUG; mytest; main, 22] [MainProcess]  main debug
```

I don't know what mechanism printed the two messages in the child process. Obviously they abide by the default level of `WARNING`, and the logging config in the main process has no effect in the child process. Remember, the worker does not need to be a small and simple function. It may call any amount of other code that produces useful logging. This loss of logging in multiprocessing is definitely **not OK**.

Two solutions appear possible. The first is to send the main-process logging settings into the child process and enact them there. The second is to send the child-process logs back into the main process and handle them here.

For the first option, I didn't research how to send and enact logging settings. But there are other difficulties. For example, if user has configured logs to be written to a file, how do we deal with file states in the child process? What if there are multiple child processes? What about the order of log messages produced by multiple processes? Will the messages be messed up, hijacked half-sentence as we have fondly seen? Another example, suppose the user has configured logs to be sent to some remote service, and the handler involves credentials and such, which have all been taken care of in the main process. How can we "send and enact" these settings in a child process?

These make the first option an unlikely solution. So let's go with the second one. I'm going to implement a general solution and then simply use it in the `SpawnProcess` above, while the solution is also usable independent of this `SpawnProcess`.

The standard module `logging.handlers` has a `QueueHandler`, which places all logging messages in a queue.
This is precisely what's needed in the child process!
There's also a `logging.handlers.QueueListener`, which appears to be the counterpart to be used in the main process. Unfortunately, this class does not do all I need. I could customize `QueueListener` or roll my own.
Since it's simple enough, I decided implement the main-process side from scratch.

There are mainly three aspects to think about:

- How to collect logging messages in the child process.
- How to handle the messages in the main process.
- How to start/stop and hook things up.

The high-level usage will be like this:

1. In the main process, create an object (which contains a queue) and start it.
2. Pass this object to the child process.
3. In the child process, start it.

That's it. User should not worry about anything else. In order to properly wrap up and clean up, we use the object in a context manager. The behavior of certain methods of the object depends on whether it's in the main process or the child process. In order to tell whether it's in the main or the child process, I customize the pickling of the object---if being pickled, it's in the main process; if being un-pickled, it's in the child process. Here we start:

```python
class ProcessLogger:
    def __init__(self, *, ctx: multiprocessing.context.BaseContext):
        self._ctx = ctx
        self._t = None

    def __getstate__(self):
        # In the process that has created `self`.
        assert self._t is not None
        return self._q

    def __setstate__(self, state):
        # In another process.
        self._q = state

    def __enter__(self):
        if hasattr(self, '_t'):
            self._start_in_main_process()
        else:
            self._start_in_other_process()
        return self

    def __exit__(self, *args):
        if hasattr(self, '_t'):
            self._stop_in_main_process()
        else:
            self._stop_in_other_process()
```

The background thread `self._t` will be started in `_start_in_main_process`. The log queue `self._q` will also be created there, before being passed to the child process via pickling (`__getstate__`) and unpickling (`__setstate__`). Let's implement the operations in the child process:

```python
    def _start_in_other_process(self):
        root = logging.getLogger()
        root.setLevel(logging.DEBUG)
        qh = logging.handlers.QueueHandler(self._q)
        root.addHandler(qh)

    def _stop_in_other_process(self):
        self._q.close()
```

Now let's implement the main-process:

```python
    def _start_in_main_process(self):
        assert self._t is None
        self._q = self._ctx.Queue()

        self._t = threading.Thread(target=self._logger_thread, args=(self._q, ))
        self._t.start()

    @staticmethod
    def _logger_thread(q: multiprocessing.queues.Queue):
        threading.current_thread().name = 'logger_thread'
        while True:
            record = q.get()
            if record is None:
                break
            logger = logging.getLogger(record.name)
            if record.levelno >= logger.getEffectiveLevel():
                logger.handle(record)

    def _stop_in_main_process(self):
        assert self._t is not None
        self._q.put(None)
        self._t.join()
        self._t = None
```

The closing method of the context manager places a sentinel in the queue to signal
that the child process has terminated and there will be no more log messages.
This sentinel is checked by the background thread to stop waiting for more messages and exit.

Here it's worthwhile to talk a little bit about the "level" in logging. There are two places to specify "level": on a logger (i.e. what is returned by `logging.getLogger`) and on a handler. If not specified, the default is `WARNING` for a logger, and `DEBUG` (or everything) for a handler. In `_start_in_other_process`, we set `DEBUG` level to the root logger. If no other loggers in the child process gets its level specified, which should be the case, all logging messages on all levels are sent to the queue handler, which accepts everything and puts it in the queue. This is necessary because the child process has no idea what logging level the end-user (in the main process) decides to use. On the other hand, in `_logger_thread` in the main process, a message is filtered according to the level of its logger. This is as if the message has been issued in the *main* process by the same logger object via the same command (`logger.warning(...)`, `logger.info(...)`, `logger.debug(...)`, etc). If the call is below the effective level of the logger, the message is not sent to its handlers at all.

Now let's put this to use. Make some changes to the testing script as follows:

```python
def worker(pl):
    with pl:
        logger.error('worker error')
        logger.warning('worker warning')
        logger.info('worker info')
        logger.debug('worker debug')


def main():
    logger.error('main error')
    logger.info('main info')
    with ProcessLogger(ctx=mp.get_context('spawn')) as pl:
        p = mp.get_context('spawn').Process(target=worker, args=(pl,))
        p.start()
        p.join()
    logger.warning('main warning')
    logger.debug('main debug')
```

Running it, we get

```
[2022-09-04 22:08:52,313.313; ERROR; mytest; main, 18] [MainProcess]  main error
[2022-09-04 22:08:52,313.313; INFO; mytest; main, 19] [MainProcess]  main info
[2022-09-04 22:08:52,377.377; ERROR; mytest; worker, 11] [SpawnProcess-1]  worker error
[2022-09-04 22:08:52,377.377; WARNING; mytest; worker, 12] [SpawnProcess-1]  worker warning
[2022-09-04 22:08:52,378.378; INFO; mytest; worker, 13] [SpawnProcess-1]  worker info
[2022-09-04 22:08:52,378.378; DEBUG; mytest; worker, 14] [SpawnProcess-1]  worker debug
[2022-09-04 22:08:52,386.386; WARNING; mytest; main, 24] [MainProcess]  main warning
[2022-09-04 22:08:52,386.386; DEBUG; mytest; main, 25] [MainProcess]  main debug
```

as expected.

How does this implementation differ from `logging.handlers.QueueListener`? There are two differences. First, `QueueListener` requires `handlers` in its `__init__`. This implementation leaves it open and flexible. For one thing, it could be the case that different loggers have different handlers. `ProcessLogger` leaves that to be determined on the fly. Second, `QueueListener` uses a *daemon* thread to process the messages. I'm a little nervous about daemon threads. I think context manager is good.

## Handling logging in the custom `Process` class

The `ProcessLogger` is well and good, but it would be even better if end-user does not need to use it at all. In particular, if we have multiple levels of processes (where a child process creates its own child processess), it can become tedious. Since we have a custom process class, we may as well take this burden off the user if they use that class.

This is easy. Using the `kwargs` trick again, we just need to pass another thing---this time a `ProcessLogger` object---into the child process and use it.

```python
class SpawnProcess(multiprocessing.context.SpawnProcess):

    def __init__(self, *args, kwargs=None, **moreargs):
        ...
        assert '__worker_logger__' not in kwargs
        worker_logger = ProcessLogger(ctx=multiprocessing.get_context('spawn'))
        worker_logger.__enter__()
        kwargs['__worker_logger__'] = worker_logger
        ...

        super().__init__(*args, kwargs=kwargs, **moreargs)

        ...
        assert not hasattr(self, '__worker_logger__')
        self.__worker_logger__ = worker_logger

        ...

    def run(self):
        with self._kwargs.pop('__worker_logger__'):
            ...
```

`worker_logger.__enter__()` is called in `__init__` because `SpawnProcess` is not context managed. Then, where should `worker_logger.__exit__()` be called?. The method `__del__` is a natural choice, but it's not totally reliable. I'm going to take a brute-force approach: call it in all the several methods that could terminate the child process or the `SpawnProcess` object:

```python
    def _stop_logger(self):
        if hasattr(self, '__worker_logger__') and self.__worker_logger__ is not None:
            self.__worker_logger__.__exit__()
            self.__worker_logger__ = None

    def __del__(self):
        self._stop_logger()

    def terminate(self):
        super().terminate()
        self._stop_logger()

    def kill(self):
        super().kill()
        self._stop_logger()

    def join(self, *args, **kwargs):
        super().join(*args, **kwargs)
        if self.done():
            self._stop_logger()
```

This is not exactly elegant, but it seems to get the job done.
Now `SpawnProcess` can be used in a pattern similar to `concurrent.futures.Future`. It's less likely to overlook crashes of the child process, and logging in the child process is handled properly.

The code presented in this articular is available in the package [mpservice](https://pypi.org/project/mpservice/).
