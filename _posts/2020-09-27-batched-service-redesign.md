---
layout: post
title: "Service Batching from Scratch, Again"
excerpt_separator: <!--excerpt-->
tags: [Python, concurrency]
---

In [a previous post]({{ site.baseurl }}/blog/batched-service/), I described an approach to serving machine learning models with built-in "batching". The design has been used in real work with good results. Over time, however, I have observed some pain points and learned some new tricks. Finally, I got a chance to sit down and make an overhaul to it from scratch.<!--excerpt-->

This is a new design. It is not recognizable as a revision of the previous one. As such, you don't need to read the old post in order to understand this one. Nevertheless, let's re-visit the main points of the previous design:

1. It is assumed that the model is decomposed into three sequential components: (1) a `Preprocessor`; (2) a `VectorTransformer`; (3) a `Postprocessor`.
2. Both `Preprocessor` and `Postprocessor` are assumed light weight. They run in the main process. The `VectorTransformer`, as the meat of the model, does heavy lifting in its own process.
3. Data flows through the components via `multiprocessing` `Queue`s.
4. The service API supports a **singleton** endpoint and a **batch** endpoint. Queries to the first are automatically batched to take advantage of vectorized computation in the `VectorTransformer`. In contrast, the latter endpoint is an "express lane", meaning batch inputs (large or small) are processed as they arrive, skipping batching (they are already batches upon input). As we can see, the `VectorTransformer` always works on batches.
5. The "coordinator" in the main process uses `asyncio`.
 In particular, `asyncio` `Future`s are used to guarantee that, after all the waiting, batching, asynchorous flow and processing in multiple stages, a result is returned to the correct request that has been waiting for it.

Some limitations of this design include

1. The assumed three-component structure, as well as the assumption that they are light-heavy-light, are rigid. In reality, it's totally reasonable to have a pipeline with more, or less, than three components, and to have any number of heavy components among them.
2. The batch endpoint adds to the complexity of the implementation and the usage. Given that the singleton endpoint does batching behind the scene, *if* this is pushed to the limit of efficiency, then removing the batch endpoint would bring nice simplification without much loss in capability.
3. The design does not help the user to make full use of CPU cores. The `VectorTransformer` runs in its own process, and it in turn is free to do multiprocessing. However, that demands a certain level of coding skills on the user.
4. The `Preprocessor` and `Postprocesser`, sharing the main process with the async coordinator, are better non-blocking---i.e., async. That again demmands a certain level of coding skills. Moreover, the benefit in making them async (at a cost to simplicity) is likely small if these processors do not involve I/O, which is often the case.

To address these flaws, the new design has some very different ideas on the high level. (For ease of explanation, I will refer to this design the "framework").

1. It supports a sequence of components (or stages), however many, with no assumptions on their roles and relative expenses.
2. Each component is "traditional" synchronous code within a single process. Multiple instances of this component can run in multiple processes as *independent* "workers". The user just specifies the number of workers, and the rest is managed by the framework.
3. The "coordinator" and the workers run in separate processes. Each process may be instructed to run on a specific CPU core.
4. Data flows through the processes in `multiprocessing` `Queue`s.
5. There is another `Queue` for errors. If error occurs in any worker, an `Exception` object is short-circuited back to the coordinator in this queue. Only valid result will proceed to the next stage. The exception object will be paired up with the awaiting request, the same way a valid result is.
6. There is only a singleton endpoint. No batch endpoint.
7. Each component can decide to take batches or singletons as input. In the batch case, the framework assembles singletons (which are flowing in the queue) into a batch, and feed it to the component, which should output a batch result. The framework dissembles the batch result to singleton results, and places them in the queue. This way, the data that flows between the components are always singletons, and each component independently decides whether it take batches or singletons.

This is a lot of power in abstract! Let's get concrete.


## The coordinator

The entrypoint, or coordinator, of the whole thing is a class called `ModelService`. It manages workers, which are concrete subclasses of `Modelet`. Before describing `ModelService`, let's get a rough picture of `Modelet`.

```python
from abc import ABCMeta, abstractmethod

class Modelet(metaclass=ABCMeta):
    def __init__(self, ...):
        ...

    @abstractmethod
    def predict(self, x):
        ...

    def start(self, q_in, q_out, q_err):
        while True:
            x = get_input_from_q_in()
            try:
                y = self.predict(x)
                q_out.put(y)
            except Exception as e:
                q_err.put(e)

    @classmethod
    def run(cls, q_in, q_out, q_err, ...):
        modelet = cls(...)
        modelet.start(q_in, q_out, q_err)
```

The workflow for a `Modelet` is quite clear in this sketch. We'll fill in details later.

A sketch of `ModelService` is as follows.

```python
import asyncio
import multiprocessing as mp
import queue
from typing import Type, List

import psutil


class ModelService:
    def __init__(self, ...)
        ...

    def add_modelet(self, modelet: Type[Modelet], ...):
        ...

    def start(self):
        ...
        start_modelet_workers()
        start_gather_results()
        ...

    async def _gather_results(self):
        ...

    def stop(self):
        ...

    async def predict(self, x):
        place_x_in_queue()
        wait_for_result()
        return_result()
```

On a `ModelService` object, we call `add_modelet` to add one "component" at a time. We can add as many as needed; they become sequential components in the model pipeline. Note that one component is added by a single call to `add_modelet`, regardless of the number of workers it needs. Worker info is taken by `add_modelet`.

Once all components have been added, we call `start`, which, besides starting the worker processes, calls `_gather_results` to launch a background job. This job picks up results of the very last modelet (via some queue) and pairs it up with the awaiting request.

At this point, the model service is ready but idle. It will get work to do only after `predict` is called. That, of course, is what a *service* is about.

Requests are received by `predict`. In this method, the input is placed in some queue. Then it asynchronously waits for the result. The input will flow through the modelets. Once its result is produced by the final modelet, it gets picked up by `_gather_results` and is provided to the awaiting request, i.e. the particular call of `predict` that handled the particular request and placed the input in the queue.
Once getting the result, `predict` returns.

Note that both `_gather_results` and `predict` are async functions.

This is still somewhat abstract.
Let's start setting up `ModelService` so that we have concrete variables to refer to.

```python
class ModelService:

    def __init__(self,
                 max_queue_size: int = None,
                 cpus: List[int] = None):
        self.max_queue_size = max_queue_size or 1024
        self._q_in_out = [mp.Queue(self.max_queue_size)]
        self._q_err = mp.Queue(self.max_queue_size)

        self._uid_to_futures = {}
        self._t_gather_results = None
        self._modelets = []

        if cpus:
            psutil.Process().cpu_affinity(cpus=cpus)

        self._started = False
```

The list of queues `_q_in_out` will grow as we add modelets. The first member of it will take input data that is received by `ModelService.predict`. The worker(s) of the first modelet will take input from this queue and process it, and put the result in the second member of `_q_in_out`---which will be added when the first modelet is added.

The parameter `cpus` is a list of integers specifying on which CPU cores this "main" process should be running. If not specified, the execution may move between all available cores. For efficiency, it is recommended to specify a core. This is known as "CPU pinning". This is probably more important for modelets that perform intensive number crunching.

Now let's add a modelet.

```python
    def add_modelet(self,
                    modelet: Type[Modelet],
                    *,
                    cpus: List[int] = None,
                    **init_kwargs):
        # `modelet` is the class object, not class instance.
        assert not self._started
        q_in = self._q_in_out[-1]
        q_out = mp.Queue(self.max_queue_size)
        self._q_in_out.append(q_out)
        if cpus is None:
            cpus = [None]
        n_cpus = psutil.cpu_count(logical=True)

        for cpu in cpus:
            if cpu is None:
                print('adding modelet %s' % modelet.__name__)
                cc = None
            else:
                assert 0 <= cpu < n_cpus
                print('adding modelet %s at CPU %d' % (modelet.__name__, cpu))
                cc = [cpu]
            self._modelets.append(
                mp.Process(
                    target=modelet.run,
                    name=f'modelet-{cpu}',
                    kwargs={
                        'q_in': q_in,
                        'q_out': q_out, 
                        'q_err': self._q_err,
                        'cpus': cc,
                        **init_kwargs,
                    },
                )
            )
```

The parameter `modelet` is the *class object* of a subclass of `Modelet`.
Its *classmethod* `run` is the target function for `multiprocessing` to call in a new process.
This target takes an input queue, an output queue, and an error queue.
The input queue is the output queue of the last modelet;
if this is the first modelet, then the input queue is the very initial queue that is populated by client requests via calls to `ModelService.predict`.
A new queue is created to act as the output queue for this new modelet.
The queue is appended to `_q_in_out` and will be the input queue for the next modelet, if one is to be added; otherwise it will be the final output queue from which `_gather_results` will collect results.
The error queue is a shared one between all modelets and workers; it is for output only.

In addition, the parameter `cpus` specifies the CPU core(s) on which this modelet should run.
One worker process is created for each element of `cpus`.

Note that `add_modelet` does not *start* the worker process; it only sets it up. Starting the workers, and the service at large, is the job of `start`:

```python
    def start(self):
        assert self._modelets
        assert not self._started
        for m in self._modelets:
            m.start()
        self._t_gather_results = asyncio.create_task(self._gather_results())
        self._started = True

    def stop(self):
        if not self._started:
            return
        if self._t_gather_results is not None and not self._t_gather_results.done():
            self._t_gather_results.cancel()
            self._t_gather_results = None
        for m in self._modelets:
            if m.is_alive():
                m.terminate()
                m.join()
        self._started = False

        # Reset CPU affinity.
        psutil.Process().cpu_affinity(cpus=[])

    def __del__(self):
        self.stop()
```

The work of `start` and `stop` is straightforward.

Besides starting the worker processes, `start` also starts the async method `_gather_results` as a background task. We need to design `_gather_results` in coordination with `predict`.

```python
    async def predict(self, x):
        fut = asyncio.get_running_loop().create_future()
        uid = id(fut)
        self._uid_to_futures[uid] = fut
        q_in = self._q_in_out[0]
        while q_in.full():
            await asyncio.sleep(0.0078)
        while True:
            try:
                q_in.put_nowait((uid, x))
                break
            except queue.Full:
                await asyncio.sleep(0.0078)
        return await fut

    async def _gather_results(self):
        q_out = self._q_in_out[-1]
        q_err = self._q_err
        futures = self._uid_to_futures
        while True:
            if not q_out.empty():
                uid, y = q_out.get()
                fut = futures.pop(uid)
                fut.set_result(y)
                await asyncio.sleep(0)
                continue

            if not q_err.empty():
                uid, err = q_err.get()
                fut = futures.pop(uid)
                fut.set_exception(err)
                await asyncio.sleep(0)
                continue

            await asyncio.sleep(0.0089)
```

A central concern to both `predict` and `_gather_results` is
how to pair up a result (collected by `_gather_results`)
with the corresponding request (received by `predict`).

The key of the mechanism is to use an `asyncio.Future` object.
One `Future` object is created per request, i.e., per call to `predict`.
The request then `await`s on the `Future` object until it has a result attached.

This "attaching" happens in `_gather_results`.
In order to know which `Future` a result should be attached to,
we create an ID for each `Future` object, and let the ID accompany the data/result of this request throughout the queues.
In particular,
the pair `(future_id, request_data)` is placed in the very first queue.
Subsequent queues know to pass on the first element of the tuple and process the second element.

Both `predict` and `_gather_results` have some logic about waiting on a queue.
In `predict`, when it tries to put the input data in the first queue, the queue could be full, in which case it needs to wait.
In `_gather_results`, when it tries to take a result out of the final output queue (or an `Exception` object out of the error queue), the queue could be empty, in which case it needs to wait. 
The queue of valid results gets priority over the queue of exceptions.
As long as the result queue is not empty, its elements will be taken out one by one.
Only when the result queue is empty is the error queue checked.

## The workers

Let's recap the work of the coordinator.
Once it receives a client request, it places the request in the first queue, and waits on the final result queue and the exception queue for a valid result or exception info, whichever comes out. Then it returns the outcome to the correct requester.

The request data flows to the first modelet (any of its workers, whoever gets it), which places the result in the next queue. This first-stage result gets picked up by the second modelet, which processes it and places the result in the next queue,... and so on. You get the idea.

Flowing in the queues are always individual requests.
If a worker can benefit from vectorization, it can choose to *batch* requests and process them at once. This is the only tricky part in the worker code. Let's start with the non-tricky parts.

```python
class Modelet(metaclass=ABCMeta):
    def __init__(self, *, batch_size: int = None, batch_wait_time: float = None):
        # `batch_wait_time`: seconds, may be 0.
        self.batch_size = batch_size or 0
        self.batch_wait_time = 0 if batch_wait_time is None else batch_wait_time

    @classmethod
    def run(cls, *,
            q_in: mp.Queue, q_out: mp.Queue, q_err: mp.Queue,
            cpus: List[int] = None, **init_kwargs):
        if cpus:
            psutil.Process().cpu_affinity(cpus=cpus)
        modelet = cls(**init_kwargs)
        modelet.start(q_in=q_in, q_out=q_out, q_err=q_err)

    def start(self, *, q_in: mp.Queue, q_out: mp.Queue, q_err: mp.Queue):
        process_name = f'{self.__class__.__name__}--{mp.current_process().name}'
        print('%s started' % process_name)
        if self.batch_size > 1:
            self._start_batch(q_in=q_in, q_out=q_out, q_err=q_err)
        else:
            self._start_single(q_in=q_in, q_out=q_out, q_err=q_err)

    @abstractmethod
    def predict(self, x):
        # `x`: a single element if `self.batch_size == 0`;
        # else, a list of elements.
        # When `batch_size == 0`, hence `x` is a single element,
        # return corresponding result.
        # When `batch_size > 0`, return list of results
        # corresponding to elements in `x`.
        raise NotImplementedError

    def _start_single(self, *, q_in, q_out, q_err):
        batch_size = self.batch_size
        while True:
            uid, x = q_in.get()
            try:
                if batch_size:
                    y = self.predict([x])[0]
                else:
                    y = self.predict(x)
                q_out.put((uid, y))
            except Exception as e:
                q_err.put((uid, e))
```

`Modelet.run` is the target function for `multiprocessing`.
It creates a class object and kicks off an infinite loop by calling
the `start` method.

One thing to note in `_start_single` is how it takes care to pass forward the ID of the `Future` object that corresponds to the request.
When a modelet has multiple workers, they concurrently process requests.
By the end of the pipeline, there is no guarantee that the outputs come in the same order as the inputs that got placed into the first queue.
This is why we need to carry around this ID
in order to pair each result with its request.

The batching behavior is controlled by two parameters:
`batch_size` and `batch_wait_time`.
If there are plenty of requests waiting in the pipeline,
we'll collect `batch_size` number of elements and process the batch.
On the other hand, consider the case where requests arrive sporadically.
Imagine we have collected fewer than `batch_size` number of elements
and have waited a while for the next one, but it's not coming!
We need to stop the wait and process the "partial" batch.
How long we should wait before moving on is controlled by `batch_wait_time`.

As it turned out, this logic is not simple.

Suppose `batch_size` is 10 and `batch_wait_time` is 1 second.
How exactly do they work together?

Say we have collected 6 elements, and have been waiting for exactly 1 second; the next is yet to come. Should we move on processing the 6 elements? Yes.

What if we currently have an empty batch, and have waited for 1 second? Should we quit the wait? Of course not. We have to get at least one element. Then should we move on right away, since we have waited for, say, 2.3 seconds for this one?

Not so fast. Imagine a whole bunch have just arrived all of a sudden.
Although we have waited for a long time, now after the first one, if we keep collecting, we may get the desired `batch_size` count of elements without much further wait.

What if the second element needs a little wait, say, 0.1 second? Should we apply `batch_wait_time` here and move on once no more is coming within this wait time?

My design for this situation is, "no more wait". Since we have waited longer than `batch_wait_time` for the first one (as we have to), we'll keep collecting up to `batch_size` elements *if they are already here*. That is, a wait time of 0 is used if the first element has taken longer than `batch_wait_time`.

Let's see the code.

```python
    def _start_batch(self, *, q_in, q_out, q_err):
        batch_size = self.batch_size
        batch_wait_time = self.batch_wait_time
        while True:
            batch = []
            uids = []
            n = 0
            if batch_wait_time == 0:
                uid, x = q_in.get()
                batch.append(x)
                uids.append(uid)
                n += 1
                while n < batch_size:
                    try:
                        uid, x = q_in.get_nowait()
                        batch.append(x)
                        uids.append(x)
                        n += 1
                    except queue.Empty:
                        break
            else:
                try:
                    uid, x = q_in.get(timeout=batch_wait_time)
                    batch.append(x)
                    uids.append(uid)
                    n += 1
                except queue.Empty:
                    uid, x = q_in.get()
                    batch.append(x)
                    uids.append(uid)
                    n += 1
                    while n < batch_size:
                        try:
                            uid, x = q_in.get_nowait()
                            batch.append(x)
                            uids.append(uid)
                            n += 1
                        except queue.Empty:
                            break
                else:
                    while n < batch_size:
                        try:
                            uid, x = q_in.get(timeout=batch_wait_time)
                            batch.append(x)
                            uids.append(x)
                            n += 1
                        except queue.Empty:
                            break

            try:
                results = self.predict(batch)
            except Exception as e:
                for uid in uids:
                    q_err.put((uid, e))
            else:
                for uid, y in zip(uids, results):
                    q_out.put((uid, y))
```

In the branch for `batch_wait_time > 0`, we first wait for up to this duration.
If the first element arrives within this duration, we continue to collect subsequent elements with this wait time.
If the first element does not arrive within this duration, we then wait for the first element for as long as it takes; after that we collect subsequent elements without waiting.

Also notice that, although predictions are made on a batch, we place individual results in the output queue.

By the way, this is the absolute first time I have ever used the `try ... except ... else` syntax.
It just occurred unconsciously.

There is a second thought on the "wait time".
The definition adopted here is the time between elements.
Imagine the situation where the arrival times of elements are spread out just slightly below `batch_wait_time` in between.
The total time spent on collecting the batch is up to `batch_wait_time` times `batch_size`.
In other words, the first element may wait up to this long before being processed.
That might be too long.

An alternative definition of `batch_wait_time` is the accumulative wait time for the whole batch. If we implement this definition, then any element will wait for no longer than `batch_wait_time` before being processed. I feel this would be a better approach.


## Example

Just to show the usage and test the code, we made up a simple example:

```python
class Scale(Modelet):
    def predict(self, x):
        return x * 2


class Shift(Modelet):
    def predict(self, x):
        return x + 3


async def test_service():
    service = ModelService(cpus=[0])
    service.add_modelet(Scale, cpus=[1,2])
    service.add_modelet(Shift, cpus=[3])
    service.start()

    z = await service.predict(3)
    assert z == 3 * 2 + 3

    x = list(range(10))
    tasks = [service.predict(v) for v in x]
    y = await asyncio.gather(*tasks)
    assert y == [v * 2 + 3 for v in x]

    service.stop()

asyncio.run(test_service())
```

It worked.
