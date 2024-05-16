---
layout: post
title: "Reading and Hacking Python's multiprocessing.managers: Part 2"
excerpt_separator: <!--excerpt-->
tags: [Python]
---

Part 2<!--excerpt--> 


## Helpers

This is a listing of some helper code that will be referenced as needed.

```python

#
# Type for identifying shared objects
#

class Token(object):
    '''
    Type to uniquely identify a shared object
    '''
    __slots__ = ('typeid', 'address', 'id')

    def __init__(self, typeid, address, id):
        (self.typeid, self.address, self.id) = (typeid, address, id)

    def __getstate__(self):
        return (self.typeid, self.address, self.id)

    def __setstate__(self, state):
        (self.typeid, self.address, self.id) = state

    def __repr__(self):
        return '%s(typeid=%r, address=%r, id=%r)' % \
               (self.__class__.__name__, self.typeid, self.address, self.id)


#
# Function for communication with a manager's server process
#

def dispatch(c, id, methodname, args=(), kwds={}):
    '''
    Send a message to manager using connection `c` and return response
    '''
    c.send((id, methodname, args, kwds))
    kind, result = c.recv()
    if kind == '#RETURN':
        return result
    raise convert_to_error(kind, result)
```

## BaseManager

The `BaseManager` class is responsible for three things, namely taking (class) registrations, starting a server process,  and creating (registered class) objects in the server process.

```python
#
# Definition of BaseManager
#

class BaseManager(object):
    '''
    Base class for managers
    '''
    _registry = {}
    _Server = Server

    def __init__(self, address=None, authkey=None, serializer='pickle',
                 ctx=None):
        if authkey is None:
            authkey = process.current_process().authkey
        self._address = address     # XXX not final address if eg ('', 0)
        self._authkey = process.AuthenticationString(authkey)
        self._state = State()
        self._state.value = State.INITIAL
        self._serializer = serializer
        self._Listener, self._Client = listener_client[serializer]
        self._ctx = ctx or get_context()

    def start(self, initializer=None, initargs=()):
        '''
        Spawn a server process for this manager object
        '''
        ...

        # pipe over which we will retrieve address of server
        reader, writer = connection.Pipe(duplex=False)

        # spawn process which runs a server
        self._process = self._ctx.Process(
            target=type(self)._run_server,
            args=(self._registry, self._address, self._authkey,
                  self._serializer, writer, initializer, initargs),
            )
        ident = ':'.join(str(i) for i in self._process._identity)
        self._process.name = type(self).__name__  + '-' + ident
        self._process.start()

        # get address of server
        writer.close()
        self._address = reader.recv()
        reader.close()

        # register a finalizer
        self._state.value = State.STARTED
        self.shutdown = util.Finalize(
            self, type(self)._finalize_manager,
            args=(self._process, self._address, self._authkey,
                  self._state, self._Client),
            exitpriority=0
            )

    @classmethod
    def _run_server(cls, registry, address, authkey, serializer, writer,
                    initializer=None, initargs=()):
        '''
        Create a server, report its address and run it
        '''
        # bpo-36368: protect server process from KeyboardInterrupt signals
        signal.signal(signal.SIGINT, signal.SIG_IGN)

        if initializer is not None:
            initializer(*initargs)

        # create server
        server = cls._Server(registry, address, authkey, serializer)

        # inform parent process of the server's address
        writer.send(server.address)
        writer.close()

        # run the manager
        util.info('manager serving at %r', server.address)
        server.serve_forever()

    def __enter__(self):
        if self._state.value == State.INITIAL:
            self.start()
        ...
        return self

    def __exit__(self, exc_type, exc_val, exc_tb):
        self.shutdown()

    @staticmethod
    def _finalize_manager(process, address, authkey, state, _Client):
        '''
        Shutdown the manager process; will be registered as a finalizer
        '''
        ...

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

    def connect(self):
        '''
        Connect manager object to the server process
        '''
        Listener, Client = listener_client[self._serializer]
        conn = Client(self._address, authkey=self._authkey)
        dispatch(conn, None, 'dummy')
        self._state.value = State.STARTED

    def _debug_info(self):
        '''
        Return some info about the servers shared objects and connections
        '''
        conn = self._Client(self._address, authkey=self._authkey)
        try:
            return dispatch(conn, None, 'debug_info')
        finally:
            conn.close()

    def _number_of_objects(self):
        '''
        Return the number of shared objects
        '''
        conn = self._Client(self._address, authkey=self._authkey)
        try:
            return dispatch(conn, None, 'number_of_objects')
        finally:
            conn.close()
```


## Server

We have seen one case of communication with the server process, which happens in `BaseManager._create`.
Much more communication will take place between a *proxy* and the server process, but the mechanism is similar. We'll examine proxy classes in a later section.

`BaseManager.start` starts a server process, in which a `Server` object is created and placed in an infinite service loop. This `Server` object is all about inter-process communication.



```python

#
# Server which is run in a process controlled by a manager
#

class Server(object):
    '''
    Server class which runs in a process controlled by a manager object
    '''
    public = ['shutdown', 'create', 'accept_connection', 'get_methods',
              'debug_info', 'number_of_objects', 'dummy', 'incref', 'decref']

    def __init__(self, registry, address, authkey, serializer):
        ...

        self.registry = registry
        self.authkey = process.AuthenticationString(authkey)
        Listener, Client = listener_client[serializer]

        # do authentication later
        self.listener = Listener(address=address, backlog=16)
        self.address = self.listener.address

        self.id_to_obj = {'0': (None, ())}
        self.id_to_refcount = {}
        self.id_to_local_proxy_obj = {}
        self.mutex = threading.Lock()

    def serve_forever(self):
        '''
        Run the server forever
        '''
        self.stop_event = threading.Event()
        process.current_process()._manager_server = self
        try:
            accepter = threading.Thread(target=self.accepter)
            accepter.daemon = True
            accepter.start()
            try:
                while not self.stop_event.is_set():
                    self.stop_event.wait(1)
            except (KeyboardInterrupt, SystemExit):
                pass
        finally:
            ...
            sys.exit(0)

    def accepter(self):
        while True:
            try:
                c = self.listener.accept()
            except OSError:
                continue
            t = threading.Thread(target=self.handle_request, args=(c,))
            t.daemon = True
            t.start()

    def _handle_request(self, c):
        request = None
        try:
            connection.deliver_challenge(c, self.authkey)
            connection.answer_challenge(c, self.authkey)
            request = c.recv()
            ignore, funcname, args, kwds = request
            assert funcname in self.public, '%r unrecognized' % funcname
            func = getattr(self, funcname)
        except Exception:
            msg = ('#TRACEBACK', format_exc())
        else:
            try:
                result = func(c, *args, **kwds)
            except Exception:
                msg = ('#TRACEBACK', format_exc())
            else:
                msg = ('#RETURN', result)

        try:
            c.send(msg)
        except Exception as e:
            try:
                c.send(('#TRACEBACK', format_exc()))
            except Exception:
                pass

    def handle_request(self, conn):
        '''
        Handle a new connection
        '''
        try:
            self._handle_request(conn)
        except SystemExit:
            # Server.serve_client() calls sys.exit(0) on EOF
            pass
        finally:
            conn.close()

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
                conn.close()
                sys.exit(1)

    def fallback_getvalue(self, conn, ident, obj):
        return obj

    def fallback_str(self, conn, ident, obj):
        return str(obj)

    def fallback_repr(self, conn, ident, obj):
        return repr(obj)

    fallback_mapping = {
        '__str__':fallback_str,
        '__repr__':fallback_repr,
        '#GETVALUE':fallback_getvalue
        }

    def dummy(self, c):
        pass

    def debug_info(self, c):
        '''
        Return some info --- useful to spot problems with refcounting
        '''
        ...

    def number_of_objects(self, c):
        '''
        Number of shared objects
        '''
        ...

    def shutdown(self, c):
        '''
        Shutdown this process
        '''
        try:
            util.debug('manager received shutdown message')
            c.send(('#RETURN', None))
        except:
            import traceback
            traceback.print_exc()
        finally:
            self.stop_event.set()

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

            self.id_to_obj[ident] = (obj, set(exposed), method_to_typeid)
            if ident not in self.id_to_refcount:
                self.id_to_refcount[ident] = 0

        self.incref(c, ident)
        return ident, tuple(exposed)

    def get_methods(self, c, token):
        '''
        Return the methods of the shared object indicated by token
        '''
        return tuple(self.id_to_obj[token.id][1])

    def accept_connection(self, c, name):
        '''
        Spawn a new thread to serve this connection
        '''
        threading.current_thread().name = name
        c.send(('#RETURN', None))
        self.serve_client(c)

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



## BaseProxy

```python

class BaseProxy(object):
    '''
    A base for proxies of shared objects
    '''
    _address_to_local = {}
    _mutex = util.ForkAwareThreadLock()

    def __init__(self, token, serializer, manager=None,
                 authkey=None, exposed=None, incref=True, manager_owned=False):
        with BaseProxy._mutex:
            tls_idset = BaseProxy._address_to_local.get(token.address, None)
            if tls_idset is None:
                tls_idset = util.ForkAwareLocal(), ProcessLocalSet()
                BaseProxy._address_to_local[token.address] = tls_idset

        # self._tls is used to record the connection used by this
        # thread to communicate with the manager at token.address
        self._tls = tls_idset[0]

        # self._idset is used to record the identities of all shared
        # objects for which the current process owns references and
        # which are in the manager at token.address
        self._idset = tls_idset[1]

        self._token = token
        self._id = self._token.id
        self._manager = manager
        self._serializer = serializer
        self._Client = listener_client[serializer][1]

        # Should be set to True only when a proxy object is being created
        # on the manager server; primary use case: nested proxy objects.
        # RebuildProxy detects when a proxy is being created on the manager
        # and sets this value appropriately.
        self._owned_by_manager = manager_owned

        if authkey is not None:
            self._authkey = process.AuthenticationString(authkey)
        elif self._manager is not None:
            self._authkey = self._manager._authkey
        else:
            self._authkey = process.current_process().authkey

        if incref:
            self._incref()

        util.register_after_fork(self, BaseProxy._after_fork)

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



## Keeping Ref Counts Correct (aka Memory Management)

We've seen a large portion of the functionalities that `multiprocessing.managers` has to offer.
We're inching close to the "crown jewel" of Python code---memory management! Yeah!! Exciting time...

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

When creating a proxy, the "creation method" that is dynamically defined in `BaseManager.reigister` ([BaseManager], lines 136-145) is called.

1. `BaseManager._create` ([BaseManager], lines 97-107) is called, which in turn calls `Server.create` ([Server], lines 218-252).
2. The proxy object is crated ([BaseManager], lines 139-142).
3. `Server.decref` is called ([BaseManager], lines 143-144; [Server], lines 286-311).

In `Server.create`, the object gets cached in `id_to_obj` of the server object; its ref count is tracked in `id_to_refcount` of the server object and is initialized to 0. Then, `Server.incref` is called to inc the ref count by 1.
Hence, the ref count is normally 1 before the server responds to the manager.
This remains the status at the end of step 1.

When the proxy object is created, `BaseProxy.__init__` calls `BaseProxy._incref` to inc the ref count by 1 ([BaseProxy], lines 45-46). Hence, upon creation of the proxy object, the ref count is 2. This is the status at the end of step 2.

In step 3, the manager calls the server to dec the ref count by 1, reducing the ref count from 2 to 1.

At the time when the user receives the proxy object, the server tracks its ref count to be 1. All is good.

But the journey is bumpy. Why inc, inc, dec? Can't we just set it to 1 and be done with it?

There's one caveat in `Server.create` during book keeping ([Server], lines 248-249):

```python
            if ident not in self.id_to_refcount:
                self.id_to_refcount[ident] = 0
```

Why the check here? The object is just creatd, how can its ref count already be tracked?
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
otherwise it's left untouched ([Server], lines 248-249). The rational is simple: if the object is already being targeted (by one or more proxies), it already has a positive ref count. Now another proxy is going to target the same object, we must not reset the existing ref count, and should just increment it by 1 for the new proxy.

Do we have to increment the ref count in the server? Can we leave this task to the user (the requester in other processes)?

No. Consider this scenario in the example above:

Suppose in thread A we call `manager.Mag(2)` to get a proxy to use.
Then in thread B we call `manager.Mag(2)` to get another proxy for unrelated use.
The server gets a `Magnifier` object but it's the same one that thread A is using, with a ref count 1,
Imagine, just as the server is sending back message to B's call without touching the ref count,
the proxy in thread A is deleted, which requests server to dec the ref count, which drops to 0, and the `Magnifier` object in the server is deleted.
When thread B receives server response, it will create a proxy pointing to a remote object that has just disappeared.

In conclusion, the call to `incref` in the server ([Server], line 250) is necessary. The server must hold a spot for the proxy before the proxy object comes into being.

Now that the server has got the ref count right, why do we inc, and then dec, on the user side?

For one thing, a new proxy may be created for an existing remote object without the server's knowledge. This happens when a proxy is passed to another process, increasing the remote object's usership by 1. In concept, it feels right that whenever a proxy object is created, the target remote object gets one more ref count, hence the `incref` call in `BaseProxy.__init__` ([BaseProxy], lines 45-46). This explains the countering `decref` after the proxy's creation ([BaseManager], lines 143-144).

The need of locking during object creation and ref count modification in the server ([Server], lines 222-249, 268-284, 292-300) may be explained in smilar scenarios, where multiple threads in the server are trying to work on the same object.

`BaseProxy._incref` is called only once in the lifetime of a proxy object. At the end of this method, a finalizer is created for this object ([BaseProxy], lines 102-107). The finalizer is invoked when the object is garbage collected.
(For a reference on such finalizers, see [this blog post](https://zpz.github.io/blog/guaranteed-finalization-without-context-manager/).)
The finalizer calls `BaseProxy._decref`, which requests the server to dec the ref count of its target object. In `Server.decref`, is the ref count of an object drops to 0, it makes sure the object is garbage collected ([Server], lines 302-311).


### Returning a proxy

Calling a proxy method to return another proxy primarily entails a call to `BaseProxy._callmethod`. The following happens in order:

1. The proxy connects to the server and sends its request ([BaseProxy], lines 63-72), which is handled by `Server.serve_client` ([Server], lines 111-142).
2. A new proxy object is created based on the response from the server ([BaseProxy], lines 77-83).
3. Make a request to the server to call its `decref` on the target object of the proxy.

Steps 2 and 3 are very similar to the steps 2 and 3 in the "creating a proxy" case. The only difference is that, when creating a proxy, the steps 2 and 3 happen in the manager, whereas when returning a proxy, the steps 2 and 3 happen in a proxy object.

The difference in step 1 of the two cases is on the server side.
In `Server.serve_client`, after we have called the requested method on the target object
([Server], lines 110-131), we check whether this method should return a proxy
([Server], lines 135). Note that `gettypeid` is the by-now-familiar `method_to_typeid`, and `typeid` is a registered `typeid` as indicated in `method_to_typeid`. If `typeid` is not `None`, we call `Server.create` ([Server], lines 136-139). Note that in this call, the result, `res`, of the request function is passed to `create` as the sole parameter after the connection object and `typeid`. The makes for some different situations compared to when `Server.create` is called by the manager.

Specifically ([Server], lines 226-232), if the `callable` for the registered class represented by `typeid` is `None`, then `res` becomes the remote object.
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

The first two lines test whether this function is being run in the server process that manages this proxy's target object. This is facilitated by one line in `Server.serve_forever` ([Server], line 33), which assigns the server object to the process as an attribute named "_manager_server".

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

