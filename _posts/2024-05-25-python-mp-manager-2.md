---
layout: post
title: "Reading and Hacking Python's multiprocessing.managers: Part 2"
excerpt_separator: <!--excerpt-->
tags: [Python]
---

In [Part 1](https://zpz.github.io/blog/python-mp-manager-1/) of this series, we have seen a large portion of the functionalities that `multiprocessing.managers` has to offer.
In this article, we will dive into some details, especially around memory management, to gain more understanding. Along the way, we are going to spot a few bugs or flaws in the standard library's implementation, and propose fixes.<!--excerpt-->

Memory management is a critical concern in the current context. We keep some things in the server process and interact with them via "proxies" that reside in other processes (yes there can be more than one client process, as will become commonplace in the following discussions). There is no Python object reference across process boundaries. The server does not have built-in tracking about how many users exist for any particular server-side object, how the users come and go, and when usership has dropped to zero, hence the server-side object should be garbage collected. As a result, the package `multiprocessing.managers` relies on "manual" bookkeeping to track the ref count of each server-side object. The goal of this bookkeeping is to make sure that an object is not garbage collected when it should not be, and is garbage collected when it should be.

Let's fix a few terminologies:


- "manager" or "manager object": instance of `BaseManager`.

- "server process": the process launched and managed by the manager to run the server.

- "server": the instance of `BaseManager.Server` that's running in the server process; "server" and "server process" are mostly used interchangeably.

- "server-side" or "in-server" or "remote" or "target" object: an object that the server creates and keeps, and users interact with via proxies.

- "proxy" or "proxy object": instance of `BaseProxy` that "references" or "represents" or "points to" a server-side object (i.e. its target object).

- "client process" or "user process": the process where the manager runs and proxies reside, and other processes other than the server process where proxies have been passed into; user interacts with proxies in these processes. A proxy may reside in the server process or a client process.

We first list relevant code blocks of the classes `Server`, `BaseManager`, and `BaseProxy` for later reference. Discussion starts right after the code listings.


Sections:

  1. [Code listings](#code-listings)

     - [Server](#server)

     - [BaseManager](#basemanager)

     - [BaseProxy](#baseproxy)

  2. [Tracking ref counts](#tracking-ref-counts)

     - [Creating a proxy](#creating-a-proxy)

     - [Returning a proxy](#returning-a-proxy)

  3. [Pickling](#pickling)

  4. [Bug in pickling](#bug-in-pickling)

  5. [Bug related to pickling](#bug-related-to-pickling)

  6. [Nested proxies](#nested-proxies)

  7. [Bug in nested proxies](#bug-in-nested-proxies)

  8. [Exposed methods](#exposed-methods)

  9. [Exceptions in the server](#exceptions-in-the-server)


## Code listings

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



## Tracking ref counts



`Server` has methods `incref` and `decref`. They are called in `Server` code both for its own need and as requested by a proxy or manager from other processes.
`BaseProxy` has methods `_incref` and `_decref`. They are called in `BaseProxy` to request `Server.incref` and `Server.decref` to be executed on the proxy's target object.
We will go over all these calls to confirm that they keep the ref count of a server-side object correct at all times.
In this context, the correct behavior of a ref count is:

- The ref count of a server-side object is positive as long as there is at least one proxy "out there" for it, and in this situation the object remains alive.
- The ref count drops to 0 once the number of proxies for the object drops to 0, and at that point the object is garbage collected.

The following is not necessary but is preferably the actual behavior:

- The ref count is equal to the number of proxies "out there" for the object in question.

There are two cases where an object is created in the server and a proxy is created for it in client processes.
The first case is by calling a method of `BaseManager` that is programmatically defined by `BaseManager.register` thanks to the parameter `create_method=True`.
The second case is by calling a method of a registered class, *via a proxy*, that appears in `method_to_typeid` of the registry of the class.
Let's call the two cases "create a proxy" and "return a proxy", respectively.
The two cases go through different code paths in both the server and client processes.

### Creating a proxy

When creating a proxy, the "creation method" that is dynamically defined in `BaseManager.register` ([BaseManager], lines 45-54) is called, going through the following steps:

1. `BaseManager._create` ([BaseManager], lines 6-16) is called, which in turn calls `Server.create` ([Server], lines 87-121).
2. The proxy object is crated ([BaseManager], lines 48-51) based on the info returned from the last step as well as registration info.
3. `Server.decref` is called ([BaseManager], lines 52-53; [Server], lines 141-167).

In `Server.create`, the object gets cached in the dict attribute `id_to_obj` of the server; its ref count is tracked in `id_to_refcount` of the server and is initialized to 0. Then, `Server.incref` is called to inc the ref count by 1.
Hence, the ref count is normally 1 before the server responds to the manager.
This remains the status till the beginning of step 2.

When the proxy object is created, `BaseProxy.__init__` calls `BaseProxy._incref` to inc the ref count by 1 ([BaseProxy], lines 12-13). Hence, upon creation of the proxy object, the ref count is 2, that is, 1 more than the correct value. This is the status at the end of step 2.

In step 3, the manager calls the server to dec the ref count by 1, reducing it to the correct value 1.

At the time when the user receives the proxy object, the server tracking indicates the target object has a ref count 1, that is, there is one proxy object "out there in the wild" referencing it. All is good.

But the journey is bumpy. Why inc, inc, dec? Can't we just set it to 1 and be done with it? If we do that, we need to decide where to do it: by the server, by the proxy initializer, or by the manager before or after proxy object creation? What could go wrong?

Let's deviate a little and see a caveat in `Server.create` during bookkeeping ([Server], lines 117-118):

{% highlight python %}
            if ident not in self.id_to_refcount:
                self.id_to_refcount[ident] = 0
{% endhighlight %}

This suggests that, although the remote object appears to have just been created, it does not have to be a new object; it could already be in the tracker. In the case of an object already in the tracker, the server just increments the existing ref count by 1.
There's at least one scenario I can think of: the user function (`callable` on [Server], line 101) may very well retrieve an existing object from some cache it is maintaining.
This is completely possible because, for one thing,
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

This clearly shows that the two proxies, `mag2` and `mag2_another`, reference the same remote object, which has a ref count of 2.

Now we can ask this question: can we leave the work of "incref" to the user, that is, the server initializes the refcount to 0 or leaves it as is?

The answer is "No". Consider this scenario in the example above:

Suppose in thread A we call `manager.Mag(2)` to get a proxy to use.
Then in thread B we call `manager.Mag(2)` to get another proxy for unrelated use.
The server gets a `Magnifier` object but it's the same one that thread A is using, with a ref count 1.
Imagine, just as the server is sending a response to B without touching the ref count,
the proxy in thread A is deleted, which requests the server to dec the ref count, causing it to drop to 0, hence the `Magnifier` object in the server is deleted.
When thread B receives the server's response with address info of the remote object, it will create a proxy, pointing to a remote object that has just disappeared.

This scenario makes the point that the call to `incref` in the server ([Server], line 120) is necessary. The server must hold a spot for the proxy before the proxy object comes into being.

Now that the server has got the ref count right, why do we inc, and then dec, on the user side?

For one thing, a new proxy may be created for an existing remote object without the server's knowledge. This happens when a proxy is passed to another process, increasing the remote object's usership by 1. In concept, it feels right that whenever a proxy object is created, the target object gets one more ref count, hence the `incref` call in `BaseProxy.__init__` ([BaseProxy], lines 12-13). This explains the countering `decref` after the proxy's creation ([BaseManager], lines 52-53).

The need of locking during object creation and ref count modification in the server ([Server], lines 91-118, 124-139, 147-155) can be explained with similar scenarios, where multiple threads in the server are trying to work on the same object.

`BaseProxy._incref` is called only once in the lifetime of a proxy object. At the end of this method, a finalizer is created for this object ([BaseProxy], lines 68-73). The finalizer is invoked when the object is garbage collected.
(For a reference on such finalizers, see [this blog post](https://zpz.github.io/blog/guaranteed-finalization-without-context-manager/).)
The finalizer calls `BaseProxy._decref`, which requests the server to dec the ref count of the proxy's target object. In `Server.decref`, if the ref count drops to 0, it removes the object from its dict cache, `id_to_obj` ([Server], lines 157-167). Often, this removal causes the server-side object to become dangling, and is subsequently garbage collected.


### Returning a proxy

Calling a "remote method" via a proxy primarily entails a call to `BaseProxy._callmethod`. When the call returns a new proxy (as opposed to any "regular" Python value), the following happens in order:

1. The proxy connects to the server and sends a request ([BaseProxy], lines 29-51), which is handled by `Server.serve_client` ([Server], lines 20-51).
2. A new proxy object is created based on the response from the server ([BaseProxy], lines 43-49).
3. A request is made to the server to call its `decref` on the target object of the new proxy ([BaseProxy], lines 50-51).

Steps 2 and 3 are very similar to the steps 2 and 3 in the "creating a proxy" case. The only difference is that, when creating a proxy, these steps happen in the manager, whereas when returning a proxy, these steps happen in a proxy object.

The difference in step 1 of the two cases is on the server side.
In `Server.serve_client`, after we have called the requested method on the target object
([Server], lines 21-41), we check whether this method should return a proxy
([Server], lines 45-46). Note that `gettypeid` is the by-now-familiar `method_to_typeid`, which is a dict mapping method name to registered `typeid`. If `typeid` is not `None`, we call `Server.create` ([Server], lines 46-49) to get proxy info. Note that the result (`res`) of the requested method is passed to `create` as the sole parameter after the connection object and `typeid`. The makes for some different situations compared to when `Server.create` is called by the manager.

Specifically ([Server], lines 95-101), if the `callable` for the registered class represented by `typeid` is `None`, then `res` becomes the remote object "as is".
On the other hand, if `callable` is not `None`, it is called with `res` as the sole, positional argument, and the output of that becomes the remote object. Let's drive this home with an example.

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



BaseManager.register(
    'Magnifier', 
    callable=Magnifier, 
    method_to_typeid={'spawn': 'Magnifier', 'clone': 'ClonedMagnifier'},
    )
BaseManager.register(
    'ClonedMagnifier', 
    callable=None, 
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
Because `clone` returns a `Magnifier` object, we don't want any further changes to it, hence we need to connect it, in `method_to_typeid`, to a registered "typeid" that has no callable.
In addition, the "typeid" should have a proxy class that works for `Magnifier`. 
Apparently, the registered `'Magnifier'` does not work because it has a callable.
We register a new entity called `'ClonedMagnifier'`.

We don't have a custom proxy class for `Magnifier`, but the class `Magnifier` is simple enough to `AuthProxy`.
Hence in the registration we do not specify a proxy class (hence `AuthProxy` will be used).
The argument `exposed` is optional in this case. 
We take care to specify `callable=None` and `create_method=False`.
This registration is purely instructions about how to handle the result of `Magnifier.clone`.

In contrast, `method_to_typeid` connects the method `spawn` to the registered `'Magnifier'`.
However, since this registry has a callable, `spawn` can not output the final thing. Instead, its output is passed to the callable, which will return the desired `Magnifier` object.

The design for these two situations is somewhat confusing. The method `spawn` certainly does not look like it intends to return a `Magnifier` object. It is more a demo of the usage than a natural use case.



## Pickling



A proxy object can be passed to another process, in which it acts as another independent communicator to the remote object in the server.
This raises the question of how a proxy object is pickled and unpickled.
`BaseProxy` implements the special method `__reduce__` to have precise control over pickling.

As listed below, `__reduce__` returns a tuple of two elements. This tuple is pickled and unpickled. 
Upon unpickling, the first element, `RebuildProxy`, is called with arguments provided by the second element. 
The arguments are a few very basic and indispensible attributes of the original proxy; many other attributes of the proxy are left out.
The output of the call is the re-constructed proxy.


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

In `RebuildProxy`, the first two lines test whether this function is being run in the server process that manages this proxy's target object. This test is made possible by one line in `Server.serve_forever`, which assigns the server object to the process as an attribute named "_manager_server".

{% highlight python %}
class Server(object):
    ...

    def serve_forever(self):
        ...
        process.current_process()._manager_server = self
        ...
{% endhighlight %}

If the test turns out positive, then the target object is cached in `server.id_to_local_proxy_obj` in addition to `server.id_to_obj`. In addition, `manager_owned=True` is passed to the proxy creation function. (We will come back to this detail later.)

The determination of `incref` (lines 32-35) involves the attribute `_inheriting` of the process. I do not understand this attribute, but experiments showed its value varies and is important.

If `manager_owned` is `True`, then `BaseProxy.__init__` does not call `BaseProxy._incref` ([BaseProxy], lines 12-13), i.e, it does not increment the ref count of the target object.
Importantly, `_incref` assigns a finalizer ([BaseProxy], lines 68-73), which decrements the ref count when the proxy object is garbage collected.
If initialization does not call `_incref`, destruction does not call `_decref` either, which is consistent. (But, we'll come back to this.)

In summary, if unpickling takes place in the server process, the new proxy does not inc ref count at initialization and does not dec ref count at destruction. On the other hand, if unpickling takes place in any other process, the new proxy increments the ref count at initialization and decrements the ref count at destruction. Both cases are self consistent in terms of ref count.


## Bug in pickling


One likely use case of pickling is to pass a proxy to another process via a queue.
There is a delay between a proxy is put in a queue (in one process) and it is taken out of the queue (in another process).
During the delay, if the proxy in the first process is garbage collected, it would decrement the target object's ref count by 1. If that reduces the ref count to 0, then the target object would be garbage collected.
In that case, when the second process gets the proxy from the queue, the proxy would be referencing a remote object that does not exist!
This is not nearly a corner case; it happens easily if the proxy in the first process is created in a loop.
Let's make up an example to show this issue.


```python
from multiprocessing.managers import SyncManager
from multiprocessing.queues import Queue
from multiprocessing import get_context, Process
from time import sleep


class Magnifier:
    def __init__(self, coef=2):
        self._coef = coef

    def scale(self, x):
        return x * self._coef
    

SyncManager.register('Magnifier', Magnifier)


def worker(q):
    coef = 0
    while True:
        proxy = q.get()
        if proxy is None:
            return
        coef += 1
        y = proxy.scale(8)
        print('coef:', coef, 'scale(8):', y)
        assert y == coef * 8
        sleep(0.01)


def main():
    ctx = get_context('spawn')
    q = Queue(ctx=ctx)
    worker_process = ctx.Process(target=worker, args=(q,))
    worker_process.start()

    with SyncManager(ctx=ctx) as manager:
        for i in range(1, 10):
            scaler = manager.Magnifier(i)
            q.put(scaler)
            sleep(0.1)
        q.put(None)


if __name__ == '__main__':
    main()
```

The example simulates a situation where we create objects in the server process in a loop
and put proxies to these objects in a queue. Another process uses the proxies taken out of the queue.
The proxy object in the main process is short-lived because `scaler` will be assigned to another proxy in the next round of the loop. For any one proxy, if it is reconstructed in the worker process before its symbol `scaler` is re-assigned, all is good. This is the case in the code above because the proxy-producing loop is 10x slower than the proxy-consuming loop.

Running this script got

```shell
$ python3 test_pickle.py 
coef: 1 scale(8): 8
coef: 2 scale(8): 16
coef: 3 scale(8): 24
coef: 4 scale(8): 32
coef: 5 scale(8): 40
coef: 6 scale(8): 48
coef: 7 scale(8): 56
coef: 8 scale(8): 64
coef: 9 scale(8): 72

```

However, after we swapped the sleep times in the two processes, we got this:


```shell
$ python3 test_pickle.py 
coef: 1 scale(8): 8
Process SpawnProcess-1:
Traceback (most recent call last):
  File "/usr/lib/python3.10/multiprocessing/process.py", line 314, in _bootstrap
    self.run()
  File "/usr/lib/python3.10/multiprocessing/process.py", line 108, in run
    self._target(*self._args, **self._kwargs)
  File "/home/docker-user/mpservice/tests/experiments/test_pickle.py", line 21, in worker
    proxy = q.get()
  File "/usr/lib/python3.10/multiprocessing/queues.py", line 122, in get
    return _ForkingPickler.loads(res)
  File "/usr/lib/python3.10/multiprocessing/managers.py", line 942, in RebuildProxy
    return func(token, serializer, incref=incref, **kwds)
  File "/usr/lib/python3.10/multiprocessing/managers.py", line 990, in AutoProxy
    proxy = ProxyType(token, serializer, manager=manager, authkey=authkey,
  File "/usr/lib/python3.10/multiprocessing/managers.py", line 792, in __init__
    self._incref()
  File "/usr/lib/python3.10/multiprocessing/managers.py", line 847, in _incref
    dispatch(conn, None, 'incref', (self._id,))
  File "/usr/lib/python3.10/multiprocessing/managers.py", line 93, in dispatch
    raise convert_to_error(kind, result)
multiprocessing.managers.RemoteError: 
---------------------------------------------------------------------------
Traceback (most recent call last):
  File "/usr/lib/python3.10/multiprocessing/managers.py", line 209, in _handle_request
    result = func(c, *args, **kwds)
  File "/usr/lib/python3.10/multiprocessing/managers.py", line 439, in incref
    raise ke
  File "/usr/lib/python3.10/multiprocessing/managers.py", line 426, in incref
    self.id_to_refcount[ident] += 1
KeyError: '7fc53f42feb0'
---------------------------------------------------------------------------
```

In this case, the main process sleeps 0.01 seconds before going to the next loop and re-assigning `scaler` to another proxy object, thereby garbage collects the previous proxy object (and its corresponding target object). In the worker process, it sleeps 0.1 seconds between loops. As a result, by the time the second proxy is taken out of the queue, its remote object has been destroyed already. This scenario is *not* a rare, special one.

On the client side in the worker process, the error happens in `BaseProxy.__init__` when it tries to increment the ref count of the remote object ([BaseProxy], lines 13, 61). In the server, the error happens in the method `Server.incref` ([Server], line 126), where the object with the specified identifier does not exist.

We realize that the purpose of pickling a proxy in this context is to use it (probably always) in another process; the purpose is never persistence. In other words, the unpickling *will happen*. Since the issue is that ref counting at unpickling may be too late, the solution is to do that preemptively at pickling.
(This reasoning is the same as the `incref` in `Server.create`; [Server], line 120.)
The fix is listed below.


```python

class BaseProxy(object):

    def __reduce__(self):
        conn = self._Client(self._token.address, authkey=self._authkey)
        dispatch(conn, None, 'incref', (self._id, ))
        # Above is fix: pre-emptive `incref`.

        # Begin original code of `__reduce__`---
        kwds = {}
        ...


def RebuildProxy(func, token, serializer, kwds):
    ...

    obj = func(token, serializer, incref=incref, **kwds)
    # Above is original code of `RebuildProxy`, just don't return yet.

    # Fix: counter the extra `incref` that's done in the line above---
    if incref:
        conn = self._Client(obj._token.address, authkey=obj._authkey)
        dispatch(conn, None, 'decref', (obj._id, ))

    return obj
```

In the revised `RebuildProxy`, we do not change the original logic of `incref` that is passed to `func` (and eventually `BaseProxy.__init__`);
usually this value is `True`. Why don't we just use `incref=False` to skip the increment, since we have already done it in `__reduce__`? The reason is we need the finalizer that's set up in `BaseProxy._incref` ([BaseProxy], lines 68-73). We counter the extra `incref` by a `decref` right after the proxy object is created.


## Bug related to pickling

There is another bug, or defect, that is related to pickling.
First, we observe that a `BaseManager` object can not be pickled because it contains some things (at least the server process object) that can not be pickled. The idea of pickling a manager and using it in another process probably does not make much sense anyway.
With this observation, we notice that `BaseProxy.__reduce__` does not pass along the `_manager` attribute of the original proxy, therefore a proxy object out of unpickling has argument `manager=None` to its `BaseProxy.__init__`.

Now look at `BaseProxy._callmethod` ([BaseProxy], line 42). After a proxy-returning method is called, the manager object is needed to construct a proxy for the method's response ([BaseProxy], line 44). This suggests: *if we pass a proxy to another process, then in the other process, we can not call proxy-returning methods on the proxy*.
This is rather suprising and broken in that the (seemingly) same proxy objects in different processes have different capabilities.

Can we fix this? The breaking line in `_callmethod` obtains `proxytype` from the registry on the manager. The natural question is, can we obtain `proxytype` from somewhere else?

Definitely, the server also has a copy of the registry. The solution is to return the relevant registry info from the remote method call. In `Server.serve_client` ([Server], lines 46-49), change

{% highlight python %}
msg = ('#PROXY', (rexposed, token))
{% endhighlight %}

to

{% highlight python %}
msg = ('#PROXY', (rexposed, token, self.registry[typeid][-1]))
{% endhighlight %}

Then in `BaseProxy._callmethod` ([BaseProxy], lines 43-52), change

{% highlight python %}
exposed, token = result
proxytype = self._manager._registry[token.typeid][-1]
{% endhighlight %}

to

{% highlight python %}
exposed, token, proxytype = result
{% endhighlight %}


This fixes the problem. After these changes, the few mentions to the manager object in the proxy object seem to be inconsequential, hence it may be okay to remove the parameer `manager` to `BaseProxy.__init__`.


## Nested proxies

Proxies can be "nested" in the sense that a remote object, which is represented by a proxy, can contain proxy objects.
Let's make up a simple example to show this idea.

```python
from multiprocessing import get_context
from multiprocessing.managers import SyncManager


def main():
    with SyncManager(ctx=get_context('spawn')) as manager:
        mydict = manager.dict(a=3, b=4)
        mylist = manager.list(['first', 'second'])
        print('mylist', type(mylist), 'at', id(mylist))

        mydict['seq'] = mylist
        yourlist = mydict['seq']
        print('yourlist', type(yourlist), 'at', id(yourlist))
        print('len(mylist):', len(mylist))
        print('len(yourlist):', len(yourlist))

        yourlist.append('third')
        print('len(yourlist):', len(yourlist))
        print('len(mylist):', len(mylist))
        print(mylist[2])
              

if __name__ == '__main__':
    main()
```

Running this script got:

```shell
$ python3 test_proxy_nest.py 
mylist <class 'multiprocessing.managers.ListProxy'> at 139749626606352
yourlist <class 'multiprocessing.managers.ListProxy'> at 139749625658768
len(mylist): 2
len(yourlist): 2
len(yourlist): 3
len(mylist): 3
third
```

The statement `mydict['seq'] = mylist` assigns the *proxy* `mylist` to the key `'seq'` in the dict inside the server. Let's repeat: the **server-side** dict (not the proxy object `mydict`) now contains an entry that is a proxy object.
The proxy object `mylist` was pickled, passed into the server process, unpickled, and added to the dict.
The statement `yourlist = mydict['seq']` gets the value at the specified key in the remote object, that is,
the proxy object in the server-side dict got pickled, passed out of the server process, and unpickled to a *new* proxy object `yourlist`.
The proxies `mylist` and `yourlist` are different objects at different memory addresses, yet they both point to the same list in the server.
The proxy in the server-side dict is a third proxy pointing to the same list.


## Bug in nested proxies


Let's have a close look at the proxy as it is passed into the server and then back out. With `mydict['seq'] = mylist`, the proxy `mylist` (in the client process) is pickled, calling `BaseProxy.__reduce__`; then in the server process, `RebuildProxy` is called to create a proxy object. Realizing that it is running inside the server process ([pickling], lines 25-26), the code does two things:

1. It passes `manager_owned=True` to `BaseProxy.__init__` ([pickling], line 28). The effect of this argument is that, in the proxy initiation, ref count is not incremented ([BaseProxy], lines 56-58); moreover, later when the proxy is garbage collected, ref count is not decremented (because the finalizer setup is skipped).

2. It adds an entry in `server.id_to_local_proxy_obj` with the same content about the remote object as the entry in `server.id_to_obj` ([pickling], lines 29-31). Specifically, the entry is a tuple consisting of the list object (the real one in the server, not a proxy), the names of its exposed methods, and its `method_to_typeid` value ([Server], line 116). 

These two actions suggest that a proxy that has been passed from outside of the server into it is not tracked by the server's `incref`/`decref` mechinism. Oddly, the `multiprocessing.managers` module does not contain a line that removes an entry from `server.id_to_local_proxy_obj`. (This is not that odd, though, because if there were such a line, it would likely be in `Server.decref` to be called when an in-server proxy is garbage collected, yet the first action has determined that the garbage collection of an in-server proxy does not trigger `decref`.) 

This is clearly a bug: `server.id_to_local_proxy_obj` only gets entries added and never gets entries removed. Once an in-server object is represented by an in-server proxy at any point, hence the object is referenced by `server.id_to_local_proxy_obj`, then it will live on in the server forever, even after all proxies to it (inside or outside of the server) are gone.

We can use an example to show this problem.

```python
from multiprocessing import get_context
from multiprocessing.managers import SyncManager as _SyncManager, Server as _Server, ListProxy
from pprint import pformat

class Server(_Server):
    def debug_info(self, c):
        '''
        Return some info --- useful to spot problems with refcounting
        '''
        # Perhaps include debug info about 'c'?
        with self.mutex:
            result = []
            result.append('  id_to_refcount:')
            for k, v in self.id_to_refcount.items():
                if k != 0:
                    result.append(f'      {k}: {v}')

            result.append('  id_to_obj:')
            for k, v in self.id_to_obj.items():
                if k != '0':
                    result.append(f'      {k}: {v[0]}')
            result.append('  id_to_local_proxy_obj:')
            for k, v in self.id_to_local_proxy_obj.items():
                if k != '0':
                    result.append(f'      {k}: {v[0]}')    

            return '\n'.join(result)
        

class SyncManager(_SyncManager):
    _Server = Server
    


def main():
    with SyncManager(ctx=get_context('spawn')) as manager:
        mydict = manager.dict(a=3, b=4)

        print()
        print('--- dict ---')
        print(manager._debug_info())
        print()


        mylist = manager.list(['a', 'b', 'c', 'd'])

        print()
        print('--- dict and list ---')
        print(manager._debug_info())
        print()

        mydict['c'] = mylist

        print()
        print('--- list added to dict ---')
        print(manager._debug_info())
        print()

        assert type(mydict['c']) is ListProxy
        assert mydict['c'][3] == 'd'

        del mylist

        print()
        print('--- list proxy deleted from client space ---')
        print(manager._debug_info())
        print()

        assert mydict['c'][2] == 'c'

        del mydict['c']

        print()
        print('--- list deleted from dict ---')
        print(manager._debug_info())
        print()



if __name__ == '__main__':
    main()
```

Running this script got:

```shell
$ python3 test_serverside_proxy.py 

--- dict ---
  id_to_refcount:
      7f14b45044c0: 1
  id_to_obj:
      7f14b45044c0: {'a': 3, 'b': 4}
  id_to_local_proxy_obj:


--- dict and list ---
  id_to_refcount:
      7f14b45044c0: 1
      7f14b4504600: 1
  id_to_obj:
      7f14b45044c0: {'a': 3, 'b': 4}
      7f14b4504600: ['a', 'b', 'c', 'd']
  id_to_local_proxy_obj:


--- list added to dict ---
  id_to_refcount:
      7f14b45044c0: 1
      7f14b4504600: 1
  id_to_obj:
      7f14b45044c0: {'a': 3, 'b': 4, 'c': <ListProxy object, typeid 'list' at 0x7f14b44df880>}
      7f14b4504600: ['a', 'b', 'c', 'd']
  id_to_local_proxy_obj:
      7f14b4504600: ['a', 'b', 'c', 'd']


--- list proxy deleted from client space ---
  id_to_refcount:
      7f14b45044c0: 1
  id_to_obj:
      7f14b45044c0: {'a': 3, 'b': 4, 'c': <ListProxy object, typeid 'list' at 0x7f14b44df880>}
  id_to_local_proxy_obj:
      7f14b4504600: ['a', 'b', 'c', 'd']


--- list deleted from dict ---
  id_to_refcount:
      7f14b45044c0: 1
  id_to_obj:
      7f14b45044c0: {'a': 3, 'b': 4}
  id_to_local_proxy_obj:
      7f14b4504600: ['a', 'b', 'c', 'd']
```

In this example, we created a dict and a list in the server, with corresponding proxies `mydict` and `mylist`, respectively. Then we added `mylist` (the proxy object, not the real list) to the dict. At this point, there are two entries in `server.id_to_refcount` (one for the dict, one for the list), two entries in `server.id_to_obj` (one for the dict, one for the list), and one entry in `server.id_to_local_proxy_obj` (for the list, which is targeted by the list proxy in the dict).

Then we deleted `mylist` in the client process, which caused the ref count of the list in `server.id_to_refcount` to drop to 0, therefore removed the entry for the list from `server.id_to_obj`. If there were no other references to the list, then this removal would have made the list "orphaned" or "dangling", hence garbage collected. However, there is another reference to it in `server.id_to_local_proxy_obj`, hence the list lives on.

At this point, there is one entry in `server.id_to_refcount` (for the dict), one entry in `server.id_to_obj` (for the dict), and one entry in `server.id_to_local_proxy_obj` (for the list).
The server-side list is alive as it should, because it is targeted by its proxy in the server-side dict, but it is not accounted for in `server.id_to_refcount` nor `server.id_to_obj`.

Finally, we deleted the list proxy from the dict iva the proxy `mydict`. At this point, the list in the server became really orphaned---there is no proxy for it outside or inside the server. However, it is referenced by the never-to-be-deleted entry in `server.id_to_local_proxy_obj`, hence the list will live on although it is no longer accessible by the user or the server, because no code ever reaches into `server.id_to_local_proxy_obj` to excavate useful things.

How can we fix this?

A reasonable direction is to treat all proxies, inside or outside the server process, in one consistent way. This involves two changes:

1. Do not use `server.id_to_local_proxy_obj` and replicate entries to it from `server.id_to_obj`. Just use `server.id_to_obj` to track all proxies, wherever they reside.
2. Do not let `manager_owned=True` skip `incref` in proxy initialization and consequently `decref` in proxy teardown ([BaseProxy], lines 56-58) ; instead, let `incref` and `decref` take place whether `manager_owned` is `True` or `False`. This change will make sure deleting an in-server proxy object will trigger appropriate ref-count decrement.

These changes are supported by the fixes to pickling discussed above.
After these changes, the parameter `manager_owned` for `BaseProxy.__init__` has no impact, hence this parameter may as well be removed.


## Exposed methods

In `multiprocessing.managers`, methods of a server-side object that are made callable via a proxy are said to be "exposed".
This is a tuple or list of method names.
The specification of this value appears in two places. First, `BaseManager.register` has an optional argument `exposed` ([BaseManager], line 19); second, a custom proxy class may have a class attribute `_exposed_` ([BaseManager], line 30).
In addition, this value is also affected by "method_to_typeid", which indicates what methods of a server-side object return server-side objects (as opposed to "regular" values that are sent back to the client). Similar to `exposed`, `method_to_typeid` is an optional argument for `BaseManager.register` ([BaseManager], line 20); in the meantime a custom proxy class may have a class attribute `_method_to_typeid_` ([BaseManager], line 33).
The value of `exposed` for a registered `typeid` is part of the registry that is sent to the server object ([BaseManager], lines 40-42).

When a server-side object is created, its value of `exposed` is determined and sent back as part of the response to the client.
The determination involves two steps:

1. Get the value of `exposed` from the registry. If it is `None`, use the "public methods" of the object ([Server], lines 92-93, 103-104).
2. The methods listed in `method_to_typeid` are added to the `exposed` determined in the last step ([Server], lines 105-110).

Once the client receives `exposed` from the server, its usage has two situations:

1. If `proxytype` is `AutoProxy`, then these methods are defined on the dynamically created subclass of `BaseProxy` (see functions `AutoProxy` and `MakeProxyType` listed in [Part 1](https://zpz.github.io/blog/python-mp-manager-1/)).
2. If `proxytype` is a custom subclass of `BaseProxy`, then `exposed` is passed to `BaseProxy.__init__` as an argument ([BaseProxy], line 9). However, the argument is simply ignored in `BaseProxy.__init__`.

The treatment of this subject is a minor mess, especially when the argument `proxytype` to `BaseManager.register` is a custom class (as opposed to `AutoProxy`). Assuming `proxytype` is `MyProxy`,
there are at least these several confusing aspects:

1. There are many factors affecting its value: optional arguments `exposed` and `method_to_typeid` to `BaseManager.register`; optional class attributes `MyProxy._exposed_` and `MyProxy._method_to_typeid_`. Worse, if the argument `exposed` is specified, then `MyProxy._exposed_` is ignored ([BaseManager], line 30), but `method_to_typeid` and `MyProxy._method_to_typeid_` still have an impact.
2. However this is specified during registration and determined during server-side object creation, its value, which is returned from the server, received by the client, and passed to `MyProxy.__init__`, is *simply ignored*!
3. What methods or callable attributes are available on a `MyProxy` instance are totally determined by the class defination as usual. The class attribute `MyProxy._exposed_`, the argument `exposed`, or anything else, have no impact whatsoever on how the proxy can be used!!

This is not an isolated bug that can be fixed locally. This requires some minor re-design. The [module mpservice.multiprocessing.server_process of the package mpservice](https://github.com/zpz/mpservice/blob/main/src/mpservice/multiprocessing/server_process.py) presents some changes centered on these simplifications:

1. Encourage explicit definition of `BaseProxy` subclasses.
2. Do not use class attribute `_exposed_` (and `_method_to_typeid_`).
3. Remove parameters `exposed` to `BaseProxy.__init__` and `BaseManager.register`.
4. When `proxytype` is `AutoProxy` in the registry, the exposed methods are the object's public methods plus those listed in `method_to_typeid`.



## Exceptions in the server

When user calls a method of a server-side object via its proxy, the call is handled in the server by the method `Server.serve_client`.
The method returns traceback info in several cases, mainly concerning errors in the calling mechanism itself.
However, in probably the most common error situation, that is, erros raised in the user code of the server-side object,
the exception object itself is returned to the client ([Server], lines 42-43).
This loses any and all traceback info in pickling and unpickling.
In nontrivial code, this is not nearly as useful as it can be.

This can be remedied using the traceback-preserving helper class `_ExceptionWithTraceback` provided by the [standard module concurrent.futures](https://github.com/python/cpython/blob/3.10/Lib/concurrent/futures/process.py).


Fixes and changes discussed in this article are implemented in
[the module mpservice.multiprocessing.server_process of the package mpservice](https://github.com/zpz/mpservice).

Our exploration continues in [Part 3](https://zpz.github.io/blog/python-mp-manager-3/).
