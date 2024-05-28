---
layout: post
title: "Reading and Hacking Python's multiprocessing.managers: Part 3"
excerpt_separator: <!--excerpt-->
tags: [Python]
---

In [Part 1](https://zpz.github.io/blog/python-mp-manager-1/) 
and [Part 2](https://zpz.github.io/blog/python-mp-manager-2/)
of this series, we have gained basic understanding of the standard module `multiprocessing.managers` and fixed a few bugs in it.
In this article, we will focus on some enhancements to it that are implemented in
the [module mpservice.multiprocessing.server_process in the package mpservice](https://github.com/zpz/mpservice).<!--excerpt-->


## Server-side proxies

In the standard module, there are two situations where a new object is created in the server and is represented by a proxy:

1. When user registers a class with `BaseManager` and specifies `create_method=True` (which is the default), a "creation method" is added to the `BaseManager` class. Later when the user calls this method on a manager object, the manager calls `Server.create` of the server, which creates an instance of the registered class, keeps it in the server, and returns relevant info to the manager. Subsequently, the manager constructs a proxy object, pointing to the server-side object, and returns it to the user.
2. If a certain method of a registered class appears in the `method_to_typeid` of the registry, it indicates that calling this method on a server-side instance of said class will not return the output of the method directly to the user; instead, the output will stay in the server (possibly after some processing, depending on `method_to_typeid`), and relevant info will be returned to the caller in the client process. Eventually, the user will get a proxy. This process involves `BaseProxy._callmethod`, `Server.serve_client`, and `Server.create`.

The standard module also allows "nested proxies" in this sense: method calls on a proxy object may take another proxy object as argument. For example, say we have a dict proxy `mydict` and a list proxy `mylist`, we can do `mylist.append(mydict)` or `mydict['x'] = mylist`. The effect of such calls is that a proxy object gets passed into the server process (through pickling/unpickling) and added to a server-side object. In other words, a proxied object (not a proxy object!) now contains another proxy object.

In [Part 2](https://zpz.github.io/blog/python-mp-manager-2/), we have hardened the implementation of pickling and unpickling of proxy objects. With that (along with another fix on nested proxies), the "nested proxy" above is transparent to the server code---the proxy object is treated exactly like any other value when it enters or leaves the server process.

In the two situations above, a server-side object is created in the server, followed by a proxy object created in the client process.
We can imagine another scenario where both a server-side object and its proxy are created in the server. Consider this example:

We have registered a class `MyClass`. Among others, this class has a method called `mymethod`. This method eventually creates a list called `mylist`,
which contains a number of elements, including one that is a dict. However, this is not a "regular" dict---after the dict, called `mydict`, is created, it is not "directly" placed in `mylist`; instead, this dict is managed by the server as a "proxied" object, and a proxy to it is placed in `mylist`. Suppose `mylist` is `[28, ('a', 'b'), <proxy_to_mydict>, "123 Main Street, New York City"]`.

As to the return of `mymethod`, there are also two situations. In the first situation, the method returns a proxy to a server-side list (that is, `mylist`). In this case, during the call and return, individual elements of `mylist` are untouched. Only identification info about `mylist` is returned, based on which a proxy object is constructed. Only when user accesses the third element of the target list via its proxy, the proxy to `mydict` is returned to the user (through pickling/unpickling). From that point on, the proxy to `mydict` can be used, independently from the proxy to `mylist`, to manipulate the server-side `mydict`.

In the second situation, the method returns `mylist` as a "regular" value, that is, the list is pickled, passed out of the server process, and unpickled. The user gets a regularly-looking list object, `[28, ('a', 'b'), <proxy_to_mydict>, "123 Main Street, New York City"]`. The object `mylist` in the server is gone. The elements `28`, `('a', 'b')`, `"123 Main Street, New York City"` that the user has obtained are plain values no no ties to the server. Of course, there is one element that is special, and it is a proxy object to `mydict`, which is alive in the server. User can use this proxy to manipulate `mydict` as expected.

You can think up use cases for such proxies that are created in the server and "embedded" in another object. We will not be concerned about that here. 

There is one problem, though, and that is, the standard module does not have a way to create this kind of `mylist` in the server.

What is needed is a way to say: "this object is replaced by a proxy to it, and the real thing is managed by the server separately". We create a function called `managed` for this purpose. The gist of it is listed below.


```python

def get_server(address=None):
    # If `None`, the caller is not running in the manager server process.
    # Otherwise, the return is the `Server` object that is running in the current process.
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

You may wonder that this change to the `Server.create` API disrupts some other code. Yes, it does, and we have made other changes now shown here. Let's review all the scenarios where a proxy object is created.

1. User calls a "creation method" on the manager. The server creates a server-side object and returns meta info. The "creation method" constructs a proxy object and returns.
2. User calls `BaseProxy._callmethod` for a "proxied" method that appears in `method_to_typeid`. The server calls the method on the target object, creates a server-side object and returns meta info. `BaseProxy._callmethod` constructs a proxy object and returns.
3. User code of a registered class uses `managed` to create a server-side object and its proxy in the server.
4. Outside of the server process, a proxy object is pickled and passed to another process, in which a new proxy object is created out of unpickling.


The fourth case does not create a new server-side object and does not raise any question.
The first two cases can be implemented by constructing a proxy in the server and returning it, as opposed to returning relevant info and constructing a proxy outside the server.
Ultimately, the first three cases are all supported by the changed `Server.create`, which returns a proxy object.

We have learned that `method_to_typeid` in the registry of a class indicates methods of the class that will return proxies rather than "regular" values.
This mechanism is no longer necessary, as we can wrap the method's output by `managed` to achieve the same effect.
There is one situation that may prefer the use of `method_to_typeid`. In this situation, the class code has no consideration for its being used in a server process;
when it is used this way (possibly by a different author in different projects), `managed` would require changes to the class code (possibly by subclassing), whereas `method_to_typeid` could leave the class code intact.

TBD: make up an example showing flexible uses of `managed`.


## Shared memory


The standard module `multiprocessing.shared_memory` enables user to allocate a block of memory and access it from multiple processes.
There is a drawback or inconvenience of this facility, which is, user needs to take care of releasing a particular block of shared memory when they are sure it is no longer used.

The standard module `multiprocessing.managers` provides a subclass of `BaseManager` called `SharedMemoryManager` that helps manage "shared memory".
All shared memory blocks created by this facility are released at the end-of-life of the manager object, hence the user does not need to "manually" release them. However, there is a drawback: memory release happens only when the manger goes out of existence. If an application runs a long-lasting manager and creates shared memory anytime as needed, chances are many shared memory blocks will no longer be needed after some use. Since `SharedMemoryManger` releases shared memory only when the manger dies, user still has to manually release share memory if they want that to happen early.

We propose an alternative facility that has these characteristics:

- A shared memory block is used via its proxy; the memory is released whenever all its proxies are garbage collected.
- This facility does not take a `BaseManager` subclass that appears to be dedicated to management of shared memory. Instead, a shared memory block is a "server-side" object just like other server-side objects. The use of shared memory mixes naturally with other uses of the server process.



```python

    class MemoryBlock:
        """
        This class provides a simple tracker for a block of shared memory.

        User can use this class ONLY within a server process.
        They must wrap this object with `managed_memoryblock`.

        Outside of a server process, use `manager.MemoryBlock`.
        """

        def __init__(self, size: int):
            assert size > 0
            self._mem = SharedMemory(create=True, size=size)

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

        def _list_memory_blocks(self):
            """
            Return a list of the names of the shared memory blocks
            created and tracked by this class.
            This list includes the memory block created by the current
            ``MemoryBlock`` instance; in other words, the list
            includes ``self.name``.

            This is for use by ``MemoryBlockProxy``.
            This is mainly for debugging purposes.
            """
            server = get_server()
            blocks = []
            for z in server.id_to_obj.values():
                obj = z[0]
                if type(obj).__name__ == 'MemoryBlock':
                    blocks.append(obj.name)
            return blocks

        def __del__(self):
            """
            Release the shared memory when this object is garbage collected.
            """
            self._mem.close()
            self._mem.unlink()

        def __reduce__(self):
            # Block pickling to prevent user from accidentally using this class w/o
            # wrapping it by `managed_memoryblock`.
            raise NotImplementedError(
                f"cannot pickle instances of type '{self.__class__.__name__}"
            )

        def __repr__(self):
            return (
                f'<{self.__class__.__name__} {self._mem.name}, size {self._mem.size}>'
            )

    class MemoryBlockProxy(BaseProxy):
        def __init__(self, *args, name: str = None, size: int = None, **kwargs):
            super().__init__(*args, **kwargs)
            self._name = name
            self._size = size
            self._mem = None

        def __reduce__(self):
            func, args = super().__reduce__()
            kwds = args[-1]
            kwds['name'] = self._name
            kwds['size'] = self._size
            return func, args

        @property
        def name(self):
            """
            Return the name of the ``SharedMemory`` object.
            """
            if self._name is None:
                self._name = self._callmethod('_name')
            return self._name

        @property
        def size(self) -> int:
            """
            Return size of the memory block in bytes.
            """
            if self._mem is None:
                self._mem = SharedMemory(name=self.name, create=False)
            return self._mem.size

        @property
        def buf(self) -> memoryview:
            """
            Return a ``memoryview`` into the context of the memory block.
            """
            if self._mem is None:
                self._mem = SharedMemory(name=self.name, create=False)
            return self._mem.buf

        def _list_memory_blocks(self, *, include_self=True) -> list[str]:
            """
            Return names of shared memory blocks being
            tracked by the ``ServerProcess`` that "owns" the current
            proxy object.

            You can call this method on any memoryblock proxy object.
            If for some reason there's no such proxy handy, you can create a tiny, temporary member block
            for this purpose:

                blocks = manager.MemoryBlock(1)._list_memory_blocks(include_self=False)
            """
            blocks = self._callmethod('_list_memory_blocks')
            if not include_self:
                blocks.remove(self.name)
            return blocks

        def __str__(self):
            return f"<{self.__class__.__name__} '{self.name}' at {self._id}, size {self.size}>"
```

Register the class:

```python
class Server(multiprocessing.managers.Server):
    def shutdown(self, c):
        # If there are `MemoryBlockProxy` objects alive when the manager exists its context manager,
        # the corresponding shared memory blocks will live beyond the server process, leading to
        # warnings like this:
        #
        #    UserWarning: resource_tracker: There appear to be 3 leaked shared_memory objects to clean up at shutdown
        #
        # Because we intend to use this server process as a central manager of the shared memory blocks,
        # there shouldn't be any user beyond the lifespan of this server, hence we release these memory blocks.
        #
        # If may be good practice to explicitly `del` `MemoryBlockProxy` objects before the manager shuts down.
        # However, this may be unpractical if there are many proxies, esp if they are nested in other objects.
        #
        # TODO: print some warning or info?
        for ident in self.id_to_refcount.keys():
            z = self.id_to_obj[ident][0]
            if type(z).__name__ == 'MemoryBlock':
                z.__del__()  # `del z` may not be enough b/c its ref count could be > 1.

        super().shutdown(c)


class ServerProcess(multiprocessing.managers.BaseManager):
    _Server = Server

    ...


ServerProcess.register(
    'MemoryBlock', callable=MemoryBlock, proxytype=MemoryBlockProxy
)
ServerProcess.register(
    'ManagedMemoryBlock',
    callable=None,
    proxytype=MemoryBlockProxy,
    create_method=False,
)

managed_memoryblock = functools.partial(managed, typeid='ManagedMemoryBlock')
```


    __all__.extend(('MemoryBlock', 'managed_memoryblock'))
