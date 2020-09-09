---
layout: post
title: "Service Batching from Scratch, Again"
excerpt_separator: <!--excerpt-->
tags: [python]
---

In [a previous post]({{ site.baseurl }}/blog/batched-service/), I described an approach to serving machine learning models with built-in "batching". The design has been used in real work with good results. Over time, however, I have felt some pains and learned some new tricks. Finally, I sat down and designed a new approach. Note, this is **not** a revision to the previous approach. This is a new design from scratch. <!--excerpt-->

Still, let's re-visit the main points of the previous design:

1. It is assumed that the model is decomposed into three sequential components: (1) a `Preprocessor`; (2) a `VectorTransformer`; (3) a `Postprocessor`.
2. Both `Preprocessor` and `Postprocessor` are light weight. They run in the main process. The `VectorTransformer` is the meat of processing, and runs in its own process (and it is free to launch other processes from there).
3. Data flows through the components via `multiprocessing` `Queue`s.
4. The service API supports a "single item" endpoint and a "batch" endpoint. Queries into the first are automatically batched to take advantage of vectorized computation in the `VectorTransformer`. In contrast, the latter is an "express lane", meaning batches (large or small) are processed as they arrive, skipping batching.
5. The "coordinator" in the main process uses `asyncio`.
 In particular, `asyncio` `Future`s are used to guarantee that, after all the waiting, batching, asynchorous flow and processing in multiple stages, a result is returned to the correct request that has been waiting for it.

Some limitations that I have observed in the design include

-