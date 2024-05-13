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
# magnifier.py

import multiprocessing
from multiprocessing.managers import BaseManager


class Magnifier:
    def __init__(self, coef=2):
        self._coef = coef

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

Executing this script got this:

```shell
$ python3 magnifier.py 
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


def convert_to_error(kind, result):
    if kind == '#ERROR':
        return result
    elif kind in ('#TRACEBACK', '#UNSERIALIZABLE'):
        if not isinstance(result, str):
            raise TypeError(
                "Result {0!r} (kind '{1}') type is {2}, not str".format(
                    result, kind, type(result)))
        if kind == '#UNSERIALIZABLE':
            return RemoteError('Unserializable message: %s\n' % result)
        else:
            return RemoteError(result)
    else:
        return ValueError('Unrecognized message type {!r}'.format(kind))

class RemoteError(Exception):
    def __str__(self):
        return ('\n' + '-'*75 + '\n' + str(self.args[0]) + '-'*75)
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

### start

When we call `BaseManager.start` or enter its context manager, it starts a "server" process
(lines 34--42), in which `BaseManager._run_server` is executed.
The function `_run_server` creates an instance of the class `BaseManager._Server` (line 70) and eventually starts its infinite service loop (line 79).

Importantly, the class attribute (not instance attribute) `BaseManager._registry` is passed into the `_Server` object in the server process (line 36, 70). This dict contains info about classes that have been registered with the class `BaseManager`; this is central info telling the `_Server` object what to do with various requests.

How will the `_Server` object in the server process communicate with the `BaseManager` object or other things in the "main" process? That will use some inter-process communication mechanisms provided by `multiprocessing.connection` (`BaseManager._Listener` and `BaseManager._Client` on line 21). Upon start of the server process, the `BaseManager` object receives the address of the `_Server` object, to be used for future communications.
(Line 73: the server process sends the address; line 45: the `BaseManager` object receives the address.)

### register

`BaseManager.register` (line 110) is used to register a class that we intend to use in the server process
and talk to from other processes. This method accepts several (optional) parameters, each representing a whole lot of info that we are not able to clearly explain before seeing concrete applications.

`typeid` is a unique name of the class. Usually this is the name of the class. For example, to register the class `Doubler`,
we can use `typeid="Doubler"`.

`callable` is to be used to create an object of the said class (which is represented by `typeid`), given other parameters as appropriate. For example,
`callable=Doubler` (the class object) is a natural choice. If `callable` is `None`, an object will be created in some way without a final call to a "definitive" function (such as a class object). See `Server.create` ([Server], lines 226--232).

Why is that? Well, although we say "register a class", named by `typeid`, it does not have to be only one type. `typeid` is just a name; it could represent different things in different situations, and the right "thing" could be created by code in flexible ways.

We register something, whose type is labeled by `typeid`, to be created in the server process. In other processes, this thing will be represented by a "proxy".
The particular callable that will be used to create this proxy object is given by the parameter `proxytype`, which may be a class object or a function.
If not provided, a generic `AutoProxy` will be used (lines 118-119). This parameter is used in lines 139-142.

`exposed` is a list (or tuple) of method names of the registered class that should be made callable via its proxy. This info is saved in the registry (line 131) and passed to the server process. If `exposed` is `None` and `proxytype` is `None` (hence `proxytype` will assume `AutoProxy`, per lines 118-119), then a `None` is saved in the registry (line 121, 131). Well, this does not mean no method will be exposed on the proxy! When a proxy object is created, it receives an argument `exposed`, which is *not* this `None`, but rather is a value returned from the server process (lines 138-142). That value turns out to be all the "public" methods of our class as determined by the function `public_methods` ([Helpers], lines 57).

In our example program, we have seen the public method `scale` is automatically provided by the proxy.

The parameter `method_to_typeid` supports yet another special need. Usually, when we call an "exposed" method on the proxy, the corresponding method on the real object in the server process is called; a value (or "object", as is everything in Python) is created, pickled up, and sent back to us; we get it, unpickle it, and go ahead use it; the original value in the server process is garbage collected, since it is mission completed. However, if an exposed method intends to create some object in the server process, let it stay there, and return a proxy to us, we need `method_to_typeid` to describe this. Specifically, `method_to_typeid` is a dict with method names as keys and registered `typeid`s as values.
For example, suppose our example class `Magnifier` has a method called `spawn` that returns another `Magnifier` instance, and we want this spawned object to stay in the server process and be used through a proxy of itself, we will use `method_to_typeid={'spawn': 'Magnifier'}`.

Finally, the parameter `create_method` determines whether to create a method on `BaseManager` for creating an instance of the registered class in the server process, and giving us a proxy for it (lines 135-147). This method is named after `typeid` (lines 146-147). In our example program, we used `manager.Magnifier(...)` ([Magnifier] lines 21, 27); that is the created method at work.

Let's take a look at how this method is defined (lines 136-145). First, it calls `BaseManager._create` (lines 97-107), which talks to the server process to create an object according to `typeid`,
and returns a `Token`---containing address, ID, that sort of info useful for finding the object in the server process, as well as `exposed`---a tuple of method names to be made available on the proxy.
Second, a proxy object is created (lines 139-142). For now, it suffices to note that the proxy gets the token, the list of exposed methods, and a reference to the `BaseManager` object itself (`manager=self`). Third, a request is sent to the server process to decrement the object's reference count by 1 (lines 143-144). (The maintenance of ref count is a trick subject that we have to defer to a later section.) Finally, the method returns the proxy object.

Whew! Nice and easy!

Did you notice that `create_method` is `True` by default? That means, it can be `False`. You may wonder, when do we ever not want this method? Well, there are situations where this method is not meaningful. We may see examples in the next blog post.

Besides `_create`, several other methods, including `connect`, `_debug_info`, and `_number_of_objects` also communicate with the server process.
We'll refer back to the following code listing when we go over the server code in the next sections.


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
Note that the method `shutdown` is listed in `Server.public`. All the methods in this list are invoked by a request from other processes. In the case of `shutdown`, it is requested in `BaseManager._finalize_manager` ([BaseManager], line 91), which is executed when the `BaseManager` object exits its context manager.

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

We have encountered a call to `create`. In the "creator method" for the registered class in `BaseManager` ([BaseManager], lines 136-145), we call `BaseManager._create` ([BaseManager], line 138), which connects with the server and sends a message with this content ([BaseManager], line 104; [Helpers], line 33):

```python
(None, 'create', (typeid,) + args, kwds)
```

These happen to be the four things unpacked out of the message received on the server side.
Apparently, the first thing does not apply to this call. (In fact, the first thing has to do with some ID or address of a proxied object. The `create` method will create a new object, hence the first thing does not apply.)

Now proceed to the actual method `Server.create` (line 218).
In `create`, we first grab the registered info for the `typeid` (line 223-224), which has been passed in from the `BaseManager` ([BaseManager], lines 34-38, 70).
We create an object following instructions of the registration, tend to some book keeping (lines 246-250), and return two pieces of info concerning the ID of the object and a list of its methods that should be exposed on its proxy.

Take another example, the method `number_of_objects` (line 199). This method is requested to be invoked by `BaseManager._number_of_objects` ([BaseManager], lines 168-176). This method does not take arguments, hence the manager only needs to send in the function name ([BaseManager], line 174), whereas the positional and keyward args assume their default, empty values ([Helpers], line 29).

The methods `shutdown`, `debug_info`, `dummy` are also requested by the `BaseManager` object. In contrast, the method `get_methods` is requested by a proxy object for the list of exposed methods of its corresponding object in the server process (line 257). This info has been saved when the object is created (line 246).

There are three more methods to go through: `accept_connection`, `incref`, and `decref`. The latter two methods help maintain correct ref counts for objects created in the server process upon requests from other processes. Calls to these two methods originate both in the server process, and in other processes (from the manager or proxy objects) relayed into the server process via messages in connections. We will revisit these methods in a later section that is dedicated to memory management.

`accept_connection` is called by a proxy object ([BaseProxy], lines 56, 68) in prep for calling a method of the referenced object ([BaseProxy], line 70).

`accept_connection` calls `serve_client` after sending an acknowledgement to the requester (lines 259-266).

In `serve_client` (starting at line 96), we first retrieve the next message in the connection (line 111),
which the requesting proxy object sends after it receives the connection acknowledgement from the server process,
and unpack four things out of the message (line 112), namely `ident`, `methodname`, `args`, and `kwds`. The first contains info for identifying/locating the object; the rest are the name of the method to be called on the object, along with arguments. Subsequently, we get hold of the object (lines 113-120), the target method of it (lines 122-128), and call it (line 131).

There's a twist about the result of the function call, `res`. It could be something we will send straight back to the requester (a proxy object in another process), or it could be something we will keep in the server process and send back a proxy for it, as is dictated by the parameter `method_to_typeid` when the class is registered ([BaseManager], lines 109-133).
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

After sending result to the requestor, we are finally... not done. We loop back to wait for the next message coming in the same connection (line 107-112). Once a proxy object has opened a connection to the server process, the connection stays open ([BaseProxy], lines 50-57, 63-69). If we make multiple method calls on the proxy object, this connection is reused.

The code suggests that thread in which `accept_connection` and `serve_client` run stays live until the server dies or the proxy object dies.
When the server dies, `stop_event` is set (line 107).
When the proxy dies, the connection is closed on its end, leading to an `EOFError` on line 111, which exits the thread (lines 156-159).
Bear this in mind if your application has a large number of proxy objects.


## AutoProxy

Now we turn to the third player of the game, proxies.
Our example program does not define its own proxy class. Instead, it relies on `AutoProxy` to dynamically define a subclass of `BaseProxy` for it ([BaseManager], line 118-119, 139-142). Let's look into `AutoProxy` before `BaseProxy`.


```python
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

First thing to note is that `AutoProxy` is a function, not a class. It can be thought of as a class because its goal is to return an instance of a certain class.
(By the way, there are some similar examples in Python. For example, `list`, `range`, `int`, etc., are classes rather than functions, whereas `multiprocessing.Queue`, `multiprocessing.Lock`, `multiprocessing.Manager` are methods rather than classes.)

The function itself is simple, although its parameters have a lot of implications.

`token` contains identity/address info of the object in the server process that the proxy is to represent.

`serializer` has to do with the type of inter-process communication mechanism to be used. We'll just use the default value `"pickle"`. Consequently, the communication uses `multiprocessing.conneciton.Listener` and `multiprocessing.connection.Client` ([Helpers], lines 68-71), i.e. `multiprocessing.connection.SocketListener` and `multiprocessing.connection.SocketClient`.

`manager` is the `BaseManager` object, for example, `manager` in our example program.
The proxy object will keep a reference to this manager.

`exposed` is a tuple of the methods (method names, that is) of the object that are to be made available through the proxy. This is important because the proxy class needs to provide a method for each of these methods. If `exposed` is not provided, a request is made to the server process to get the tuple (lines 34-39). This is possible because, at this time, the "target object" already exists in the server process.

Next, `MakeProxyType` is used to create a custom class, which inherits from `BaseProxy` (line 46). Then an instance of the custom class is created and returned (lines 47-50).

Looking at `MakeProxyType` (again, a function rather than a class), we see its only customization to `BaseProxy` is to add each exposed method, which simply calls `BaseProxy._callmethod` (lines 15-21).

The custom proxy class gets a class attribute `_exposed_` (line 22).
The proxy object gets an instance attribute `_isauto=True` (line 49).



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

The method `_callmethod` (lines 59-87) is the workhorse of `BaseProxy`.
It uses a connection to communicate with the server process,
requesting a certain method on the proxy's target object to be executed,
and receiving the result.
If the proxy already has a connection to the server process, it will be reused.
Otherwise, a new connection is established and cached (lines 63-69).

If the remote method returns a regular value, `_callmethod` returns that value (lines 74-75).
If the remote method returns info about a proxy (which would be the case if, e.g., the method is in `method_to_typeid`; see `BaseManager.register`), then a proxy object is created and returned (lines 76-86).

The first block in `__init__` takes some understanding. 
It becomes easier to understand if you know that `"tls"` there stands for "thread-local storage".
The gist of the block is that `self._tls` (line 18) is a `threading.local` object, hence the connection (lines 57, 64, 69) is unique to the thread.
In other words, if a proxy object is passed into other threads, in each thread it will open and use its own connection to communicate with the server.

But why, you may be wondering? The reason is that a connection needs to send and receive messages in correct sequence. If two threads use a shared connection, they will mess up and break.


## SyncManager

`multiprocessing.managers` provides a subclass of `BaseManager` called `SyncManager` that
comes with some useful proxy classes already registered. If you call `multiprocessing.Manager`, it gives you a `SyncManager` object. We are listing a few of its registered classes below.


```python

#
# Proxy types used by SyncManager
#


class IteratorProxy(BaseProxy):
    _exposed_ = ('__next__', 'send', 'throw', 'close')
    def __iter__(self):
        return self
    def __next__(self, *args):
        return self._callmethod('__next__', args)
    def send(self, *args):
        return self._callmethod('send', args)
    def throw(self, *args):
        return self._callmethod('throw', args)
    def close(self, *args):
        return self._callmethod('close', args)


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

SyncManager.register('Iterator', proxytype=IteratorProxy, create_method=False)
SyncManager.register('Queue', queue.Queue)
SyncManager.register('Event', threading.Event, EventProxy)
SyncManager.register('Lock', threading.Lock, AcquirerProxy)
SyncManager.register('Semaphore', threading.Semaphore, AcquirerProxy)
SyncManager.register('list', list, ListProxy)
SyncManager.register('dict', dict, DictProxy)
```

Suppose `sync_manager` is a `SyncManager` object that has been started or is in the context manager.

`sync_manager.Lock()` returns a `AcquirerProxy` object. It can be used to synchronize operations across processes. Similarly, `sync_manager.Event()` and `sync_manager.Semaphore()` provide event and semaphore synchronization tools across processes. Do they have unique use cases not supported by `Lock`, `Event`, and `Semaphore` from `multiprocessing.synchronize`? I don't know yet. They do differ in implementation: they are trivially built with the existing (and simpler) `Lock`, `Event`, and `Semaphore` from `threading`. If they offer performance and usability on par with their `multiprocessing.synchoronize` counterparts, then their simple implementation is indeed an advantage.

`sync_manager.Queue()` provides a queue shared between processes. This is implemented using `queue.Queue` (as opposed to `multiprocessing.queues.Queue`) and uses `AutoProxy`.

`"Iterator"` is registered with `create_method=False`, because we wouldn't do `sync_manager.Iterator(...)` to create a concrete `Iterator` object in the server.
"Iterator" is a more a "protocol" than a concrete class.
`IteratorProxy` is used to represent an iterator in the server. Such objects are always returned from a function or method call. If we want to leave the iterator in the server and use it via a proxy, we need to indicate so with the parameter `method_to_typeid` when registering the class, or with the attribute `_method_to_typeid_` of the proxy class. We'll see an example in a short moment.
`IteratorProxy` implements the methods `__iter__`, `__next__`, among others. As long as the remote object has these methods (as expected if they are "iterators"), we can call these methods on its proxy.

In this context, it is important to note that if the method raises an exception in the server, the same type of exception will be raised by the proxy ([Server], lines 132-133; [BaseProxy], lines 73-87; [Helpers], 74-76). Therefore, when `__next__` on the remote object raises `StopIteration`, the `__next__` on the proxy also raises `StopIteration`, correctly signaling exhaustion of the iterator.

`sync_manager.dict()` creates a dict in the server and returns a `DictProxy`. This class is created by `MakeProxyType`. The only addition is the class attribute (lines 76-78)

```
DictProxy._method_to_typeid_ = {'__iter__': 'Iterator'}
```

This is the application of `IteratorProxy` we promised a short moment ago!
Let's do some experiments.

```python
# test_dict.py

from multiprocessing import get_context


def main():
    data = {'a': 3, 'b': 4, 'c': 5}
    print(type(data.keys()))
    print(type(data.values()))
    print(type(data.items()))
    print()

    manager = get_context('spawn').Manager()
    with manager:
        data = manager.dict(a=3, b=4, c=5)
        print(type(data))
        print()
        print('keys:')
        keys = data.keys()
        print(type(keys))
        print(keys)
        print()
        print('values:')
        values = data.values()
        print(type(values))
        print(values)
        print()
        print('items:')
        items = data.items()
        print(type(items))
        print(items)
        print()
        print('__iter__:')
        it = data.__iter__()
        print(type(it))
        print(it)
        for v in data:
            print(v)


if __name__ == '__main__':
    main()
```

Running it got

```
$ python test_dict.py 
<class 'dict_keys'>
<class 'dict_values'>
<class 'dict_items'>

<class 'multiprocessing.managers.DictProxy'>

keys:
<class 'list'>
['a', 'b', 'c']

values:
<class 'list'>
[3, 4, 5]

items:
<class 'list'>
[('a', 3), ('b', 4), ('c', 5)]

__iter__:
<class 'multiprocessing.managers.IteratorProxy'>
<dict_keyiterator object at 0x7f312cb26160>
a
b
c
```

I was suprised that the functions `keys`, `values`, and `items` do not get the `"Iterator"` treatment in `DictProxy._method_to_typeid_`.
Then I was puzzled by the fact that they return lists on the proxy, while instances of `dict_keys`, `dict_values`, and `dict_items` not only do not pickle to lists, they are not pickleable at all!
A simple check revealed that `dict_keys`, `dict_values`, and `dict_items` objects have method `__iter__` but not `__next__`, therefore
`keys`, `values`, and `items` cannot be mapped to `IteratorProxy` in `_method_to_typeid_`.
To make them map to `IteratorProxy`, some helper code is needed to call `iter(...)` on them and return the result of that.
I still don't know why the author of `multiprocessing.managers` did not go that extra mile.
A possible reason is that the author believed the main usage of a managed dict is via `__getitem__` and `__setitem__`.

`sync_manager.list()` creates a list in the server and returns a `ListProxy`. To define the class `ListProxy`, class `BaseListProxy` is first created by `MakeProxyType`, then `ListProxy` subclasses `BaseListProxy` and adds two methods that `MakeProxyType` can not handle.
Note that the proxy class does not expose the method `__iter__`. It is not necessary because the proxy class already has method `__getitem__` and it raises `IndexError` for out-of-range input. See this example:

```python
>>> class My:
...     def __getitem__(self, idx):
...         if idx >= 10:
...             raise IndexError(idx)
...         return idx
>>> my = My()
>>> for k in my:
...     print(k)
0
1
2
3
4
5
6
7
8
9
>>>
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

