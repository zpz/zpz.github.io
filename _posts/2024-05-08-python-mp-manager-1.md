---
layout: post
title: "Python's multiprocessing.Manager: a Code Reader"
excerpt_separator: <!--excerpt-->
tags: [Python]
---

There are several ways to communicate between Python processes (as created by the standard package `multiprocessing`).
One common and hugely useful way is by queues. This is an async, passive mechanism, in the sense that a process (the "receiver") waits on a queue and gets whatever a "sender" (in another process) has placed in the queue. The receiver has no control over what to get and when to get---it gets whatever arrives when it arrives.

Async communication is like mails, whereas sync communication is like phonecalls. `multiprocessing.managers` provides ways to make phone calls between processes.<!--excerpt--> By this mechanism, one process would make a specific request to another process and wait for reply on the spot.

I once had a need to enhance or customize or hack `multiprocessing.managers`. For that purpose, I read, and re-read, and re-read, its implementation, trying to understand how it works. It is not the easiest code to follow! This post attemps to explain what I have understood. It can be treated as an annotated version of the [CPython 3.10 module multiprocessing.managers](https://github.com/python/cpython/blob/3.10/Lib/multiprocessing/managers.py). The code listings below ommit segments that are not necessary to understanding. (Some omissions are indicated by `...`.) Also, I only cover parts of the code that "I" needed to understand, but I believe those are the most important parts for a basic understanding.

Let's start with an example. 


## Magnifier

I define a class, run its functionalities in one process, and request the results from another process.

```python
# mag.py

import multiprocessing
from multiprocessing.managers import BaseManager


class Magnifier:
    def __init__(self, coefficient=2):
        self._coef = coefficient

    def scale(self, x):
        return x * self._coef



BaseManager.register('Magnifier', Magnifier)


def main():
    with BaseManager(ctx=multiprocessing.get_context('spawn')) as manager:
        mag2 = manager.Magnifier()
        x = 3
        y = mag2.scale(x)
        print(f"x: {x}, y: {y}")
        assert y == x * 2

        mag3 = manager.Magnifier(3)
        x = 3
        y = mag3.scale(x)
        print(f"x: {x}, y: {y}")
        assert y == x * 3


if __name__ == '__main__':
    main()
```

Executing this script gets this:

```shell
$ python3 mag.py 
x: 3, y: 6
x: 3, y: 9
```

Here is what's happening:

We define a class `Magnifier` and *register* it with the `BaseManager` class (line 16).

With a `BaseManger` object running (line 20), it "manages" a background *server process*, which knows how to use the classes that have been registered.

We call `manager.Magnifier()` (line 21). This method is dynamically defined on the `BaseManager` class when we did the registration. This call tells the server process to create an instance of `Magnifier` and holds on to it for later use. In detail, the server process calls the second parameter given to `register` (line 16), which is the `Magnifier` class, that is, it calls `Magnifier.__init__`, taking parameters as usual. On line 21, it uses the default parameter value(s); one line 27, we pass in an argument. The call to `manager.Magnifier` returns a *proxy* to the `Magnifier` instance that has been created in the server process. `mag2` and `mag3` are proxies to two unrelated objects in the server process.

We call the method `scale` on the proxy object (line 23, 29). This asks the server process to call the `scale` method on the real `Magnifier` object corresponding to the proxy, and return the result. In the main process (the "client" or "user" process), we get this result from the call. The proxy object (`mag2` and `mag3`) is an instance of a proxy class that is dynamically defined during registration.
The proxy class defines the method `scale`, by default, because it is a "public" method of the class that is being registered.

That's it.

If this is new to you, think about it. This opens the door to many, many possibilities.


## Big picture

The manager facility has three players: 

- the manager: it takes class registrations, manages a server process, creates proxy objects for registered classes;
- the server: it is started by the manager in a server process;
- proxies: these are "references" to objects created in the server process; they are used in client processes to communicate with their corresponding "real" objects that live in the server process.

[diagram]

`multiprocessing.managers` defines classes `BaseManager`, `Server`, and `BaseProxy` for these concepts. We'll dig into each of them very soon.



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


#
# Functions for finding the method names of an object
#

def all_methods(obj):
    '''
    Return a list of names of methods of `obj`
    '''
    temp = []
    for name in dir(obj):
        func = getattr(obj, name)
        if callable(func):
            temp.append(name)
    return temp


def public_methods(obj):
    '''
    Return a list of names of methods of `obj` which do not start with '_'
    '''
    return [name for name in all_methods(obj) if name[0] != '_']


#
# Mapping from serializer name to Listener and Client types
#

listener_client = {
    'pickle' : (connection.Listener, connection.Client),
    'xmlrpclib' : (connection.XmlListener, connection.XmlClient)
    }
```

## BaseManager

The `BaseManager` class is responsible for two things, namely starting a server process and taking registrations.

### BaseManager.start

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
```

When we call `BaseManager.start` or enter its context manager, it starts a "server" process
(lines 33--41), in which executing `BaseManager._run_server`.
The function `_run_server` creates an instance of the class `BaseManager._Server` (line 71) and eventually starts its infinite service loop (line 79).

Importantly, the class attribute `BaseManager._registry` (not an instance attribute) is passed into the `_Server` object in the server process (line 34, 71). This dict contains info about classes that have been registered with the class `BaseManager` (see next section); this is central info telling the `_Server` object what to do with various requests.

How will the `_Server` object in the server process communicate with the `BaseManager` object or other things in the "main" process? That will use some inter-process communication mechanisms provided by `multiprocessing.connection` (`BaseManager._Listener` and `BaseManager._Client` on line 21). Upon start of the server process, the `BaseManager` object receives the address of the `_Server` object, to be used for future communications.
(Line 74: the server process sends the address; line 46: the `BaseManager` object receives the address.)

### BaseManager.register

```python

class BaseManager(object):

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


`BaseManager.register` is used to register a class that we intend to use in the server process
and talk to from other processes. This method accepts several (optional) parameters, each representing a whole lot of info that we are not able to clearly explain before seeing concrete applications.

`typeid` is a unique name of the class. Usually this is the name of the class. For example, to register the class `Doubler`,
we can use `typeid="Doubler"`.

`callable` is to be used to create an object of the said class (which is represented by `typeid`), given other parameters as appropriate. For example,
`callable=Doubler` (the class object) is a natural choice. If `callable` is `None`, an object will be created in some way without a final call to a "definitive" function (such as a class object). See `Server.create` (lines 208--214).

Why is that? Well, although we say "register a class", named by `typeid`, it does not have to be only one type. `typeid` is just a name; it could represent different things in different situations, and the right "thing" could be created by code in flexible ways.

We register something, whose type is labeled by `typeid`, to be created in the server process. In other processes, this thing will be represented by a "proxy".
The particular callable that will be used to create this proxy object is given by the parameter `proxytype`, which may be a class object or a function.
If not provided, a generic `AutoProxy` will be used (lines 24-25). This parameter is used in lines 45--48.

`exposed` is a list (or tuple) of method names of the registered class that should be made callable via its proxy. This info is saved in the registry (line 37) and passed to the server process. If `exposed` is `None` and `proxytype` is `None` (hence `proxytype` will assume `AutoProxy`, per lines 24-25), then a `None` is saved in the registry (line 27, 37). Well, this does not mean no method will be exposed on the proxy! When a proxy object is created, it receives an argument `exposed`, which is *not* this `None`, but rather is a value returned from the server process (lines 44-48). That value turns out to be all the "public" methods of our class as determined by the function `public_methods` ([Helpers], lines 57).

In our example program, we have seen the public method `scale` is automatically provided by the proxy.

The parameter `method_to_typeid` supports yet another special need. Usually, when we call an "exposed" method on the proxy, the corresponding method on the real object in the server process is called; a value (or "object", as is everything in Python) is created, pickled up, and sent back to us; we get it, unpickle it, and go ahead use it; the original value in the server process is garbage collected, since it is mission completed. However, if an exposed method intends to create some object in the server process, let it stay there, and return a proxy to us, we need `method_to_typeid` to describe this. Specifically, `method_to_typeid` is a dict with method names as keys and registered `typeid`s as values.
For example, suppose our example class `Magnifier` has a method called `spawn` that returns another `Magnifier` instance, and we want this spawned object to stay in the server process and be used through a proxy of itself, we will use `method_to_typeid={'spawn': 'Magnifier'}`.

Finally, the parameter `create_method` determines whether to create a method on `BaseManager` for creating an instance of the registered class in the server process, and giving us a proxy for it (lines 41-53). This method is named after `typeid` (lines 52-53). In our example program, we used `manager.Magnifier(...)` ([Magnifier] lines 21, 27); that is the created method at work.

Let's take a look at how this method is defined (lines 42-51). First, it calls `BaseManager._create` (lines 3-13), which talks to the server process to create an object according to `typeid`,
and returns a `Token`---containing address, ID, that sort of info useful for finding the object in the server process, as well as `exposed`---a tuple of method names to be made available on the proxy.
Second, a proxy object is created (lines 45-48). For now, it suffices to note that the proxy gets the token, the list of exposed methods, and a reference to the `BaseManager` object itself (`manager=self`). Third, a request is sent to the server process to decrement the object's reference count by 1 (lines 49-50). (The maintenance of ref count is a trick subject that we have to defer to a later section.) Finally, the method returns the proxy object.

Whew! Nice and easy!

Did you notice that `create_method` is `True` by default? That means, it can be `False`. You may wonder, when do we ever not want this method? Well, there are situations where this method is not meaningful. We may see examples in the next blog post.


### BaseManager.connect

Besides `_create`, several other methods, including `connect`, `_debug_info`, and `_number_of_objects` also communicate with the server process.
We'll refer back to the following code listing when we go over the server code in the next sections.


```python

class BaseManager(object):

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

### serve_forever

Once the object is initialized, `serve_forever` is called to put it in an infinite loop (lines 39-40). The loop is halted once the `threading.Event` object `self.stop_event` is set, which happens in the method `shutdown` (line 216).
Note that the method `shutdown` is listed in `Server.public`. All the methods in this list are invoked by a request from other processes. In the case of `shutdown`, it is requested in `BaseManager._finalize_manager` ([BaseManager.start], line 91), which is executed when the `BaseManager` object exits its context manager.

While the server is waiting for a "shutdown" signal, it has a background thread in its own infinite loop (lines 35-37, 48-55.) This thread's job is to accept incoming connection requests. Upon accepting a new connection request, a new thread is started (lines 53-55). The new thread takes over the handling of the new connection (receiving messages, sending responses). The "old" thread continues to wait for new connections.
This is a standard pattern in such server code.

For ease of writing, we'll call the two threads the "connection-accepting" thread and "connection-handling" thread, respectively.

Connection requests come from the `BaseManager` object, which has created this server process, or from proxy objects, which "reference" objects in this server process.

## handle_request

The connection-handling thread calls `handle_request` (lines 84-94), and in turn `_handle_request` (lines 57-82).

The parameter `c` to `_handle_request` is an instance of `multiprocessing.connection.Connection`. We grab the message from this connection and unpack it to four things (lines 62-63). Ignoring the first, the second is a function name, and the remaining two are positional and keyword args to the function.

The function must be one of those listed in `Server.public` (line 64), namely,

```python
 ['shutdown', 'create', 'accept_connection', 'get_methods',
              'debug_info', 'number_of_objects', 'dummy', 'incref', 'decref']
```

Each of these is a method of `Server`. This method is then called, with the positional and keyward args just obtained (lines 65, 70).
The result is sent back over the connection (line 77). Then the connection is closed (line 94) and the connection-handling thread exits.

Let's look at a few concrete examples.

We have encountered a call to `create`. In the "creator method" for the registered class in `BaseManager` ([BaseManager.register], lines 42-51), we call `BaseManager._create` (line 44), which connects with the server and sends a message with this content ([BaseManager.register], line 10; [Helpers], line 33):

```python
(None, 'create', (typeid,) + args, kwds)
```

These happen to be the four things unpacked out of the message received on the server side.
Apparently, the first thing does not apply to this call. (In fact, the first thing has to do with some ID or address of a proxied object. The `create` method will create a new object, hence the first thing does not apply.)

Now proceed to the actual method `Server.create` (line 218).
In `create`, we first grab the registered info for the `typeid` (line 223-224), which has been passed in from the `BaseManager` ([BaseManager.start], lines 34-38, 70).
We create an object following instructions of the registration, tend to some book keeping (lines 246-250), and return two pieces of info concerning the ID of the object and a list of its methods that should be exposed on its proxy.

Take another example, the method `number_of_objects` (line 199). This method is requested to be invoked by `BaseManager._number_of_objects` ([BaseManager.connect], lines 22-30). This method does not take arguments, hence the manager only needs to send in the function name ([BaseManager.connect], line 28), whereas the positional and keyward args assume their default, empty values ([Helpers], line 29).

The methods `shutdown`, `debug_info`, `dummy` are also requested by the `BaseManager` object. In contrast, the method `get_methods` is requested by a proxy object for the list of exposed methods of its corresponding object in the server process (line 257). This info has been saved when the object is created (line 246).

There are three more methods to go through: `accept_connection`, `incref`, and `decref`. The latter two methods help maintain correct ref counts for objects created in the server process upon requests from other processes. Calls to these two methods originate both in the server process, and in other processes (from the manager or proxy objects) relayed into the server process via messages in connections. We will revisit these methods in a later section that is dedicated to memory management.

`accept_connection` is called by a proxy object ([BaseProxy._callmethod], lines 56, 68) in prep for calling a method of the referenced object ([BaseProxy._callmethod], line 71).

`accept_connection` calls `serve_client` after sending an acknowledgement to the requester (lines 259-266).

In `serve_client` (starting at line 96), we first retrieve the next message in the connection (line 111),
which the requesting proxy object sends after it receives the connection acknowledgement from the server process,
and unpack four things out of the message (line 112), namely `ident`, `methodname`, `args`, and `kwds`. The first contains info for identifying/locating the object; the rest are the name of the method to be called on the object, along with arguments. Subsequently, we get hold of the object (lines 113-120), the target method of it (lines 122-128), and call it (line 131).

There's a twist about the result of the function call, `res`. It could be something we will send straight back to the requester (a proxy object in another process), or it could be something we will keep in the server process and send back a proxy for it, as is dictated by the parameter `method_to_typeid` when the class is registered ([BaseManager.register], lines 15-39).
In the second case, we call `create` (lines 135-141) to save `res` in the server process (by putting it in some dict) and obtain info for its proxy.
The call gets three positional arguments (line 137):

```python
rident, rexposed = self.create(conn, typeid, res)
```

where `typeid` is obtained from `gettypeid` (line 135), which is exactly `method_to_typeid`.

There's another twist in the call to `create` related to `typeid` and its registration info (lines 223-224).
If `callable` is `None`, then `res` is directly the object to be saved in the server process, and a proxy for it is returned. On the other hand, if `callable` is not `None`, it will be called with `res` as the sole argument, and the result of that call is to be saved and proxyed.

An example can make this more clear. To continue with the earlier thought of adding a method `spawn` to `Magnifier`, it will go like this:

```python
class Magnifier:
    def __init__(self, coefficient=2):
        ...

    ...

    def spawn(self, coef):
        return coef
```

The registration will go like this

```python
BaseManager.register('Magnifier', Magnifier, method_to_typeid={'spawn': 'Magnifier'})
```

Note that `spawn` does not return a `Magnifier` object. Rather, it returns a value that will be passed to `callable` as the sole positional argument (lines 137, 232). In this example, `callable` is `Magnifier`, hence the value passed to it is the parameter `coefficient`. This appears somewhat restrictive, and requires careful coordination in the designs of the `spawn` and the callable, which does not have to be `Magnifier`---it all depends on that `typeid` is specified in `method_to_typeid`.

Now back to `serve_client`. By the time we are ready to send back a response to the request (line 166), the response message has been prepared as a length-two tuple. The first element is a flag indicating the nature of the value; the second element is the value. 
If the value is a "regular" object, the flag is `"#RETURN"`.
If the value is info for a proxy, the flag is `"#PROXY"`.
Otherwise, some situation has happened and the flag is `"#ERROR` or `"#TRACEBACK"`.

After sending result to the requestor, we are finally... not done. We loop back to wait for the next message coming in the same connection (line 107-112). Once a proxy object has opened a connection to the server process, the connection stays open ([BaseProxy._callmethod], lines 50-57, 63-69). If we make multiple method calls on the proxy object, this connection is reused.

The code suggests that thread in which `accept_connection` and `serve_client` run stays live until the server dies or the proxy object dies.
When the server dies, `stop_event` is set (line 107).
When the proxy dies, the connection is closed on its end, leading to an `EOFError` on line 111, which exits the thread (lines 156-159).
Bear this in mind if your application has a large number of proxy objects.


## Proxy

### BaseProxy._callmethod


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

### Pickling

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

#
# Functions to create proxies and proxy types
#

def MakeProxyType(name, exposed, _cache={}):
    '''
    Return a proxy type whose methods are given by `exposed`
    '''
    exposed = tuple(exposed)
    try:
        return _cache[(name, exposed)]
    except KeyError:
        pass

    dic = {}

    for meth in exposed:
        exec('''def %s(self, /, *args, **kwds):
        return self._callmethod(%r, args, kwds)''' % (meth, meth), dic)

    ProxyType = type(name, (BaseProxy,), dic)
    ProxyType._exposed_ = exposed
    _cache[(name, exposed)] = ProxyType
    return ProxyType


def AutoProxy(token, serializer, manager=None, authkey=None,
              exposed=None, incref=True, manager_owned=False):
    '''
    Return an auto-proxy for `token`
    '''
    _Client = listener_client[serializer][1]

    if exposed is None:
        conn = _Client(token.address, authkey=authkey)
        try:
            exposed = dispatch(conn, None, 'get_methods', (token,))
        finally:
            conn.close()

    if authkey is None and manager is not None:
        authkey = manager._authkey
    if authkey is None:
        authkey = process.current_process().authkey

    ProxyType = MakeProxyType('AutoProxy[%s]' % token.typeid, exposed)
    proxy = ProxyType(token, serializer, manager=manager, authkey=authkey,
                      incref=incref, manager_owned=manager_owned)
    proxy._isauto = True
    return proxy
```



## Proxy examples

```python

#
# Proxy types used by SyncManager
#

class AcquirerProxy(BaseProxy):
    _exposed_ = ('acquire', 'release')
    def acquire(self, blocking=True, timeout=None):
        args = (blocking,) if timeout is None else (blocking, timeout)
        return self._callmethod('acquire', args)
    def release(self):
        return self._callmethod('release')
    def __enter__(self):
        return self._callmethod('acquire')
    def __exit__(self, exc_type, exc_val, exc_tb):
        return self._callmethod('release')


class EventProxy(BaseProxy):
    _exposed_ = ('is_set', 'set', 'clear', 'wait')
    def is_set(self):
        return self._callmethod('is_set')
    def set(self):
        return self._callmethod('set')
    def clear(self):
        return self._callmethod('clear')
    def wait(self, timeout=None):
        return self._callmethod('wait', (timeout,))


class ValueProxy(BaseProxy):
    _exposed_ = ('get', 'set')
    def get(self):
        return self._callmethod('get')
    def set(self, value):
        return self._callmethod('set', (value,))
    value = property(get, set)

    __class_getitem__ = classmethod(types.GenericAlias)


BaseListProxy = MakeProxyType('BaseListProxy', (
    '__add__', '__contains__', '__delitem__', '__getitem__', '__len__',
    '__mul__', '__reversed__', '__rmul__', '__setitem__',
    'append', 'count', 'extend', 'index', 'insert', 'pop', 'remove',
    'reverse', 'sort', '__imul__'
    ))
class ListProxy(BaseListProxy):
    def __iadd__(self, value):
        self._callmethod('extend', (value,))
        return self
    def __imul__(self, value):
        self._callmethod('__imul__', (value,))
        return self


DictProxy = MakeProxyType('DictProxy', (
    '__contains__', '__delitem__', '__getitem__', '__iter__', '__len__',
    '__setitem__', 'clear', 'copy', 'get', 'items',
    'keys', 'pop', 'popitem', 'setdefault', 'update', 'values'
    ))
DictProxy._method_to_typeid_ = {
    '__iter__': 'Iterator',
    }


#
# Definition of SyncManager
#

class SyncManager(BaseManager):
    '''
    Subclass of `BaseManager` which supports a number of shared object types.
    '''

SyncManager.register('Queue', queue.Queue)
SyncManager.register('Event', threading.Event, EventProxy)
SyncManager.register('Lock', threading.Lock, AcquirerProxy)
SyncManager.register('Semaphore', threading.Semaphore, AcquirerProxy)
SyncManager.register('list', list, ListProxy)
SyncManager.register('dict', dict, DictProxy)
SyncManager.register('Value', Value, ValueProxy)
```

## Keeping Ref Counts Correct (aka Memory Management)
