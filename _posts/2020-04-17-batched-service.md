---
layout: post
title: "Service Batching from Scratch"
excerpt_separator: <!--excerpt-->
tags: [python]
---

TensorFlow Serving has "batching" capability because the model inference can be "vectorized".
This means that if you let it predict `y` for `x`, versus one hundred `y`s for one hundred `x`s, the latter may not be much slower than the former (definitely *much* better than one hundred times slower),
because the algorithm takes the batch and performs fast matrix computation, or even sends the job to GPU for very fast parallel computation.
The effect is that the batch may take somewhat longer than a single element,
but per-item time is *much* shorter, hence *throughput* is much higher.<!--excerpt-->

As a model service, this does not require the client to send a batch of `x` at a time.
Instead, the client sends individual items as usual.
The server collects these requests but does not do the real processing right away
until a certain amount has been accumulated or a pre-determined waiting-time is up.
At that point, the server processes the batch of input, gets a batch of results,
and distributes individual responses (out of the batch result) to the waiting requests.
Obviously, it has to make sure that things are in correct order, for example,
the second request should not get the result that is for the first request.

This is a common need in machine learning services.
What if TensorFlow Serving is not applicable for the particular situation?
In this post, we'll design such a server in Python using only the standard library.
Although we'll use machine-learning terminology like "model" and "inference",
the idea and code are general.

## Overall design considerations

We're going to develop a class `BatchedService` for this purpose.
The "core" vectorized model is a class named `VectorTransformer`.
A high-level design decision is to let `VectorTransformer` run in a separate process,
so that the main process concentrates on handling requests from the client,
including receiving, batching, preprocessing, dispatching results to waiting requests, and so on.
This multiprocess structure allows the logistics in the main process to happen in parallel to the "real work" of the `VectorTransformer`, which is compute-intensive.
(By the way I dislike the fashion of using "compute" as a noun, but there's nothing I can do about it...)

We may as well show the architecture up front and explain below.

![arch-1](../images/batched-service-1.png)

As individual requests come in, a "batcher" collects them and holds them in a buffer.
Once the buffer is full or waiting-time is over (even if the buffer is not yet full), the content of the buffer will be placed in a queue (`queue-batches`). The unit of the data in the queue is "batch", or basically a `list` of individual requests.
`Preprocessor` takes a batch at a time out of the queue, does whatever preprocessing it needs to do, and puts the preprocessed batch in a queue (`queue-to-worker`) that is going through the process boundary. The worker process takes one batch at a time out of this queue, makes predictions for this batch by the vectorized model, i.e. `VectorTransformer`, and puts the result in another queue (`queue-from-worker`). Back in the main process, a `Postprocessor` takes one result batch at a time off of this queue, does whatever postprocessing it needs to do, and critically, unpack the batch and distributes individual results to the individual requests, which have been waiting.

It's useful to highlight that

- In the "worker process", execution is sequential.
- In the main process, the parts before `queue-to-worker` and after `queue-from-worker` are concurrent.
- The work between `queue-batches` and `queue-to-worker` is sequential.
- The jobs of the `Batcher` and `Preprocessor` are concurrent. If preprocessing is anything expensive, collection of new requests should continue as one batch is being preprocessed.
- The `Preprocessor` put things in two queues: `queue-to-worker` is consumed by the worker process, whereas `queue-future-results` is consumed by `Postprocessor`.
- There must be a mechanism for `Postprocessor` to pair each individual result with its corresponding request. The diagram suggests that this is accomplished by `queue-future-results`. The "lollypop symbols" pulled out of the queue turn out to be objects of type `asyncio.Future`. The queue guarantees these `Future` objects come in order consistent with the elements in `queue-batches`, `queue-to-worker` and `queue-from-worker`. In the meantime, each original request also holds a reference of its corresponding `Future` object. We'll come back to this point later.

Now let's code up this design.




