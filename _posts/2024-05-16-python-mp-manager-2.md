---
layout: post
title: "Reading and Hacking Python's multiprocessing.managers: Part 2"
excerpt_separator: <!--excerpt-->
tags: [Python]
---

In Part 1, we have seen a large portion of the functionalities that `multiprocessing.managers` has to offer.
This article goes deeper than Part 1, most around the subject of memory management.<!--excerpt-->

Memory management is a critical concern we keep some things in the server process and use them via "proxies" that reside in other processes (yes there can be more than one client processes). There is no object references across process boundaries. The server does not have an existing way to know how many users exist for one specific server-side object, how they are changing, and when there is no more user and the object should be garbage collected. We need to go out of our usual way to track the ref count of each object created in the server and pointed to by proxies. The goal of this explicit book keeping is to make sure that an object is not garbage collected when it should not be, and is garbage collected when it should be.


### Server

```python

class Server(object):
    '''
    Server class which runs in a process controlled by a manager object
    '''

    def serve_client(self, conn):
        '''
        Handle requests from the proxies in a particular process/thread
        '''
        util.debug('starting server thread to service %r',
                   threading.current_thread().name)

        recv = conn.recv
        send = conn.send
        id_to_obj = self.id_to_obj

        while not self.stop_event.is_set():

            try:
                methodname = obj = None
                request = recv()
                ident, methodname, args, kwds = request
                try:
                    obj, exposed, gettypeid = id_to_obj[ident]
                except KeyError as ke:
                    try:
                        obj, exposed, gettypeid = \
                            self.id_to_local_proxy_obj[ident]
                    except KeyError:
                        raise ke

                if methodname not in exposed:
                    raise AttributeError(
                        'method %r of %r object is not in exposed=%r' %
                        (methodname, type(obj), exposed)
                        )

                function = getattr(obj, methodname)

                try:
                    res = function(*args, **kwds)
                except Exception as e:
                    msg = ('#ERROR', e)
                else:
                    typeid = gettypeid and gettypeid.get(methodname, None)
                    if typeid:
                        rident, rexposed = self.create(conn, typeid, res)
                        token = Token(typeid, self.address, rident)
                        msg = ('#PROXY', (rexposed, token))
                    else:
                        msg = ('#RETURN', res)

            except AttributeError:
                if methodname is None:
                    msg = ('#TRACEBACK', format_exc())
                else:
                    try:
                        fallback_func = self.fallback_mapping[methodname]
                        result = fallback_func(
                            self, conn, ident, obj, *args, **kwds
                            )
                        msg = ('#RETURN', result)
                    except Exception:
                        msg = ('#TRACEBACK', format_exc())

            except EOFError:
                util.debug('got EOF -- exiting thread serving %r',
                           threading.current_thread().name)
                sys.exit(0)

            except Exception:
                msg = ('#TRACEBACK', format_exc())

            try:
                try:
                    send(msg)
                except Exception:
                    send(('#UNSERIALIZABLE', format_exc()))
            except Exception as e:
                util.info('exception in thread serving %r',
                        threading.current_thread().name)
                util.info(' ... message was %r', msg)
                util.info(' ... exception was %r', e)
                conn.close()
                sys.exit(1)

    def create(self, c, typeid, /, *args, **kwds):
        '''
        Create a new shared object and return its id
        '''
        with self.mutex:
            callable, exposed, method_to_typeid, proxytype = \
                      self.registry[typeid]

            if callable is None:
                if kwds or (len(args) != 1):
                    raise ValueError(
                        "Without callable, must have one non-keyword argument")
                obj = args[0]
            else:
                obj = callable(*args, **kwds)

            if exposed is None:
                exposed = public_methods(obj)
            if method_to_typeid is not None:
                if not isinstance(method_to_typeid, dict):
                    raise TypeError(
                        "Method_to_typeid {0!r}: type {1!s}, not dict".format(
                            method_to_typeid, type(method_to_typeid)))
                exposed = list(exposed) + list(method_to_typeid)

            ident = '%x' % id(obj)  # convert to string because xmlrpclib
                                    # only has 32 bit signed integers
            util.debug('%r callable returned object with id %r', typeid, ident)

            self.id_to_obj[ident] = (obj, set(exposed), method_to_typeid)
            if ident not in self.id_to_refcount:
                self.id_to_refcount[ident] = 0

        self.incref(c, ident)
        return ident, tuple(exposed)

    def incref(self, c, ident):
        with self.mutex:
            try:
                self.id_to_refcount[ident] += 1
            except KeyError as ke:
                # If no external references exist but an internal (to the
                # manager) still does and a new external reference is created
                # from it, restore the manager's tracking of it from the
                # previously stashed internal ref.
                if ident in self.id_to_local_proxy_obj:
                    self.id_to_refcount[ident] = 1
                    self.id_to_obj[ident] = \
                        self.id_to_local_proxy_obj[ident]
                    obj, exposed, gettypeid = self.id_to_obj[ident]
                    util.debug('Server re-enabled tracking & INCREF %r', ident)
                else:
                    raise ke

    def decref(self, c, ident):
        if ident not in self.id_to_refcount and \
            ident in self.id_to_local_proxy_obj:
            util.debug('Server DECREF skipping %r', ident)
            return

        with self.mutex:
            if self.id_to_refcount[ident] <= 0:
                raise AssertionError(
                    "Id {0!s} ({1!r}) has refcount {2:n}, not 1+".format(
                        ident, self.id_to_obj[ident],
                        self.id_to_refcount[ident]))
            self.id_to_refcount[ident] -= 1
            if self.id_to_refcount[ident] == 0:
                del self.id_to_refcount[ident]

        if ident not in self.id_to_refcount:
            # Two-step process in case the object turns out to contain other
            # proxy objects (e.g. a managed list of managed lists).
            # Otherwise, deleting self.id_to_obj[ident] would trigger the
            # deleting of the stored value (another managed object) which would
            # in turn attempt to acquire the mutex that is already held here.
            self.id_to_obj[ident] = (None, (), None)  # thread-safe
            util.debug('disposing of obj with id %r', ident)
            with self.mutex:
                del self.id_to_obj[ident]
```

### BaseManager

```python

class BaseManager(object):
    '''
    Base class for managers
    '''

    def _create(self, typeid, /, *args, **kwds):
        '''
        Create a new shared object; return the token and exposed tuple
        '''
        assert self._state.value == State.STARTED, 'server not yet started'
        conn = self._Client(self._address, authkey=self._authkey)
        try:
            id, exposed = dispatch(conn, None, 'create', (typeid,)+args, kwds)
        finally:
            conn.close()
        return Token(typeid, self._address, id), exposed

    @classmethod
    def register(cls, typeid, callable=None, proxytype=None, exposed=None,
                 method_to_typeid=None, create_method=True):
        '''
        Register a typeid with the manager type
        '''
        if '_registry' not in cls.__dict__:
            cls._registry = cls._registry.copy()

        if proxytype is None:
            proxytype = AutoProxy

        exposed = exposed or getattr(proxytype, '_exposed_', None)

        method_to_typeid = method_to_typeid or \
                           getattr(proxytype, '_method_to_typeid_', None)

        if method_to_typeid:
            for key, value in list(method_to_typeid.items()): # isinstance?
                assert type(key) is str, '%r is not a string' % key
                assert type(value) is str, '%r is not a string' % value

        cls._registry[typeid] = (
            callable, exposed, method_to_typeid, proxytype
            )

        if create_method:
            def temp(self, /, *args, **kwds):
                util.debug('requesting creation of a shared %r object', typeid)
                token, exp = self._create(typeid, *args, **kwds)
                proxy = proxytype(
                    token, self._serializer, manager=self,
                    authkey=self._authkey, exposed=exp
                    )
                conn = self._Client(token.address, authkey=self._authkey)
                dispatch(conn, None, 'decref', (token.id,))
                return proxy
            temp.__name__ = typeid
            setattr(cls, typeid, temp)
```


### BaseProxy


```python

class BaseProxy(object):
    '''
    A base for proxies of shared objects
    '''
    _address_to_local = {}
    _mutex = util.ForkAwareThreadLock()

    def __init__(self, token, serializer, manager=None,
                 authkey=None, exposed=None, incref=True, manager_owned=False):

        ...
        if incref:
            self._incref()
        ...

    def _connect(self):
        util.debug('making connection to manager')
        name = process.current_process().name
        if threading.current_thread().name != 'MainThread':
            name += '|' + threading.current_thread().name
        conn = self._Client(self._token.address, authkey=self._authkey)
        dispatch(conn, None, 'accept_connection', (name,))
        self._tls.connection = conn

    def _callmethod(self, methodname, args=(), kwds={}):
        '''
        Try to call a method of the referent and return a copy of the result
        '''
        try:
            conn = self._tls.connection
        except AttributeError:
            util.debug('thread %r does not own a connection',
                       threading.current_thread().name)
            self._connect()
            conn = self._tls.connection

        conn.send((self._id, methodname, args, kwds))
        kind, result = conn.recv()

        if kind == '#RETURN':
            return result
        elif kind == '#PROXY':
            exposed, token = result
            proxytype = self._manager._registry[token.typeid][-1]
            token.address = self._token.address
            proxy = proxytype(
                token, self._serializer, manager=self._manager,
                authkey=self._authkey, exposed=exposed
                )
            conn = self._Client(token.address, authkey=self._authkey)
            dispatch(conn, None, 'decref', (token.id,))
            return proxy
        raise convert_to_error(kind, result)

    def _incref(self):
        if self._owned_by_manager:
            util.debug('owned_by_manager skipped INCREF of %r', self._token.id)
            return

        conn = self._Client(self._token.address, authkey=self._authkey)
        dispatch(conn, None, 'incref', (self._id,))
        util.debug('INCREF %r', self._token.id)

        self._idset.add(self._id)

        state = self._manager and self._manager._state

        self._close = util.Finalize(
            self, BaseProxy._decref,
            args=(self._token, self._authkey, state,
                  self._tls, self._idset, self._Client),
            exitpriority=10
            )

    @staticmethod
    def _decref(token, authkey, state, tls, idset, _Client):
        idset.discard(token.id)

        # check whether manager is still alive
        if state is None or state.value == State.STARTED:
            # tell manager this process no longer cares about referent
            try:
                util.debug('DECREF %r', token.id)
                conn = _Client(token.address, authkey=authkey)
                dispatch(conn, None, 'decref', (token.id,))
            except Exception as e:
                util.debug('... decref failed %s', e)

        else:
            util.debug('DECREF %r -- manager already shutdown', token.id)

        # check whether we can close this thread's connection because
        # the process owns no more references to objects for this manager
        if not idset and hasattr(tls, 'connection'):
            util.debug('thread %r has no more proxies so closing conn',
                       threading.current_thread().name)
            tls.connection.close()
            del tls.connection
```



## Keeping track of ref counts



`Server` has methods `incref` and `decref`. They are called in `Server` code both for its own need and as requested by a proxy or manager from other processes.
`BaseProxy` has methods `_incref` and `_decref`. They are called in `BaseProxy` to request `Server.incref` and `Server.decref` to be called on the object that the proxy "references".
We will go over all these calls to confirm that they keep the ref count of an object in the server, which is represented by a proxy or proxies outside of the server (in other processes), correct as all times.
In this context, the correct behavior of a ref count is:

- the ref count of an object in server is positive as long as there is at least one proxy "out there" for it, and in this situation the object remains alive
- the ref count drops to 0 once the number of proxies for the object drops to 0, and at that point the object is garbage collected

The following is not necessary but is likely the actual behavior (for most of the time):

- the ref count is equal to the number of proxies "out there" for the object in question

There are two cases where an object is created in the server and a proxy is created for it in other processes.
The first case is by calling a method that is defined on `BaseManager` in `BaseManager.register` thanks to the parameter `create_method=True`.
The second case is by calling a method on a proxy, and as a result the eventual method that is called on the server object (in the server process) is in the `method_to_typeid` record of the class registration.
Let's call the two cases "create a proxy" and "return a proxy", respectively.

### Creating a proxy

When creating a proxy, the "creation method" that is dynamically defined in `BaseManager.register` ([BaseManager], lines 45-54) is called.

1. `BaseManager._create` ([BaseManager], lines 6-16) is called, which in turn calls `Server.create` ([Server], lines 87-121).
2. The proxy object is crated ([BaseManager], lines 48-51).
3. `Server.decref` is called ([BaseManager], lines 52-53; [Server], lines 141-167).

In `Server.create`, the object gets cached in `id_to_obj` of the server object; its ref count is tracked in `id_to_refcount` of the server object and is initialized to 0. Then, `Server.incref` is called to inc the ref count by 1.
Hence, the ref count is normally 1 before the server responds to the manager.
This remains the status at the end of step 1.

When the proxy object is created, `BaseProxy.__init__` calls `BaseProxy._incref` to inc the ref count by 1 ([BaseProxy], lines 12-13). Hence, upon creation of the proxy object, the ref count is 2. This is the status at the end of step 2.

In step 3, the manager calls the server to dec the ref count by 1, reducing the ref count from 2 to 1.

At the time when the user receives the proxy object, the server tracks its ref count to be 1. All is good.

But the journey is bumpy. Why inc, inc, dec? Can't we just set it to 1 and be done with it?

There's one caveat in `Server.create` during book keeping ([Server], lines 117-118):

```python
            if ident not in self.id_to_refcount:
                self.id_to_refcount[ident] = 0
```

Why the check here? The object is just created, how can its ref count already be tracked?
There's at least one scenario I can think of: the object does not have to be brand new.
It can be an existing object retrieved from some cache. This is completely possible because, for one thing,
the thing registered with `BaseManager` does not have to be a *class*; it can very well be a function.
We can show this scenario with an example:

```python
# mag.py

import multiprocessing
from multiprocessing.managers import BaseManager


class Magnifier:
    def __init__(self, coef=2):
        self._coef = coef

    def scale(self, x):
        return x * self._coef



BaseManager.register('Magnifier', Magnifier)


mags = {}


def Mag(coef):
    try:
        return mags[coef]
    except KeyError:
        v = Magnifier(coef)
        mags[coef] = v
        return v


BaseManager.register('Mag', Mag)

    
def main():
    with BaseManager(ctx=multiprocessing.get_context('spawn')) as manager:
        mag2 = manager.Mag(2)
        x = 3
        y = mag2.scale(x)
        assert y == x * 2

        print('mag2._id:', mag2._id)
        print('number_objects:', manager._number_of_objects())
        print('debug_info:', manager._debug_info())
        print()

        mag2_another = manager.Mag(2)
        assert mag2_another.scale(10) == 20
        print('mag2_another._id:', mag2_another._id)
        print('number_objects:', manager._number_of_objects())
        print('debug_info:', manager._debug_info())

if __name__ == '__main__':
    main()
```

Running this script got:

```shell
$ python3 mag.py 
mag2._id: 7f3702d168f0
number_objects: 1
debug_info:   7f3702d168f0:       refcount=1
    <__mp_main__.Magnifier object at 0x7f3702d168f0>

mag2_another._id: 7f3702d168f0
number_objects: 1
debug_info:   7f3702d168f0:       refcount=2
    <__mp_main__.Magnifier object at 0x7f3702d168f0>
```

`Server.create` sets the ref count to 0 only if the entry does not exist;
otherwise it's left untouched ([Server], lines 117-118). The rational is simple: if the object is already being targeted (by one or more proxies), it already has a positive ref count. Now another proxy is going to target the same object, we must not reset the existing ref count, and should just increment it by 1 for the new proxy.

Do we have to increment the ref count in the server? Can we leave this task to the user (the requester in other processes)?

No. Consider this scenario in the example above:

Suppose in thread A we call `manager.Mag(2)` to get a proxy to use.
Then in thread B we call `manager.Mag(2)` to get another proxy for unrelated use.
The server gets a `Magnifier` object but it's the same one that thread A is using, with a ref count 1,
Imagine, just as the server is sending back message to B's call without touching the ref count,
the proxy in thread A is deleted, which requests server to dec the ref count, which drops to 0, and the `Magnifier` object in the server is deleted.
When thread B receives server response, it will create a proxy pointing to a remote object that has just disappeared.

In conclusion, the call to `incref` in the server ([Server], line 120) is necessary. The server must hold a spot for the proxy before the proxy object comes into being.

Now that the server has got the ref count right, why do we inc, and then dec, on the user side?

For one thing, a new proxy may be created for an existing remote object without the server's knowledge. This happens when a proxy is passed to another process, increasing the remote object's usership by 1. In concept, it feels right that whenever a proxy object is created, the target remote object gets one more ref count, hence the `incref` call in `BaseProxy.__init__` ([BaseProxy], lines 12-13). This explains the countering `decref` after the proxy's creation ([BaseManager], lines 52-53).

The need of locking during object creation and ref count modification in the server ([Server], lines 91-118, 124-139, 147-155) may be explained in smilar scenarios, where multiple threads in the server are trying to work on the same object.

`BaseProxy._incref` is called only once in the lifetime of a proxy object. At the end of this method, a finalizer is created for this object ([BaseProxy], lines 68-73). The finalizer is invoked when the object is garbage collected.
(For a reference on such finalizers, see [this blog post](https://zpz.github.io/blog/guaranteed-finalization-without-context-manager/).)
The finalizer calls `BaseProxy._decref`, which requests the server to dec the ref count of its target object. In `Server.decref`, if the ref count of an object drops to 0, it makes sure the object is garbage collected ([Server], lines 157-167).


### Returning a proxy

Calling a proxy method to return another proxy primarily entails a call to `BaseProxy._callmethod`. The following happens in order:

1. The proxy connects to the server and sends its request ([BaseProxy], lines 29-38, which is handled by `Server.serve_client` ([Server], lines 20-51).
2. A new proxy object is created based on the response from the server ([BaseProxy], lines 43-49).
3. Make a request to the server to call its `decref` on the target object of the proxy ([BaseProxy], lines 50-51).

Steps 2 and 3 are very similar to the steps 2 and 3 in the "creating a proxy" case. The only difference is that, when creating a proxy, the steps 2 and 3 happen in the manager, whereas when returning a proxy, the steps 2 and 3 happen in a proxy object.

The difference in step 1 of the two cases is on the server side.
In `Server.serve_client`, after we have called the requested method on the target object
([Server], lines 21-41), we check whether this method should return a proxy
([Server], lines 45). Note that `gettypeid` is the by-now-familiar `method_to_typeid`, and `typeid` is a registered `typeid` as indicated in `method_to_typeid`. If `typeid` is not `None`, we call `Server.create` ([Server], lines 46-49). Note that in this call, the result, `res`, of the request function is passed to `create` as the sole parameter after the connection object and `typeid`. The makes for some different situations compared to when `Server.create` is called by the manager.

Specifically ([Server], lines 95-101), if the `callable` for the registered class represented by `typeid` is `None`, then `res` becomes the remote object.
On the other hand, if `callable` is not `None`, it is called with `res` as the sole, positional argument. Let's drive this home with an example.

```python
# clone.py

import multiprocessing
from multiprocessing.managers import BaseManager


class Magnifier:
    def __init__(self, coef=2):
        self._coef = coef

    def scale(self, x):
        return x * self._coef

    def clone(self):
        return self.__class__(self._coef)
    
    def spawn(self, coef):
        return coef



BaseManager.register('Magnifier', callable=Magnifier, 
                     method_to_typeid={'spawn': 'Magnifier', 'clone': 'ClonedMagnifier'},
                     )
BaseManager.register('ClonedMagnifier', callable=None, 
                     exposed=('scale', 'clone', 'spawn'), 
                     method_to_typeid={'spawn': 'Magnifier', 'clone': 'ClonedMagnifier'}, 
                     create_method=False,
                     )

    
def main():
    with BaseManager(ctx=multiprocessing.get_context('spawn')) as manager:
        mag = manager.Magnifier(2)
        print(type(mag))
        assert mag.scale(3) == 6
        print(manager._debug_info())
        print()

        clone = mag.clone()
        print(type(clone))
        assert clone.scale(9) == 18
        print(manager._debug_info())
        print()

        spawn = mag.spawn(3)
        print(type(spawn))
        assert spawn.scale(5) == 15
        print(manager._debug_info())
        print()

        clone_spawn = clone.spawn(10)
        print(type(clone_spawn))
        assert clone_spawn.scale(9) == 90
        print(manager._debug_info())
        print()

        spawn_clone = spawn.clone()
        print(type(spawn_clone))
        assert spawn_clone.scale(5) == 15
        print(manager._debug_info())

if __name__ == '__main__':
    main()
```

Running this script got this:

```shell
$ python3 clone.py
<class 'multiprocessing.managers.AutoProxy[Magnifier]'>
  7f4f7018e8f0:       refcount=1
    <__mp_main__.Magnifier object at 0x7f4f7018e8f0>

<class 'multiprocessing.managers.AutoProxy[ClonedMagnifier]'>
  7f4f7018e8f0:       refcount=1
    <__mp_main__.Magnifier object at 0x7f4f7018e8f0>
  7f4f7018ebc0:       refcount=1
    <__mp_main__.Magnifier object at 0x7f4f7018ebc0>

<class 'multiprocessing.managers.AutoProxy[Magnifier]'>
  7f4f7018e8f0:       refcount=1
    <__mp_main__.Magnifier object at 0x7f4f7018e8f0>
  7f4f7018ebc0:       refcount=1
    <__mp_main__.Magnifier object at 0x7f4f7018ebc0>
  7f4f7018ec50:       refcount=1
    <__mp_main__.Magnifier object at 0x7f4f7018ec50>

<class 'multiprocessing.managers.AutoProxy[Magnifier]'>
  7f4f7018e740:       refcount=1
    <__mp_main__.Magnifier object at 0x7f4f7018e740>
  7f4f7018e8f0:       refcount=1
    <__mp_main__.Magnifier object at 0x7f4f7018e8f0>
  7f4f7018ebc0:       refcount=1
    <__mp_main__.Magnifier object at 0x7f4f7018ebc0>
  7f4f7018ec50:       refcount=1
    <__mp_main__.Magnifier object at 0x7f4f7018ec50>

<class 'multiprocessing.managers.AutoProxy[ClonedMagnifier]'>
  7f4f7018e740:       refcount=1
    <__mp_main__.Magnifier object at 0x7f4f7018e740>
  7f4f7018e8f0:       refcount=1
    <__mp_main__.Magnifier object at 0x7f4f7018e8f0>
  7f4f7018eb00:       refcount=1
    <__mp_main__.Magnifier object at 0x7f4f7018eb00>
  7f4f7018ebc0:       refcount=1
    <__mp_main__.Magnifier object at 0x7f4f7018ebc0>
  7f4f7018ec50:       refcount=1
    <__mp_main__.Magnifier object at 0x7f4f7018ec50>
```

We want `clone` and `spawn` to return proxies for `Magnifier` objects.
Because `clone` returns a `Magnifier` object, we don't want any further changes to it, hence we need to link it, in `method_to_typeid`, to a registered "typeid" that has no callable.
In addition, the "typeid" should have a proxy class that works for `Magnifier`. We choose to let `AutoProxy` create a proxy class for us, and all we need to do is to tell it what methods should be exposed. This is necessary because, given `callable=None`, the code has no way to infer "public methods" by itself (from nothing!). We also take care to use `create_method=False`. After all, there is no class for function called `"ClonedMagnifier"`. The registration is purely "free" instructions about how to handle the result of `Magnifier.clone`.

For `spawn`, we choose to link it, in `method_to_typeid`, to the registered `"Magnifier"`. However, since the registered `"Magnifier"` has a callable, `Magnifier.spawn` can not return the final thing. Instead, it must return a single value that is going to be passed to the callable, which will return a `Magnifier`.

Now, a natural question is: what are the needs for these two different paths? (To be answered ...)


## Pickling



A proxy object can be passed to another process.
Upon unpickling, a new, independent proxy object is created as a new client to the remote object in the server.
A proxy object can not be naively pickled, because several things needs special attention, one of them being ref count management.

Pickling is controlled by the method `BaseProxy.__reduce__` (listed below). `__reduce__` returns a tuple of two elements; the first is a function that is going to be called during unpicking to create the new object; the second is a tuple of parameters for the first element. This tuple pickled and then used during unpickling.

Depending on whether the proxy object has been created by `AutoProxy` or is an instance of a hand-rolled class, 
`__reduce__` instructs unpickling to call `RebuildProxy` (listed below) with `AutoProxy` or the hand-rolled class along with a few minimal and original parameters. It leaves out most of the attributes the proxy object has accumulated. (Naive, default pickling would just pickle every attribute of the object.)


```python

class BaseProxy(object):

    def __reduce__(self):
        kwds = {}
        if get_spawning_popen() is not None:
            kwds['authkey'] = self._authkey

        if getattr(self, '_isauto', False):
            kwds['exposed'] = self._exposed_
            return (RebuildProxy,
                    (AutoProxy, self._token, self._serializer, kwds))
        else:
            return (RebuildProxy,
                    (type(self), self._token, self._serializer, kwds))


#
# Function used for unpickling
#

def RebuildProxy(func, token, serializer, kwds):
    '''
    Function used for unpickling proxy objects.
    '''
    server = getattr(process.current_process(), '_manager_server', None)
    if server and server.address == token.address:
        util.debug('Rebuild a proxy owned by manager, token=%r', token)
        kwds['manager_owned'] = True
        if token.id not in server.id_to_local_proxy_obj:
            server.id_to_local_proxy_obj[token.id] = \
                server.id_to_obj[token.id]
    incref = (
        kwds.pop('incref', True) and
        not getattr(process.current_process(), '_inheriting', False)
        )
    return func(token, serializer, incref=incref, **kwds)
```

The first two lines test whether this function is being run in the server process that manages this proxy's target object. This is facilitated by one line in `Server.serve_forever`, which assigns the server object to the process as an attribute named "_manager_server".

```python
class Server(object):
    ...

    def serve_forever(self):
        ...
        process.current_process()._manager_server = self
        ...
```

If the test turns out positive, then the target object is cached in `server.id_to_local_proxy_obj` in addition to `server.id_to_obj`. (TODO: how is this useful?) In addition, `manager_owned=True` is passed to the proxy creation function; this info is useful for nested proxy objects. (TODO: how?)

The determination of `incref` (lines 32-35) involves the attribute `_inheriting` of the process. I tried to understand it. It seems to be relevant on Windows only and is set only during the creation of a process. In regular situations, it seems, `incref` will be `False` only if the input `kwds` contains a `False` value for `"incref"`. This does not occur in the module `multiprocessing.managers` but can be useful in the method `__reduce__` of a custom subclass of `BaseProxy`.

## Nested proxies

## Server-side proxies

## Shared memory

## Providing useful info for exceptions

