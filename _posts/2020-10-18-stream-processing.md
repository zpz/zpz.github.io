---
layout: post
title: "Simple Stream Processing With I/O Operations"
excerpt_separator: <!--excerpt-->
tags: [python]
---

Suppose we need to process a data stream in a sequence of steps (or "operations" or "functions"). If all the steps are CPU-bound, then we just chain them up. Each data item goes through the CPU-bound functions one by one; the CPU is fully utilized. Now, if one or more of the steps involve I/O, typical examples being disk I/O or http service calls, things get interesting.<!--excerpt-->

The interesting part is that I/O-bound operations spend much time waiting on something external of the CPU. If we naively call the I/O function and wait for its result, much of the time is wasted, because during the time of waiting, the CPU may very well do something else useful. The solution, of course, is concurrency---let multiple calls to the I/O function be active at the same time, so that their waiting periods overlap. Nowadays a good way to do I/O concurrency is using `asyncio`. In this post, I'll develop a few utilities to make this task nice and easy.

# Async generators and async iterators

Since much of the code will run in an async context, I'll require the data stream to be an **async iterable**, that is, we're going to consume the data with `aync for ...` whenever we can. As it turned out, we need a little more than async *iterable*---we need the data stream to be an async *iterator* so that we don't have to be in the `async for ...` loop. Instead, we could request "the next item" by calling `data.__anext__()` directly. That gives the algorithm a lot of extra flexibility and, fortunately, this extra doesn't require a lot of extra effort.

In fact, the preferred way to create async iterators is by async generators. (Analogous to the recommendation of [using generators to produce iterators](https://treyhunner.com/2018/06/how-to-make-an-iterator-in-python/).)
An *async generator* is an async function that contains `yield` statements. For example, below is a small convenience function that turns a (sync) iterable into an async iterator:

```python
import asyncio
from collections.abc import Iterable, AsyncIterator

async def stream(x: Iterable) -> AsyncIterator:
    for xx in x:
        yield xx
```

Let's verify the types of things:

```python
In [2]: x = range(10)

In [3]: y = stream(x)

In [4]: type(y)
Out[4]: async_generator

In [5]: isinstance(y, AsyncIterator)
Out[5]: True
```

Note that although `stream` is an *async* function, we don't call it with `await stream(...)`.  Instead we directly call `stream(...)` and the result is an *async generator*, which satisfies the interface of `AsyncIterator`. A little puzzling, I know. Just get used to it. In fact, [`AsyncGenerator` inherits from `AsyncIterator`](https://docs.python.org/3/library/collections.abc.html).

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

The parameter `func` represents an async I/O operation.
(It doesn't have to be "I/O". For example, it can be the async API to a service that is running in other processes, such as [the model service I described previously]({{ site.baseurl }}/blog/batched-service-redesign/).)
The parameter `workers` specify how many concurrent calls are allowed to `func`. The case where `workers < 2` is simple and has no concurrency at all, so we get it out of the way first. The interesting part comes next.

Because the input could be an infinite stream, the design must take care to control input consumption, concurrent calls, and output production so that no component starves or gets overwhelmed. A good tool for such control is a size-capped queue.
The designed revolves around four questions:

- How is input consumed?
- How is output returned?
- How are input items processed concurrently?
- How is the order of output items controlled to follow that of the input?

```python
    # ... continuing

    out_stream = asyncio.Queue(workers * 8)
    lock = asyncio.Lock()
    finished = False
    NO_MORE_DATA = object()

    async def _process(in_stream, lock, out_stream, func, **kwargs):
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

            y = await func(x, **kwargs)
            fut.set_result(y)

    t_workers = [
        asyncio.create_task(_process(
            in_stream,
            lock,
            out_stream,
            func,
            **func_args))
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

Concurrency is achieved by as many as `workers` count of "background" tasks,
each of which independently and repeatedly fetches items from the input stream, processes them, and places results in an output queue.
As such, the input stream is not consumed in an `async for ...` loop. Rather, each task directly calls `__anext__` of the input stream (which is an `AsyncIterator`) to request the next item whenever the task is ready to work on the next item. The task ends when `__anext__` indicates the input stream has run out.

The tasks are launched by `asyncio.create_task` and scheduled by the event loop to run "soon".
Let's say `workers` is 4, then 4 copies of the function `_process` are running concurrently. At the end, we take care to `await` on each task to be sure they finish up properly. 

Once one task sees the end of `in_stream` in `_process`,
it needs to prevent other tasks from calling `in_stream.__anext__`. This is achieved by an indicator variable `finished` and a lock.

Please look closely at what is controlled by the lock.
Although the loop is controlled by `while not finished`, the indicator `finished` is checked again after the lock is acquired, because, during the time of acquiring the lock, `finished` may have been set by another task. After this check finds `finished` is `False`, `finished` won't be changed by another task (because the code that changes `finished` is in the locked block).

Also in the locked scope is the fetching of the next item from `in_stream`, as well as handling of `StopAsyncIteration`. This means that no two tasks are trying to fetch from `in_stream` at the same time. If `in_stream` has run out, then the task that first knows it will set `finished` to `True`. When another task acquires the lock and intends to fetch from `in_stream`, it will see `finished` and quit.

The order of output is maintained by `asyncio.Future` objects.
Specifically, once an item is fetched out of `in_stream`, a `Future` object is immediately created and placed in the output queue to "hold the spot" for the item.
After this, the item is processed by `func`---this can take as long as needed, since the spot for the result is already secured. Once done, the result is set to the `Future` object.

Please note that placement of the `Future` object in the output queue also happens with the lock. Otherwise the order of the outputs can not be guaranteed. The lock also guarantees that the indicator `NO_MORE_DATA` is the very last element placed in `out_stream`.

Finally, all these background tasks are set up and doing their job. All that remains to be done is wait on the output queue, and yield the results as they become available. One caveat is that the elements in the queue are not results, but rather `Future` objects. When a `Future` object is taken off the queue, it may not contain result yet, so we `await` it and yield the result.

Let's verify it works.

```python
import asyncio
import random
import pytest

async def f1(x):
    await asyncio.sleep(random.random() * 0.01)
    return x + 3.8


@pytest.mark.asyncio
async def test_transform():
    x = list(range(10000))
    expected = [v + 3.8 for v in x]
    s = transform(stream(x), f1, workers=10)
    got = [v async for v in s]
    assert got == expected
```

It does.

# Unordered transformers

In `transform`, a queue holds `Future` objects in order to maintain order of the elements.
If the processing of a later element has finished sooner,
it will wait for its turn to be de-queued and yielded, because
the queue is first-in first-out.
This could be suboptimal if the application does not need the output to be in the same order as the input.
These applications may use the "unordered" version below.

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
    lock = asyncio.Lock()
    finished = False
    NO_MORE_DATA = object()
    n_active_workers = workers

    async def _process(in_stream, lock, out_stream, func, **kwargs):
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
            y = await func(x, **kwargs)
            await out_stream.put(y)

        nonlocal n_active_workers
        n_active_workers -= 1
        if n_active_workers == 0:
            await out_stream.put(NO_MORE_DATA)

    t_workers = [
        asyncio.create_task(_process(
            in_stream,
            lock,
            out_stream,
            func,
            **func_args,
        ))
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

This differs from `transform` at two places.
First, it does not put `Future` objects in the output queue to "hold spots", because there is no need to maintain particular order of the elements. The input item is processed, and only the result is placed in the output queue.

Second, the indicator `NO_MORE_DATA` is not placed in the output queue as soon as the task sees the end of the input stream.
The reason is that at this moment there may very well be items being processed in other tasks. When they are done and their results enter the queue, they would appear *after* `NO_MORE_DATA`! The solution is to enqueue `NO_MORE_DATA` by the very last task that shuts down, and just before it shuts down.


# Sinks

A slight variant to "transformer" is a "sink", which processes data but does not produce results. Or, more accurately, we don't care to receive the results. Examples include writing data to files, inserting to databases, sending out emails, etc.
For verb for a "sink" is "drain", so that's the name of the function. It simply uses `transform` or `unordered_transform` and ignores their output.

```python

async def drain(
        in_stream: typing.AsyncIterable[T],
        func: Callable[[T], Awaitable[Any]],
        *,
        workers: int = None,
        log_every: int = 1000,
        **func_args,
) -> int:
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
    async for _ in trans:
        if log_every:
            n += 1
            if n % log_every == 0:
                print('drained', n)
    return n
```

The function reports progress every 1000 elements.
This is not needed in `transform` and `unordered_transform` because the downstream consumer can do that in its own way.


# Batch and un-batch

Some operations can take advantage of "vectorization", i.e., processing many items at once at higher efficiency than processing them one by one. A small convenience function can bundle up input elements ahead of such vectorized functions.

```python

async def batch(in_stream: AsyncIterable, batch_size: int):
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

Suppose a vectorized function takes `list` input and produces `list` of results, but the function coming next prefers to process one item at a time. The following small utility un-bundles the stream for it.

```python
async def unbatch(in_stream: AsyncIterable):
    async for batch in in_stream:
        for x in batch:
            yield x
```

Note that `batch` and `unbatch` do not have to be used in pairs.
It all depends on the actual need of the pipeline.

# Buffers

Look at the function `batch` above.
It does not consider situations like:
the next item is not coming after a long wait, so go ahead and produce the items collected so far as a smaller batch.
It always collects the specified number of items (with the only exception of the final batch, which may be partial).
This works fine if the input stream always has abundance of supply.
Otherwise there could be scenarios of glaring inefficiency.
Such scenarios could be mitigated by having a "buffer" ahead of it.

A "buffer" is a speed stabilizer.
The idea is simple: proactively fetch items from the input stream; if the downstream function does it need them now or consumes them at a lower pace than they are being fetched, just store them temporarily in a private pool.

```python
async def buffer(in_stream: AsyncIterable, buffer_size: int):
    out_stream = asyncio.Queue(maxsize=buffer_size)
    NO_MORE_DATA = object()

    async def buff(in_stream, out_stream):
        async for x in in_stream:
            await out_stream.put(x)
        await out_stream.put(NO_MORE_DATA)

    t = asyncio.create_task(buff(in_stream, out_stream))

    while True:
        x = await out_stream.get()
        if x is NO_MORE_DATA:
            break
        yield x

    await t
```

If the downstream consumer is slow, the function `buff` will proactively collect items from `in_stream` and stuff `out_stream` to its capacity.


# Put it all together

Let's cook up an example to show off all these utilities.

```python
x = [1, 5, 6, 3, 7, 9, 2, 4, 4, 5, 1]

async def diff_shoot(batch):
    return range(max(batch) - min(batch))


async def scale(x):
    return x * 2


class Sink:
    def __init__(self):
        self.sum = 0

    async def __call__(self, x):
        self.sum += x


batches = batch(stream(x), 3)
shots = transform(batches, diff_shoot, workers=3)
flat = buffer(unbatch(shots), 10)
doubled = unordered_transform(flat, scale, workers=2)

mysink = Sink()

n = asyncio.run(drain(doubled, mysink, workers=1))
print('processed', n, 'elements')
print('sum is', mysink.sum)
```

Before running it, let's figure out what the result should be.

1. `batch` will bundle `x` into 
   - `[1, 5, 6]`
   - `[3, 7, 9]`
   - `[2, 4, 4]`
   - `[5, 1]`
2. `diff_shoot` will find the ranges of these bundles to be 
`5 (= 6 - 1)`, `6 (= 9 - 3)`, `2 (= 4 - 2)`, `4 (= 5 - 1)`, and produce the ranges
   - `range(5)`
   - `range(6)`
   - `range(2)`
   - `range(4)`
3. `unbatch` will flatten these out (note that the upstream of `unbatch` provided `range` instead of `list` objects, and that's fine) into the stream
   - `0, 1, 2, 3, 4, 0, 1, 2, 3, 4, 5, 0, 1, 0, 1, 2, 3`

   These are 17 numbers and `buffer` does not change this.
4. `scale` doubles each of these numbers.
5. `drain` and `mysink` will add them up.

All in all, the final sum should be 64.
   
```python
In [3]: 

   ...: 
   ...: x = [1, 5, 6, 3, 7, 9, 2, 4, 4, 5, 1]
   ...: 
   ...: async def diff_shoot(batch):
   ...:     return list(range(max(batch) - min(batch)))
   ...: 
   ...: 
   ...: async def scale(x):
   ...:     return x * 2
   ...: 
   ...: 
   ...: class Sink:
   ...:     def __init__(self):
   ...:         self.sum = 0
   ...: 
   ...:     async def __call__(self, x):
   ...:         self.sum += x
   ...: 
   ...: 
   ...: 
   ...: batches = batch(stream(x), 3)
   ...: shots = transform(batches, diff_shoot, workers=3)
   ...: flat = buffer(unbatch(shots), 10)
   ...: doubled = unordered_transform(flat, scale, workers=2)
   ...: 
   ...: mysink = Sink()
   ...: 
   ...: n = asyncio.run(drain(doubled, mysink, workers=1))
   ...: print('processed', n, 'elements')
   ...: print('sum is', mysink.sum)
   ...: 
processed 17 elements
sum is 64

In [4]: 
```

# Final thoughts

A very nice feature of these functions is that the `AsyncIterator` type unites them all. The output of one can be the input of another. As a result, they can be used in a "pipe" fashion in flexible ways.

The `transform` and `unordered_transform` are like "mappers", but they don't have to take one input and generate one output. They could aggregate and expand, with the help of `batch` and `unbatch`.

The final example shows that `drain` could behave like a "reducer". But not quite, as `drain` can not produce an output stream. That's something to think about!
