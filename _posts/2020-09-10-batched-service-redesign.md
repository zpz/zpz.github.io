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


## ModelService

## Modelet

### batching

## Example
