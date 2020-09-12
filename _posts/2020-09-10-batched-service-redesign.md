---
layout: post
title: "Service Batching from Scratch, Again"
excerpt_separator: <!--excerpt-->
tags: [python]
---

In [a previous post]({{ site.baseurl }}/blog/batched-service/), I described an approach to serving machine learning models with built-in "batching". The design has been used in real work with good results. Over time, however, I have felt some pains and learned some new tricks. Finally, I sat down and designed a new approach. Note, this is **not** a revision to the previous approach. This is a new design from scratch. <!--excerpt-->

First, let's re-visit the main points of the previous design:

1. It is assumed that the model is decomposed into three sequential components: (1) a `Preprocessor`; (2) a `VectorTransformer`; (3) a `Postprocessor`.
2. Both `Preprocessor` and `Postprocessor` are light weight. They run in the main process. The `VectorTransformer` is the meat of processing, and runs in its own process (and it is free to launch other processes from there).
3. Data flows through the components via `multiprocessing` `Queue`s.
4. The service API supports a **singleton** endpoint and a **batch** endpoint. Queries into the first are automatically batched to take advantage of vectorized computation in the `VectorTransformer`. In contrast, the latter is an "express lane", meaning batches (large or small) are processed as they arrive, skipping batching.
5. The "scheduler" in the main process uses `asyncio`.
 In particular, `asyncio` `Future`s are used to guarantee that, after all the waiting, batching, asynchorous flow and processing in multiple stages, a result is returned to the correct request that has been waiting for it.

Some limitations that I have observed in the design include

- The assumed three-component structure, as well as the assumption that they are light-heavy-light, are rigid. It's totally reasonable to have a pipeline with more, or less, than three components, and to have more than one heavy component.
- The batch endpoint adds to the complexity of the implementation and the usage. Given that the singleton endpoint does batching behind the scene, if this is pushed to the limit of efficiency, then removing the batch endpoint would bring nice simplification without much loss in capability.
- The design does not help the user to make full use of CPU cores. The `VectorTransformer` runs in its own process, and it in turn is free to do multiprocessing. However, that is some demand on the user's programming skill.
- The `Preprocessor` and `Postprocesser`, sharing the main process with the async scheduler, are better async. That again is some demand on the user's programming skill. Moreover, that is likely unbeneficial if these processors do not involve I/O.

To overcome these flaws, the new design has some very different ideas on the high level. For ease of explanation, I will refer to this design the "framework".

- It supports a sequence of components, however many, with no assumption on their roles and relative expenses.
- Each component is "traditional" synchronous code within a single process. However, one component can use multiple processes as independent "workers". This will be managed by the framework.
- The "scheduler" and the components run in separate processes. The number of processes is one (the scheduler) plus the number of workers (each component uses one or more workers). Each process may be instructed to run on a specific CPU core.
- Data flows through the processes in `multiprocessing` `Queue`s.
- There is another `Queue` for errors. If error occurs in any process, the `Exception` object is short-circuited back to the scheduler in this queue. Only valid result will proceed to the next component, as valid input. The exception object will be paired up with the awaiting request, just like a valid result is.
- There is only a singleton endpoint. No batch endpoint.
- Each component can decide to take batches or singletons as input. In the former case, the framework assembles singletons (which are flowing in the queue) into a batch, and feed it into the component. The component should output a list of results corresponding to the singletons in the batch input. The framework dissembles the batch output to singleton results, and places them in the queue.

This is a lot of power in abstract! Now let's get concrete.


## The scheduler

The entrypoint, or scheduler, of the whole thing is a class called `ModelService`. It manages workers, which are concrete subclasses of `Modelet`. Before describing `ModelService`, let's get a rough picture of `Modelet`.

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
                place_y_in_q_out()
            except Exception as e:
                place_e_in_q_err()

    @classmethod
    def run(cls, q_in, q_out, q_err, ...):
        modelet = cls(...)
        modelet.start(q_in, q_out, q_err)
```

The workflow for a `Modelet` is quite clear in this sketch. We'll fill in details later.

A sketch for `ModelService` is as follows.

```python
import asyncio
import logging
import multiprocessing as mp
import queue
from typing import Type, List

import psutil

logger = logging.getLogger(__name__)


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

    def stop(self):
        ...

    async def _gather_results(self):
        ...

    async def predict(self, x):
        ...
        place_x_in_queue()
        wait_for_result()
        return_result()
        ...
```

On a `ModelService` object, we call `add_modelet` to add one "component" at a time. Once all components have been added, we call `start` to start the service. Besides starting the worker processes, `_gather_results` is started at this time as a background job. This job picks up results at the end of the queue and pair it up with the awaiting request.

Requests are received by `predict`. In this method, the input is placed in the queue. Then it waits for the result. The input will flow through the workers. In the end, the result will be picked up by `_gather_results`. Once getting the result, `predict` returns. Note that both `_gather_results` and `predict` are async functions.

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

The first queue in `ModelService._q_in_out` will take input data that is received by `ModelService.a_predict`. The worker(s) of the first component will take input from this queue and process it, and put the result in the second queue in `_q_in_out`---which will be added when the first modelet is added.

The parameter `cpus` is a list of integers specifying on which CPU cores this "main" process should be running. If not specified, the execution may move between all available cores. For efficiency, it is recommended to specify a core. This is known as "CPU pinning".

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
                logger.info('adding modelet %s', modelet.__name__)
                cc = None
            else:
                assert 0 <= cpu < n_cpus
                logger.info('adding modelet %s at CPU %d', modelet.__name__, cpu)
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

When we add a modelet, it takes input from the last queue in `_q_in_out`. A new queue is created to receive its output.
This queue is appended to `_q_in_out`.
It will be the input queue for the next modelet.
If there is no more modelets, it will be the final output queue from which `_gather_results` collects results.

The parameter `cpus` specifies the CPU core(s) on which this modelet should run.
One worker process is created for each element of `cpus`.
Note that `cpus` may contain repeat values, meaning the workers do not have to all run on distinct cores.

We'll get to `_gather_results` in a moment. At this point, the work needed to start and stop the service is pretty clear:

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
        for m in self._modelets:
            m.terminate()
            m.join()
        self._started = False
```

Now the job of `_gather_results` becomes straightforward.

```python
    async def _gather_results(self):
        q_out = self._q_in_out[-1]
        q_err = self._q_err
        futures = self._uid_to_futures
        while True:
            if not q_out.empty():
                uid, y = q_out.get()
                fut = futures.pop(uid)
                if not fut.cancelled():
                    fut.set_result(y)
                else:
                    logger.warning('Future object is already cancelled')
                continue

            if not q_err.empty():
                uid, err = q_err.get()
                fut = futures.pop(uid)
                fut.set_exception(err)
                continue

            await asyncio.sleep(0.0089)
```

Now we can implement `_gather_results` and `predict`.
The former needs to know how to pair up a result with the corresponding request,
and that mechanism is of concern to the latter, of course.

The key of the mechanism is to use an `asyncio.Future` object.
One `Future` object is created per request, i.e., per call to `predict`.
The request then `await`s on the `Future` object until it has a result attached.

Results are attached to `Future` objects in `_gather_results`.
In order to know which is attached to which, a uuid for each `Future` object is accompanies its input data.
The pair of `(future_id, input_data)` flows through the queues.
For simplicity, we take `id(future_object)` as the uuid.

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
                if not fut.cancelled():
                    fut.set_result(y)
                else:
                    logger.warning('Future object is already cancelled')
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

Both `predict` and `_gather_results` has some logic about waiting on a queue.
In `predict`, when it tries to put the input data in the first queue, the queue could be full, in which case it needs to wait.
In `_gather_results`, when it tries to take a result out of the final output queue (or an `Exception` object out of the error queue), the queue could be empty, in which case it needs to wait. 

The output queue gets priority over the error queue.
As long as the output queue is not empty, its elements will be taken out one by one.
Only when the output queue is empty is the error queue checked.

## The workers

`Modelet.run` is the target function for `multiprocessing`.
It's a simple method, so we'll start with it.

```python
class Modelet(metaclass=ABCMeta):

    @classmethod
    def run(cls, *,
            q_in: mp.Queue, q_out: mp.Queue, q_err: mp.Queue,
            cpus: List[int] = None, **init_kwargs):
        if cpus:
            psutil.Process().cpu_affinity(cpus=cpus)
        modelet = cls(**init_kwargs)
        modelet.start(q_in=q_in, q_out=q_out, q_err=q_err)
```

```python
    def __init__(self, *, batch_size: int = None, batch_wait_time: float = None):
        # `batch_wait_time`: seconds, may be 0.
        self.batch_size = batch_size or 0
        self.batch_wait_time = 0 if batch_wait_time is None else batch_wait_time

    def start(self, *, q_in: mp.Queue, q_out: mp.Queue, q_err: mp.Queue):
        process_name = f'{self.__class__.__name__}--{mp.current_process().name}'
        logger.info('%s started', process_name)
        if self.batch_size > 1:
            self._start_batch(q_in=q_in, q_out=q_out, q_err=q_err)
        else:
            self.__start_single(q_in=q_in, q_out=q_out, q_err=q_err)
```

```python
    @abstractmethod
    def predict(self, x):
        # `x`: a single element if `self.batch_size == 0`;
        # else, a list of elements.
        # When `batch_size == 0`, hence `x` is a single element,
        # return corresponding result.
        # When `batch_size > 0`, return list of results
        # corresponding to elements in `x`.
        raise NotImplementedError
```

```python
    def __start_single(self, *, q_in, q_out, q_err):
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
                logger.info(e)
                # There are opportunities to print traceback
                # and details later using the `SubProcessError`
                # object. Be brief on the logging here.
                err = SubProcessError(e)
                q_err.put((uid, err))
```
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
                logger.info(e)
                err = SubProcessError(e)
                for uid in uids:
                    q_err.put((uid, err))
            else:
                for uid, y in zip(uids, results):
                    q_out.put((uid, y))
```

### batching

## Example
