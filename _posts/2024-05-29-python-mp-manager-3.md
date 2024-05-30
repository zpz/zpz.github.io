---
layout: post
title: "Reading and Hacking Python's multiprocessing.managers: Part 3"
excerpt_separator: <!--excerpt-->
tags: [Python]
---

In [Part 1](https://zpz.github.io/blog/python-mp-manager-1/) 
and [Part 2](https://zpz.github.io/blog/python-mp-manager-2/)
of this series, we have gained basic understanding of the standard module `multiprocessing.managers` and fixed a few bugs in it.
In this article, we will move on to make some enhancements to it.
The complete code of these enhancements is available in
the [module mpservice.multiprocessing.server_process in the package mpservice](https://github.com/zpz/mpservice).<!--excerpt-->


## Server-side proxies

In the standard module, there are two situations where a new object is created in the server and is represented by a proxy:

1. When user registers a class with `BaseManager` and specifies `create_method=True` (which is the default), a "creation method" is added to the `BaseManager` class. Later when the user calls this method on a manager object, the manager calls `Server.create` of the server, which creates an instance of the registered class, keeps it in the server, and returns relevant info to the manager. Subsequently, the manager constructs a proxy for the server-side object and returns it to the user.
2. If a certain method of a registered class appears in the `method_to_typeid` of the registry, it indicates that calling this method on a server-side instance of said class (via a proxy in a client process) will not return the output of the method "as is" to the user; instead, the output will stay in the server (possibly after some processing, depending on the info in `method_to_typeid`), and relevant info will be returned to the caller in the client process. Eventually, the user will get a proxy. This process involves `BaseProxy._callmethod`, `Server.serve_client`, and `Server.create`.

The standard module also allows "nested proxies" in this sense: method calls on a proxy object may take another proxy object as argument. For example, say we have a dict proxy `mydict` and a list proxy `mylist`, we can do `mylist.append(mydict)` or `mydict['x'] = mylist`. The effect of such calls is that a proxy object gets passed into the server process (through pickling/unpickling) and added to a server-side object. In other words, a proxied object (not a proxy object!) now contains another proxy object.

In [Part 2](https://zpz.github.io/blog/python-mp-manager-2/), we have hardened the implementation of pickling and unpickling of proxy objects. With that (along with another fix on nested proxies), the "nested proxy" above is transparent to the server code---the proxy object is treated exactly like any other value when it enters or leaves the server process.

In the two situations above, a server-side object is created in the server, followed by a proxy object created in the client process.
We can imagine scenarios where both a server-side object and its proxy are created in the server. Consider the following scenario.

We have registered a class `MyClass`. Among others, this class has a method called `mymethod`. This method creates a list called `mylist`,
which contains a number of elements, including one that is a dict. However, this is not a "regular" dict---after the dict, called `mydict`, is created, it is not "directly" placed in `mylist`; instead, this dict is managed by the server as a "proxied" object, and a proxy to it is placed in `mylist`. In the end, `mylist` may look like this: `[28, ('a', 'b'), <proxy_to_mydict>, "123 Main Street, New York City"]`.
Keep in mind that `mymethod` happens in the server process.

As to the return of `mymethod` or what the client receives, there are two situations. In the first situation, the method returns a proxy to the server-side `mylist`. In this return, individual elements of `mylist` are undisturbed. Only meta info about `mylist` is returned, based on which a proxy object is constructed. When user accesses the third element of `mylist` via its proxy, `<proxy_to_mydict>` is returned to the user (through pickling/unpickling). From that point on, the proxy to `mydict` can be used, independent of the proxy to `mylist`, to manipulate the server-side `mydict`.

In the second situation, the method returns `mylist` as a "regular" value, that is, the list is pickled, passed out of the server process, and unpickled. The user gets a regularly-looking list object, `[28, ('a', 'b'), <proxy_to_mydict>, "123 Main Street, New York City"]`. The object `mylist` in the server is gone. The list elements are plain values now with no ties to server, except for the third element,
which is a proxy object to the server-side `mydict`. User can use this proxy to manipulate `mydict` as expected.

To summarize, the two situations are: (1) method returns a proxy; the proxied object contains another proxy; (2) method returns a regular object that contains a proxy somewhere in it.
You can think up real world applications where this can be useful.
There is one problem, though, and that is, *the standard module does not have a way to create this kind of `mylist` in the server*.

What is needed is a way to express, in server-side code: "this object is replaced by a proxy to it, and the real thing is managed by the server separately". We create a function called `managed` for this purpose. The gist of it is listed below.


```python

def get_server(address=None):
    """
    If `None`, the caller is not running in the manager server process.
    Otherwise, the return is the `Server` object that is running in the current process.
    """
    server = getattr(current_process(), '_manager_server', None)
    if server is None:
        return None
    if address is None or server.address == address:
        return server
    return None


def managed(obj, *, typeid: str):
    """
    This function is used within a server process.
    It creates a proxy for the input `obj`.
    The resultant proxy can be sent back to a requester outside of the server process.
    The proxy may also be saved in another data structure (either residing in the server or sent back to
    the client), e.g. as an element in a managed list or dict.
    """
    server = get_server()
    if not server:
        return obj

    proxy = server.create(None, typeid, obj)

    return proxy
```

This looks quite simple, but it relies on other changes. Apparently, we have made changes to `Server.create` to make it return a proxy object.

```python
class Server(multiprocessing.managers.Server):

    def create(self, c, typeid, /, *args, **kwargs):
        ident, exposed = super().create(c, typeid, *args, **kwargs)
        # This has called `incref` once

        token = Token(typeid, self.address, ident)
        proxytype = self.registry[typeid][-1]

        proxy = proxytype(
            token,
            serializer=self.serializer,
            authkey=self.authkey,
            exposed=exposed,
        )
        # This has called `incref` once

        self.decref(None, ident)

        return proxy
```

You may wonder that this change to the `Server.create` API disrupts some other code. Yes, it does, and we have made other changes not shown here. Let's review all the scenarios where a proxy object is created.

1. User calls a "creation method" on the manager. The server creates a server-side object and returns meta info. The "creation method" constructs a proxy object.
2. User calls `BaseProxy._callmethod` for a "proxied" method that appears in `method_to_typeid`. The server calls the method on the target object, creates a server-side object and returns meta info. `BaseProxy._callmethod` constructs a proxy object.
3. User code of a registered class uses `managed` to create a server-side object and its proxy; this happens all in the server.
4. Outside of the server process, a proxy object is pickled and passed to another process, in which a new proxy object is created out of unpickling.


The fourth case does not create a new server-side object and does not raise any question.
The first two cases can be implemented, using `managed`, by constructing a proxy in the server and returning it, as opposed to returning meta info and constructing a proxy outside the server.
Ultimately, the first three cases are all supported by the changed `Server.create`, which returns a proxy for a server-side object that has just been created.

We have learned that `method_to_typeid` in the registry of a class indicates methods of the class that will return proxies rather than "regular" values.
This mechanism is no longer necessary, as we can wrap the method's output by `managed` to achieve the same effect.
This is how the second situation could be re-implemented as hinted above.

There is one scenario where the use of `method_to_typeid` may be preferred. In this scenario, the code of the registered class has no considerations for its being used in a server process.
When it is used in a server process, possibly by a different author in different projects, `managed` would require changes to the class code, whereas `method_to_typeid` would leave the class code intact.

We'll demonstrate the use of `managed` in a later section. Now on to another enhancement.


## Shared memory


The standard module `multiprocessing.shared_memory` provides class `SharedMemory` that enables a user to allocate a block of memory and access it from multiple processes. The allocated memory is not "managed" by the `SharedMemory` instance. The instance merely remembers the unique *name* of the memory block, and provides ways to communicate with the "system" about the named memory block.
In particular, `SharedMemory` has methods for the user to declare that the memory is no longer needed anywhere hence should be released.
*The user must explicitly call these methods.*
The reason is that the demise of a `SharedMemory` instance in any process contains no info about whether the memory is still used by any other process.
This is like the user has to manage the "ref count" of the memory block manually.
It is clearly an inconvenience.

The standard module `multiprocessing.managers` goes one step further with a subclass of `BaseManager` called `SharedMemoryManager` that helps manage shared memory.
All shared memory blocks created by this facility are released when the manager (and server) reaches end of life, hence the user does not need to manually release them. However, the timing of the release leaves much to be desired: it is only better than never, as it is quite late.
This is as if Python did not garbage collect any object ever created in a program until the program terminates.

Of course, Python does better than that. It destroys objects as soon as it can tell the object is no longer used, regardless of how long the program will continue.
We would like shared memory to work the same way: a block is released as soon as Python can tell it is no longer used, regardless of how long the manager/server will continue.

Our solution achieves this goal in two steps. First, we define a class called `MemoryBlock` that controls the lifetime of a shared memory block.
In contrast to a `SharedMemory` instance, which communicates with "the system" about a memory block that is managed by the latter,
a `MemoryBlock` instance in effect *hosts* a memory block and makes it live/die along with its host.
The gist of the class definition is as follows.

```python
class MemoryBlock:

    def __init__(self, size: int):
        assert size > 0
        self._mem = SharedMemory(create=True, size=size)
        self.release = multiprocessing.util.Finalize(self, type(self)._release, args=(self._mem, ), exitpriority=10)

    def _name(self):
        # This is for use by ``MemoryBlockProxy``.
        return self._mem.name

    @property
    def name(self):
        return self._mem.name

    @property
    def size(self):
        return self._mem.size

    @property
    def buf(self):
        return self._mem.buf

    @staticmethod
    def _release(mem):
        mem.close()
        mem.unlink()

    def __del__(self):
        """
        Release the shared memory when this object is garbage collected.
        """
        self.release()
```

A new memory block is allocated in `__init__` and released when the `MemoryBlock` instance is garbage collected, guaranteed by `self.release`.
(See [this block post](https://zpz.github.io/blog/guaranteed-finalization-without-context-manager/) about `Finalize`.)
From now on, we will use `MemoryBlock` for shared memory and never reach for the raw `SharedMemory`. The lifetime of the memory block is taken care of by Python's garbage collector, just like any other object.

This solves the "auto release" problem; we have yet to address how to "share" it across processes.
For effective and flexible sharing, we need a `MemoryBlock` instance to live as long as needed while being accessed from multiple processes,
while the processes may not coordinate between themselves (such as using one of the synchronization mechanisms).
This is squarely where a "manager" shines.

So, the second step of the solution requires that `MemoryBlock` instances always reside in the server process of a manager, and are accessed from other processes via proxies. We know that the server-side `MemoryBlock` object will live as long as there is one or more proxy objects out there targeting it, and this does not require any coordination between the proxies. The proxy class is defined as follows.


```python
class MemoryBlockProxy(BaseProxy):

    def __init__(self, *args, **kwargs):
        super().__init__(*args, **kwargs)
        self._name = None
        self._size = None
        self._mem = None

    @property
    def name(self):
        if self._name is None:
            self._name = self._callmethod('_name')
        return self._name

    @property
    def size(self) -> int:
        if self._mem is None:
            self._mem = SharedMemory(name=self.name, create=False)
        return self._mem.size

    @property
    def buf(self) -> memoryview:
        if self._mem is None:
            self._mem = SharedMemory(name=self.name, create=False)
        return self._mem.buf
```

Finally, let's hook it up by registration:


```python
class Server(multiprocessing.managers.Server):
    ...


class ServerProcess(multiprocessing.managers.BaseManager):
    _Server = Server
    ...


ServerProcess.register(
    'MemoryBlock', 
    callable=MemoryBlock, 
    proxytype=MemoryBlockProxy, 
    create_method=True
)

ServerProcess.register(
    'ManagedMemoryBlock',
    callable=None,
    proxytype=MemoryBlockProxy,
    create_method=False,
)

managed_memoryblock = functools.partial(managed, typeid='ManagedMemoryBlock')
```

In the example below, a shared memory block is created via the manager and is used in multiple processes.


```python
import time
from pprint import pprint

from mpservice.multiprocessing import Process, Queue
from mpservice.multiprocessing.server_process import ServerProcess


def inc_worker(q):
    time.sleep(1)
    mem = q.get()
    assert mem.size == 10
    buf = mem.buf
    assert len(buf) == 10
    assert buf[4] == 100
    buf[4] += 1


def main():
    with ServerProcess() as manager:
        mem = manager.MemoryBlock(10)
        assert type(mem.buf) is memoryview
        assert len(mem.buf) == 10
        mem.buf[4] = 100

        print()
        pprint(manager._debug_info())
        print()
        # Shows ref count 1.
    
        q = Queue()
        q.put(mem)
        time.sleep(0.01)
        # Wait to make sure ref-count manipulation in pickling is done
        # hence reflected below.

        print()
        pprint(manager._debug_info())
        print()
        # Shows ref count 2.

        p = Process(target=inc_worker, args=(q,))
        p.start()

        print()
        pprint(manager._debug_info())
        print()
        # Shows ref count 2.

        p.join()

        assert mem.buf[4] == 101

        print()
        pprint(manager._debug_info())
        print()
        # Shows ref count 1.


if __name__ == '__main__':
    main()
```

Running this script got ths following:

```sh
$ python3 test_sharedmem.py 

[{'id': '7f6e2d537850',
  'preview': "<MemoryBlock 'psm_a005552c', size 10>",
  'refcount:': 1,
  'type': 'MemoryBlock'}]


[{'id': '7f6e2d537850',
  'preview': "<MemoryBlock 'psm_a005552c', size 10>",
  'refcount:': 2,
  'type': 'MemoryBlock'}]


[{'id': '7f6e2d537850',
  'preview': "<MemoryBlock 'psm_a005552c', size 10>",
  'refcount:': 2,
  'type': 'MemoryBlock'}]


[{'id': '7f6e2d537850',
  'preview': "<MemoryBlock 'psm_a005552c', size 10>",
  'refcount:': 1,
  'type': 'MemoryBlock'}]
```

## A fancy example

Time to show a *fancy* example!

```python
from pprint import pprint

from mpservice.multiprocessing import Process
from mpservice.multiprocessing.server_process import (
    MemoryBlock,
    ServerProcess,
    managed_list,
    managed_memoryblock,
)


class MemoryWorker:
    def memory_block(self, size):
        return size

    def make_dict(self, size):
        mem = MemoryBlock(size)
        mem.buf[3] = 26

        return {
            'size': size,
            'block': managed_memoryblock(mem),
            'list': managed_list([1, 2]),
            'tuple': ('first', managed_list([1, 2]), 'third'),
        }

    def make_list(self):
        return [
            managed_memoryblock(MemoryBlock(10)),
            {'a': 3, 'b': managed_list([1, 2])},
            managed_memoryblock(MemoryBlock(20)),
        ]


def worker_mem(data):
    data.buf[10] = 10


def worker_dict(data, size):
    assert data['size'] == size
    mem = data['block']
    assert mem.size == size
    assert mem.buf[3] == 26
    mem.buf[3] = 62
    data['tuple'][1].append(30)
    assert len(data['tuple'][1]) == 3


def worker_list(data):
    data[0].buf[5] = 27
    data[2].buf[8] = 88
    data[1]['b'].remove(2)


def main():
    ServerProcess.register(
        'MemoryWorker', MemoryWorker, method_to_typeid={'memory_block': 'MemoryBlock'}
    )
    with ServerProcess() as server:
        worker = server.MemoryWorker()
        m = worker.memory_block(20)

        print('m:')
        print(m)

        print()
        print('server-side ref counts:')
        pprint(server._debug_info())
        print()

        assert m.buf[10] == 0
        p = Process(target=worker_mem, args=(m,))
        p.start()
        p.join()
        assert m.buf[10] == 10

        data = worker.make_dict(64)
        assert data['size'] == 64
        assert data['block'].size == 64
        assert data['block'].buf[3] == 26

        print()
        print("data['tuple'][1]:")
        print('  value:', data['tuple'][1])
        print('  type:', type(data['tuple'][1]))
        print()

        assert len(data['tuple'][1]) == 2
        p = Process(target=worker_dict, args=(data, 64))
        p.start()
        p.join()
        assert data['block'].buf[3] == 62
        assert len(data['tuple'][1]) == 3
        assert data['tuple'][1][2] == 30

        data = worker.make_list()
        p = Process(target=worker_list, args=(data,))
        p.start()
        p.join()
        assert data[0].buf[5] == 27
        assert data[2].buf[8] == 88
        assert len(data[1]['b']) == 1
        assert data[1]['b'][0] == 1


if __name__ == '__main__':
    main()
```

Here's the printout:

```sh
$ python3 test_managed.py 
m:
<MemoryBlockProxy 'psm_c3f7af00' at 7ff7c9b5fa00, size 20>

server-side ref counts:
[{'id': '7ff7c9b5fb80',
  'preview': '<__mp_main__.MemoryWorker object at 0x7ff7c9b5fb80>',
  'refcount:': 1,
  'type': 'MemoryWorker'},
 {'id': '7ff7c9b5fa00',
  'preview': "<MemoryBlock 'psm_c3f7af00', size 20>",
  'refcount:': 1,
  'type': 'MemoryBlock'}]


data['tuple'][1]:
  value: [1, 2]
  type: <class 'mpservice.multiprocessing.server_process.ListProxy'>
```