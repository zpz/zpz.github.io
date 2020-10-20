---
layout: post
title: "Simple Stream Processing With I/O Steps"
excerpt_separator: <!--excerpt-->
tags: [python]
---

Suppose we need to process a stream of data in a sequence of steps. If all the steps are CPU-bound, then we just chain up the steps (or operations, or functions). Each data item goes through the CPU-bound functions one by one; the CPU is fully utilized. Now, if one or more of the steps involve I/O, typical examples beining disk I/O or http service calls, things get interesting.<!--excerpt-->


The interesting part is that an I/O bound operation spends much time waiting on something external instead of using the CPU to crunch through numbers. If we naively call the I/O function and wait for its result, much of the time is wasted, because during the time of waiting, the CPU may very well do something else useful. The solution, of course, is concurrency---let multiple calls to the I/O function be active at the same time,  so that their waiting operiods overlap. Nowadays a good way to do I/O concurrency is using `asyncio`.  In this post, I'll develop a few utilities to make this task nice and easy.

# Async generators and async iterators

Since much of the code will run in an async context, I'll require the data stream to be an *async iterable*, that is, we're going to consume the data with `aync for` most of the time. As it turned out, we need a little more than *async iterable*---we need the data stream to be an *async iterator* so that we can directly call `data.__anext__()`. Fortunately, it's easy to create async iterators. In particular, an async generator returns an async iterator.

An *async generator* is an async function that contains `yield` statements. For example, this is a small convenience function that turns a (sync) iterable into an async iterator:

```python
import asyncio
from collections.abc import Iterable

async def stream(x: Iterable):
    for xx in x:
        yield xx
        await asyncio.sleep(0)
```

Let's verify the types of things:

```python
In [2]: x = range(10)

In [3]: y = stream(x)

In [4]: type(y)
Out[4]: async_generator

In [5]: from collections.abc import AsyncIterator

In [6]: isinstance(y, AsyncIterator)
Out[6]: True
```

Note that although `stream` is an *async* function, we don't call it with `await stream(...)`.  Instead we directly call `stream(...)` and the result is an *async generator*, which satisfies the interfact of `AsyncIterator`. A little puzzling, I know. Just get used to it. The relation between `AsyncGenerator` and `AsyncIterator` is made clear in [the official doc about `collections.abc`](https://docs.python.org/3/library/collections.abc.html).

Sure enough, we can consume the elements of an `AsyncIterator` (or just `AsyncIterable`) with `async for`:

```python
In [11]: import asyncio

In [12]: async def foo():
    ...:     async for z in y:
    ...:         print(z)
    ...: 

In [13]: asyncio.run(foo())
0
1
2
3
4
5
6
7
8
9

In [14]: 
```

# Transformers

I'm going to call the mechanism that handles I/O operations a `transformer`.
It takes data items out of an `AsyncIterator`, makes concurrent calls to an async I/O function, and outputs the results in the same order that the data items come in.

The input can be an infinite stream. The design must take care to have input consumption, concurrent calls, and output in control so that no component "starves" or "blows up". A good tool for such control is a size-capped queue.

```python
import typing
from typing import Callable, Awaitable, Any, TypeVar

T = TypeVar('T')

async def transform(
    in_stream: typing.AsyncIterator[T],
    func: Callable[[T], Awaitable[Any]],
    *,
    workers: int,
    **func_args,
):
    if workers < 2:
        async for x in in_stream:
            y = await func(x, **func_args)
            yield y
        return

    # to be continued ...
```

The async function parameter `func` represents an I/O operation.
(It doesn't have to be "I/O". For example, it can be the async API to a service that is running in other processes.)
The parameter `workers` specify how many concurrent calls are allowed to the async function `func`. The case where `workers < 2` is simple and has no concurrency at all; we get it out of the way first. The func part comes next.

```python
    # ... continuing

    out_stream = asyncio.Queue(workers * 8)
    finished = False
    lock = asyncio.Lock()
    NO_MORE_DATA = object()

    async def _process():
        nonlocal finished
        while not finished:
            async with lock:
                if finished:
                    break
                try:
                    x = await in_stream.__anext__()
                except StopAsyncIteration:
                    finished = True
                    await out_stream.put(NO_MORE_DATA)
                    break
                fut = asyncio.Future()
                await out_stream.put(fut)

            y = await func(x, **func_args)
            fut.set_result(y)

    t_workers = [
        asyncio.create_task(_process())
        for _ in range(workers)
    ]

    while True:
        fut = await out_stream.get()
        if fut is NO_MORE_DATA:
            break
        yield await fut

    for t in t_workers:
        await t
```

There are four components to this:

- Launch a number of concurrent, "background" tasks. Each task repeated fetches items from the input stream, processes them, and places them in an output queue.
- The processing function run by each task, which is an infinite loop until the input stream runs out.
- `yield` elements of the output queue to the caller.
- How to maintain the order of the output to be consistent with the input.

The tasks are launched by `asyncio.create_task`. At the end of the funcion, we take care to `await` on each task to be sure they finish up properly. Tasks are scheduled by the event loop to run "soon".
Let's say `workers` is 4, then 4 copies of the function `_process` are running concurrently.

In `_process`, we repeatedly fetch the next element from `in_stream`.
Once one task sees the end of `in_stream`, we need to prevent other tasks from calling `in_stream.__anext__`. This is achieved by an indicator variable `finished` and a lock.

Look closely at what is controlled by the lock.
Although the loop is controlled by `while not finished`, the indicator `finished` is checked again after the lock is acquired, because, during the time of acquiring the lock, `finished` may have been set by another task. Once within the lock, `finished` won't be changed by another task (because the code that changes `finished` is in the locked block).

Also in the lock scope is the fetching of the next item from `in_stream`, as well as handling of `StopAsyncIteration`. This means that no two tasks are trying to fetch from `in_stream` at the same time. If `in_stream` has run out, then the task that first knows it has set `finished` to `True`. When another task gets the lock and intends to fetch from `in_stream`, it will see `finished` and quit.

To maintain the order of the output, once a data item is fetched out of `in_stream`, a `Future` object is immediately created to place-hold for this item, and placed in the output queue.
Now the order-keeping spot is secured, we run `func` on the data element. This can take as long as it needs. Once done, the result is set to the `Future` object.

Please note that placement of the `Future` object in the output queue also happens within the lock. Otherwise the output order can not be guaranteed. The lock also guarantees that the indicator `NO_MORE_DATA` is the very last element put in `out_stream`.

Finally, all these background tasks are set up and doing their job. All the rest we need to do is wait at the end of the output queue, and yield the results to the caller as they become available. One caveat is that the elements in the queue are `Future` objects. When a `Future` object is taken off the queue, it may not have result yet, so we `await` it and yield the result. If a later element has result sooner, it will sit there waiting for its turn, because the queue is FIFO (first-in first-out).

Let's verify it works....

# Unordered transformers

```python

async def unordered_transform(
    in_stream: typing.AsyncIterator[T],
    func: Callable[[T], Awaitable[Any]],
    *,
    workers: int = None,
    **func_args,
):
    assert workers > 1
    out_stream = asyncio.Queue(workers * 8)
    finished = False
    lock = asyncio.Lock()
    n_active_workers = workers
    NO_MORE_DATA = object()

    async def _process():
        nonlocal finished
        while not finished:
            async with lock:
                if finished:
                    break
                try:
                    x = await in_stream.__anext__()
                except StopAsyncIteration:
                    finished = True
                    break
            y = await func(x, **func_args)
            await out_stream.put(y)

        nonlocal n_active_workers
        n_active_workers -= 1
        if n_active_workers == 0:
            await out_stream.put(NO_MORE_DATA)

    t_workers = [
        create_loud_task(_process())
        for _ in range(workers)
    ]

    while True:
        y = await out_stream.get()
        if y is NO_MORE_DATA:
            break
        yield y

    for t in t_workers:
        await t
```

# Sinks

```python

async def drain(
        in_stream: typing.AsyncIterable[T],
        func: Callable[[T], Awaitable[None]],
        *,
        workers: int = None,
        log_every: int = 1000,
        **func_args,
) -> int:
    '''
    `func`: an async function that takes a single input item
    but does not produce (useful) return.
    Example operation of `func`: insert into DB.
    Additional arguments can be passed in via `func_args`.

    When `workers > 1` (or is `None`),
    order of processing of elements in `in_stream`
    is NOT guaranteed to be the same as the elements' order
    in `in_stream`.
    However, the shuffling of order is local.
    '''
    if workers is not None and workers < 2:
        trans = transform(
            in_stream,
            func,
            workers=workers,
            **func_args,
        )
    else:
        trans = unordered_transform(
            in_stream,
            func,
            workers=workers,
            **func_args,
        )

    n = 0
    nn = 0
    async for _ in trans:
        if log_every:
            n += 1
            nn += 1
            if n == log_every:
                logger.info('drained %d', nn)
                n = 0
    return nn
```

# Batch and un-batch

```python

async def batch(in_stream: AsyncIterable, batch_size: int):
    '''
    Take elements from an input stream,
    and bundle them up into batches up to a size limit,
    and produce the batches in an iterable.

    The output batches are all of the specified size, except
    possibly the final batch.
    There is no 'timeout' logic to produce a smaller batch.
    For efficiency, this requires the input stream to have a steady
    supply.
    If that is a concern, having a `Buffer` on the input stream
    may help.
    '''
    assert 0 < batch_size <= 10000
    batch_ = []
    n = 0
    async for x in in_stream:
        batch_.append(x)
        n += 1
        if n >= batch_size:
            yield batch_
            batch_ = []
            n = 0
    if n:
        yield batch_
```

```python
async def unbatch(in_stream: AsyncIterable):
    async for batch in in_stream:
        for x in batch:
            yield x
            await asyncio.sleep(0)
```

# Buffers

```python

class Buffer(AsyncIterator):
    def __init__(self, in_stream: AsyncIterable, buffer_size: int = None):
        self._data = asyncio.Queue(maxsize=buffer_size or 1024)
        self._in_stream = in_stream
        self._t_start = create_loud_task(self._start())

    async def _start(self):
        async for x in self._in_stream:
            await self._data.put(x)
        await self._data.put(NO_MORE_DATA)

    def __aiter__(self):
        return self

    async def __anext__(self):
        z = await self._data.get()
        if z is NO_MORE_DATA:
            raise StopAsyncIteration
        return z
```