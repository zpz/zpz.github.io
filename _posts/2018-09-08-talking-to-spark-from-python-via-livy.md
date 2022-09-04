---
layout: post
title: Talking to Spark from Python via Livy
excerpt_separator: <!--excerpt-->
tags: [Spark, Livy, Python]
---

In order to use [Spark](https://spark.apache.org) in my self-sufficient Docker containers without worrying about access to a Spark client environment (to use `spark-submit`, for example), I found the Apache [Livy](https://livy.incubator.apache.org/) project. Livy provides a REST service for interacting with a Spark cluster.
<!--excerpt-->
On one hand, Livy is installed (and configured) in a place where it can talk to the Spark server. On the other hand, applications send Spark code as plain text to the Livy server, via regular HTTP mechanisms; no `spark-submit` (or any part of the Spark environment) is needed on this side.

I wrote a small Python utility module to talk to Livy, sending code to it in a number of natural ways and retrieving result from it.

There was some initial struggle around being sure the Livy server is accessible from the development machine and things are configured correctly. The following script tested these things. It also gave me basic understanding of the work flow.

```python
# test_livy_connection.py

import json
import os
import time

import requests

# This test is intended to be run both inside and outside of Docker
# to verify basic connection to the Livy server.

livy_server_url = os.environ['LIVY_SERVER_URL']

host = 'http://' + livy_server_url
headers = {'Content-Type': 'application/json'}
headers_csrf = {'Content-Type': 'application/json', 'X-Requested-By': 'ambari'}


def _wait(url, wait_for_state):
    while True:
        r = requests.get(url, headers=headers)
        print()
        print('in _wait')
        print('    r:    ', r)
        print('    __dict__:    ', r.__dict__)
        rr = r.json()
        print('    json:    ', rr)
        print('    state:    ', rr['state'])
        if rr['state'] == wait_for_state:
            return rr
        time.sleep(2)


def _post(url, data):
    r = requests.post(url, data=json.dumps(data), headers=headers_csrf)
    print()
    print('in _post')
    print('    headers:    ', r.headers)
    print('    __dict__:    ', r.__dict__)
    print('    location:    ', r.headers['location'])
    return r


def test_scala():
    data = {'kind': 'spark'}
    r = _post(host + '/sessions', data)
    location = r.headers['location']
    if location.startswith('null/'):
        location = location[4:]
    assert location.startswith('/')

    session_url = host + location
    _wait(session_url, 'idle')

    data = {'code': '1 + 1'}
    r = _post(session_url + '/statements', data)

    statements_url = host + r.headers['location']
    rr = _wait(statements_url, 'available')
    print()
    print('output:  ', rr['output'])
    assert rr['output']['status'] == 'ok'
    assert rr['output']['data']['text/plain'] == 'res0: Int = 2'

    # Close the session
    requests.delete(session_url, headers=headers_csrf)


if __name__ == '__main__':
    test_scala()
```

When things are set up correctly, the output contains a lot of info about the Spark connection.
Eventually the printout contains

```sh
output:   {'status': 'ok', 'execution_count': 0, 'data': {'text/plain': 'res0: Int = 2'}}
```

Now let's build the utility class `SparkSession`, which accepts either Python or Scala code.
The very basic methods of the class take care of starting and closing a connection to the Livy server:

```python
import json
import os
import time
from typing import Union, Optional, Iterable, Iterator

import requests


def _get_livy_host(livy_server_url: Optional[str] = None):
    '''
    `livy_server_url` is server IP including port number, like '1.2.3.4:8998'.
    '''
    livy_server_url = livy_server_url or os.environ['LIVY_SERVER_URL']

    # remove protocol
    idx = livy_server_url.find('://')
    if idx >= 0:
        livy_server_url = livy_server_url[(idx + 3):]

    return 'http://' + livy_server_url


def polling_intervals(start: Iterable[float],
                      rest: float,
                      max_duration: float = None) -> Iterator[float]:
    def _intervals():
        yield from start
        while True:
            yield rest

    cumulative = 0.0
    for interval in _intervals():
        cumulative += interval
        if max_duration is not None and cumulative > max_duration:
            break
        yield interval


class JsonClient:
    def __init__(self, url: str) -> None:
        self.url = url
        self.session = requests.Session()
        self.session.headers.update({'X-Requested-By': 'ambari'})

    def close(self) -> None:
        self.session.close()

    def get(self, endpoint: str = '') -> dict:
        return self._request('GET', endpoint)

    def post(self, endpoint: str, data: dict = None) -> dict:
        return self._request('POST', endpoint, data)

    def delete(self, endpoint: str = '') -> dict:
        return self._request('DELETE', endpoint)

    def _request(self, method: str, endpoint: str, data: dict = None) -> dict:
        url = self.url.rstrip('/') + endpoint
        response = self.session.request(method, url, json=data)
        response.raise_for_status()
        return response.json()


class SparkSession:
    def __init__(self,
                 livy_server_url=None,
                 kind: str = 'spark'):
        '''
        When `kind` is 'spark', this takes Scala code.
        When `kind` is 'pyspark', this takes Python code.
        '''
        if kind == 'scala':
            kind = 'spark'
        assert kind in ('spark', 'pyspark')

        self._host = _get_livy_host(livy_server_url)
        self._client = JsonClient(self._host)
        self._kind = kind
        self._start()

    def _wait(self, endpoint: str, wait_for_state: str):
        intervals = polling_intervals([0.1, 0.2, 0.3, 0.5], 1.0)
        while True:
            rr = self._client.get(endpoint)
            if rr['state'] == wait_for_state:
                return rr
            time.sleep(next(intervals))

    def _start(self):
        r = self._client.post('/sessions', data={'kind': self._kind})
        session_id = r['id']
        self._session_id = session_id
        self._wait('/sessions/{}'.format(session_id), 'idle')


    def __del__(self):
        if hasattr(self, '_session_id'):
            try:
                self._client.delete('/sessions/{}'.format(self._session_id))
            finally:
                pass
```

The main method of `SparkSession` is called `run`, which takes a piece of Python or Scala code as plain string, and sends it to the Livy server, which in turn sends it to Spark to run.

```python
import textwrap

class SparkSession:
    # ...
    # ...

    def run(self, code: str) -> str:
        data = {'code': textwrap.dedent(code)}
        r = self._client.post(
            '/sessions/{}/statements'.format(self._session_id), data=data)
        statement_id = r['id']
        z = self._wait(
            '/sessions/{}/statements/{}'.format(self._session_id,
                                                statement_id), 'available')
        output = z['output']
        assert output['status'] == 'ok'
        return output['data']['text/plain']

    #...
    # ...
```

What does this method return?
Basically, it returns the "interesting" part of the response of the HTTP `GET` in the method `_wait`.
The `GET` queries the result of executing the submitted code.
The interesting part of the response may be, for example,
some "return value" of the code or some print-out in the code.
This will become more clear as we encounter different code patterns.

Let's subclass `SparkSession` with a `ScalaSparkSession` (with `PySparkSession` to come next).

```python
class ScalaSparkSession(SparkSession):
    def __init__(self, livy_server_url=None):
        super().__init__(livy_server_url, kind='spark')
```

Now the kind of things in the test script above can be done very easily:

```python
# test_spark.py

import pytest

# import our utility functions...

@pytest.fixture(scope='module')
def pysession():
    return PySparkSession()


@pytest.fixture(scope='module')
def scalasession():
    return ScalaSparkSession()


pi_scala = """
    val NUM_SAMPLES = 100000;
    val count = sc.parallelize(1 to NUM_SAMPLES).map { i =>
        val x = Math.random();
        val y = Math.random();
        if (x*x + y*y < 1) 1 else 0
        }.reduce(_ + _);
    val pi = 4.0 * count / NUM_SAMPLES;
    println(\"Pi is roughly \" + pi)
    """

def test_scala(scalasession):
    z = scalasession.run('1 + 1')
    assert z == 'res0: Int = 2'

    z = scalasession.run(pi_scala)
    assert 'Pi is roughly 3.1' in z
```

Let's see it in action:
```sh
$py.test -s test_spark.py:test_scala

============================= test session starts ==============================
platform linux -- Python 3.6.6, pytest-3.8.1, py-1.6.0, pluggy-0.7.1
rootdir: /home/docker-user/work/src/utils, inifile:
collected 1 item

test_spark.py .

========================== 1 passed in 24.39 seconds ===========================
```

Well, not much to see. It simply passed.

The meat of this utility is to help us use `PySpark`.
Basic things can be done with an object created by `SparkSession(kind='pyspark')`, like this:

```python
# test_spark.py

# ...

def test_pyspark():
    sess = SparkSession(kind='pyspark')

    z = sess.run('1 + 1')
    assert z == '2'

    z = sess.run('import math; math.sqrt(2.0)')
    assert z.startswith('1.4142')
```

This works, but we want to make more things easy. The first is to retrieve values of variables
that exist in the Spark session. Suppose we have just run a piece of code with `SparkSession`
and the code defines, calculates, updates some variables. Then, after the call to `run` returns,
these variables still exist in the "session". How can we get their values? Note that with the Livy REST service, all we can get is serialized text. The work here is to find out the type of the target variable,
fetch its value in some textual format, and then restore not only the value, but also the correct type.


```python
import random

import pandas as pd


def unquote(s: str) -> str:
    return s[1:][:-1]  # remove the quotes at the start and end of the string value


class PySparkSession(SparkSession):
    def __init__(self, livy_server_url=None):
        super().__init__(livy_server_url, kind='pyspark')
        self._tmp_var_name = '_tmp_' + str(id(self)) + '_' + str(random.randint(100, 100000))
        self.run('import pyspark; import json')

    def read(self, code: str):
        '''
        `code` is a variable name or a single, simple expression that is valid in the Spark session.        
        '''
        code = '{} = {}; type({}).__name__'.format(self._tmp_var_name, code,
                                                   self._tmp_var_name)
        z_type = unquote(self.run(code))

        if z_type in ('int', 'float', 'bool'):
            z = self.run(self._tmp_var_name)
            return eval(z_type)(z)

        if z_type == 'str':
            z = self.run(self._tmp_var_name)
            return unquote(z)

        if z_type == 'DataFrame':
            code = """\
                for _livy_client_serialised_row in {}.toJSON().collect():
                    print(_livy_client_serialised_row)""".format(
                self._tmp_var_name)
            output = self.run(code)

            rows = []
            for line in output.split('\n'):
                if line:
                    rows.append(json.loads(line))
            z = pd.DataFrame.from_records(rows)
            return z

        try:
            z = self.run('json.dumps({})'.format(self._tmp_var_name))
            zz = json.loads(unquote(z))
            return eval(z_type)(zz)
        except Exception as e:
            raise Exception(
                'failed to fetch an "{}" object: {};  additional error info: {}'.
                format(z_type, z, e))
```

The method `read` takes either a variable's name, or a simple expression, like `'x + 3'`.
First, we pick a random name (to avoid collision with other things) and assign `code` to this variable.
If `code` is simply a variable's name, then this creates a trivial reference.
If `code` is an expression that computes something, then this assigns the result to this random name.
Then we retrieve the type of this random variable, to be used to restore the object on our side.
How we retrieve the value and restore the object depends on this type info.

If the type is `int`, `float`, `bool`, and `str`, we call `run` directly with the random variable's name,
then cast the textual value to the correct type. Note that if the type is `str`,
the value transferred over the internet includes quotes, like "'my string value'". We need to remove the quotes to get the string value "my string value".

If the type is a Spark `DataFrame`, we print it out on the Spark side row by row,
get the print-outs on our side, and assembled the rows into a `pandas` `DataFrame`.

If the type is something else, we "json-ize" it on the Spark side,
and "de-json-ize" the textual value on our side. This works for simple `list`, `dict`, and `tuple` values.
This is not bullet-proof for other types; we'll deal with issues as they come up in practice.

Let's test this part:

```python
# test_spark.py

# ...

pi_py = """\
    import random

    NUM_SAMPLES = 100000
    def sample(p):
        x, y = random.random(), random.random()
        return 1 if x*x + y*y < 1 else 0

    count = sc.parallelize(range(0, NUM_SAMPLES)).map(sample).reduce(lambda a, b: a + b)
    pi = 4.0 * count / NUM_SAMPLES

    mylist = [1, 3, 'abc']
    mytuple = ('a', 'b', 'c', 1, 2, 3)
    mydict = {'a': 13, 'b': 'usa'}

    # spark 2.0    
    # from pyspark.sql import Row
    # pi_df = spark.createDataFrame([Row(value=pi)])

    # spark 1.6:
    from pyspark.sql import SQLContext, Row
    pi_df = SQLContext(sc).createDataFrame([Row(value=pi)])
"""


def test_py(pysession):
    print()

    pysession.run('z = 1 + 3')
    z = pysession.read('z')
    assert z == 4

    pysession.run(pi_py)

    pi = pysession.read('pi')
    print('printing a number:')
    print(pi)
    assert 3.0 < pi < 3.2

    code = '''pip2 = pi + 2'''
    pysession.run(code)
    pip2 = pysession.read('pip2')
    assert 3.0 < pip2 - 2 < 3.2

    mylist = pysession.read('mylist')
    assert mylist == [1, 3, 'abc']
    mytuple = pysession.read('mytuple')
    assert mytuple == ('a', 'b', 'c', 1, 2, 3)
    mydict = pysession.read('mydict')
    assert mydict == {'a': 13, 'b': 'usa'}

    local_df = pysession.read('pi_df')
    print()
    print('printing a {}:'.format(type(local_df)))
    print(local_df)
    pi = local_df.iloc[0, 0]
    assert 3.0 < pi < 3.2

    assert pysession.read('3 + 6') == 9

    print()
    print('printing in Spark session:')
    z = pysession.run('''print(type(pip2))''')
    # `run` does not print.
    # printouts in Spark are collected in the return of `run`.
    print(z)

    # `str` comes out as `str`
    print()
    print(pysession.read('str(type(pi))'))
    print(pysession.read('type(pi_df).__name__'))

    # `bool` comes out as `bool`
    z = pysession.read('''isinstance(pi, float)''')
    print()
    print('printing boolean:')
    print(z)
    print(type(z))
    assert z is True

    assert pysession.read('str(isinstance(pi, float))') == 'True'

    # `bool` comes out as `numpy.bool_`
    # assert session.read(
    #     '''isinstance(pi_df, pyspark.sql.dataframe.DataFrame)''')
```

Here's the outcome:
```sh
$py.test -s test_spark.py:test_py

============================= test session starts ==============================
platform linux -- Python 3.6.6, pytest-3.8.1, py-1.6.0, pluggy-0.7.1
rootdir: /home/docker-user/work/src/utils, inifile:
collected 1 item

test_spark.py 
printing a number:
3.1212

printing a <class 'pandas.core.frame.DataFrame'>:
    value
0  3.1212

printing in Spark session:
<type 'float'>

<type 'float'>
DataFrame

printing boolean:
True
<class 'bool'>
.

========================== 1 passed in 48.85 seconds ===========================
```

The second thing we want `PySparkSession` to help us with is to submit code that reside in different places. So far we have treated code as a prepared string.
Two other common sources of code is a module that defines a collection of PySpark-aware functions,
and a stand-alone script. Note that in both cases we require the code to be valid in the Spark environment, but not necessarily vaid in the Python environment on our side.
For one thing, we do not count on the presence of `pyspark` in our environment but it is necessarily required by the code. For another, the Spark cluster may very well have an older version of Python (like 2.7!) and different set of third party packages. This dicates that we can not `import` the module or script. Rather, the strategy is to find out the path to their files, and grab their content as text.

```python
import pkgutil

class PySparkSession(SparkSession):
    # ...
    # ...

    def run_module(self, module_name: str) -> None:
        '''
        Run the source of a Python module (not a package) in Spark.

        `module_name` is an absolute module name, meaning it's not relative
        to where this code is called. The named module must be reachable
        on the system path via `import`

        Functions and variables defined in that module are available to subsequent code.
        This module must depend on only Python's stblib and Spark modules (plus other modules
        you are sure exist on the Spark cluster). Modules loaded this way should not 'import'
        each other. Imagine the source code of the module is copied and submitted to Spark,
        and the module name is lost.

        The named module is written against the Python version and packages available on the
        Spark cluster. The module is NOT imported by this method; it's entirely possible that
        loading the module in the current environment (not on the Spark cluster) would fail.
        The current method retrieves the source code of the module as text and sends it to Spark.

        Similar concerns about dependencies apply to `run_file`.

        Example (assuming it is meaningful to submit the module `myproject.util.spark` to Spark):

            sess = PySparkSession()
            sess.run_module('myproject.util.spark')

        The module shoule define functions and variables only; it should not actually run
        and return anything meaningful.
        '''
        pack = pkgutil.get_loader(module_name)
        self.run_file(pack.path)

    def run_file(self, file_path: str) -> str:
        '''
        `file_path` is the absolute path to a Python (script or module) file.

        When you know the relative location of the file, refer to `relative_path`.

        This returns a string, which is unlikely to be very useful.
        If you want to capture the result in type-controlled ways, it's better
        to let the file define functions only, then call `run_file` followed by
        `run_function`.
        '''
        assert file_path.startswith('/')
        code = open(file_path, 'r').read()
        return self.run(code)

    # ...
    # ...

import inspect
from pathlib import PurePath


def relative_path(path: str) -> str:
    '''
    Given path `path` relative to the file where this function is called,
    return absolute path.

    For example, suppose this function is called in file '/home/work/src/repo1/scripts/abc.py', then

        relative_path('../../make_data.py')

    returns '/home/work/src/make_data.py', whereas

        relative_path('./details/sum.py')

    returns '/home/work/src/repo1/scripts/details/sum.py'.
    '''
    if path.startswith('/'):
        return path

    caller = inspect.getframeinfo(inspect.stack()[1][0]).filename
    assert caller.endswith('.py')
    p = PurePath(caller).parent
    while True:
        if path.startswith('./'):
            path = path[2:]
        elif path.startswith('../'):
            path = path[3:]
            p = p.parent
        else:
            break
    return str(p.joinpath(path))
```

We intentionally let `run_module` return `None`.

Here is a small test for `run_file`:

```python
# test_spark.py

# ...

def test_file(pysession):
    pysession.run_file(relative_path('./spark_test_scripts/script_a.py'))
    z = pysession.read('magic')
    assert 6.0 < z < 7.0
```

Create sub-directory `spark_test_scripts/` in the directory where `test_file.py` resides,
and put the following file in `spark_test_scripts/`.

```python
# script_a.py

import math

magic = math.pi + 3
```

Running `test_file.py` with `py.test` passes.

Both `run_module` and `run_file` submit the code of entire files.
What about submitting the definition of a single function that is defined in regular ways,
that is, not contained in a text block?
On the first thought this feels systematic alongside `run_module` and `run_file`,
and the name `run_function` or `run_obj` is handy.
(Just for the record, `inspect.getsource` can be useful for this.)
On a second thought, however, this has pretty big complications: it is *very* hard to make sure
this function's dependencies (packages and other functions) are satisfied in the Spark session.
So, let's settle with this: to submit code on the sub-file level, just prepare the code as text.

Now it's really easy to send modules to Spark so that a bunch of functions are defined and ready to be used "in the other world". How do we call these functions? Ideally, we don't want to prepare yet another text block just to call a function. We want it to be nice and easy, like native Python, or like this:

```python
class PySparkSession(SparkSession):
    # ...
    # ...

    def run_function(self, fname: str, *args, **kwargs):
        '''
        `fname`: name of a function existing in the Spark session environment.

        `args`, `kwargs`: simple arguments to the function, such as string, number, short list.
        '''
        code = f'{fname}(*{args}, **{kwargs})'
        return self.read(code)

    # ...
    # ...
```

To test it,

```python
# test_spark.py

# ...

def test_func(pysession):
    f = '''\
    def myfunc(a, b, names, squared=False):
        assert len(a) == 3
        assert len(b) == 3
        assert len(names) == 3
        c = [aa + bb for (aa, bb) in zip(a, b)]
        if squared:
            c = [x*x for x in c]
        d = {k:v for (k,v) in zip(names, c)}
        return d
    '''

    pysession.run(f)

    z = pysession.run_function('myfunc', [1,2,3], [4,6,8], ['first', 'second', 'third'])
    assert {k: z[k] for k in sorted(z)} == {'first': 5, 'second': 8, 'third': 11}

    z = pysession.run_function('myfunc', [1,2,3], [4,6,8], squared=True, names=['first', 'second', 'third'])
    assert {k: z[k] for k in sorted(z)} == {'first': 25, 'second': 64, 'third': 121}
```

This test passes happily.

And I am happy, too.

Until I ask, "what if something goes wrong in the submitted code?"

That could happen. Oh, that will happen. It's on the server side. The HTTP client at best gets informed something is wrong, but has no clue what is wrong.
Some sort of exception handling is in order.

```python

class SparkSessionError(Exception):
    def __init__(self, name, value, traceback, kind) -> None:
        self._name = name
        self._value = value
        self._tb = traceback
        self._kind = kind
        super().__init__(name, value, traceback)

    def __str__(self) -> str:
        kind = 'Spark' if self._kind == 'spark' else 'PySpark'
        return '{}: {} error while processing submitted code.\nname: {}\nvalue: {}\ntraceback:\n{}'.format(
            self.__module__ + '.' + self.__class__.__name__, kind, self._name,
            self._value, ''.join(self._tb))


class SparkSession:
    # ...
    # ...

    def run(self, code: str) -> str:
        data = {'code': textwrap.dedent(code)}
        r = self._client.post(
            '/sessions/{}/statements'.format(self._session_id), data=data)
        statement_id = r['id']
        z = self._wait(
            '/sessions/{}/statements/{}'.format(self._session_id,
                                                statement_id), 'available')
        output = z['output']
        if output['status'] == 'error':
            raise SparkSessionError(output['ename'], output['evalue'],
                                    output['traceback'], self._kind)
        assert output['status'] == 'ok'
        return output['data']['text/plain']

    # ...
    # ...
```

Let's confirm that we get some useful error info:

```python
# test_spark.py

# ...

scala_error = """
    val NUM = 1000
    val count = abc.NUM
"""


def test_scala_error(scalasession):
    try:
        z = scalasession.run(scala_error)
    except SparkSessionError as e:
        print(e)


py_error = """\
    class MySparkError(Exception):
        pass

    a = 3
    b = 4
    raise MySparkError('some thing is so wrong!)
    print('abcd')
"""

def test_py_error(pysession):
    try:
        z = pysession.run(py_error)
    except SparkSessionError as e:
        print(e)
```

Here's the outcome:

```sh
$py.test -s test_spark.py:test_scala_error test_spark.py:test_py_error

============================= test session starts ==============================
platform linux -- Python 3.6.6, pytest-3.8.1, py-1.6.0, pluggy-0.7.1
rootdir: /home/docker-user/work/src/utils, inifile:
collected 2 items

test_spark.py utils.spark.SparkSessionError: Spark error while processing submitted code.
name: Error
value: <console>:25: error: not found: value abc
         val count = abc.NUM
                     ^
traceback:

.utils.spark.SparkSessionError: PySpark error while processing submitted code.
name: SyntaxError
value: EOL while scanning string literal (<stdin>, line 6)
traceback:
  File "<stdin>", line 6
    raise MySparkError('some thing is so wrong!)
                                               ^
SyntaxError: EOL while scanning string literal

.

========================== 2 passed in 29.78 seconds ===========================
```

Looks good to me.
