---
layout: post
title: Some High-level Tips for Python Projects
tags: [Python]
---

Below are tips on a few very high-level and commonly-encountered topics
in Python software development.

* [Directory structure](#directory-structure)
* [Naming](#naming)
* [Documentation](#documentation)
* [Logging](#logging)
* [Testing](#testing)


## Directory structure
<a name="directory-structure"></a>

Suppose the the probject is called 'becool'.
A largely standard directory structure looks like this:

```
becool/
  |-- archive/
  |-- config/
  |-- doc/
  |-- becool/
  |   |-- __init__.py
  |   |-- module_a.py
  |   |-- module_b.py
  |   |-- becooler/
  |   |   |-- __init__.py
  |   |   |-- module_c.py
  |   |   |-- module_d.py
  |-- tests/
  |   |-- __init__.py
  |   |-- test_module_a.py
  |   |-- test_module_b.py
  |   |-- becooler/
  |   |   |-- __init__.py
  |   |   |-- test_module_c.py
  |   |   |-- test_module_d.py
  |-- scripts/
  |-- README.md
  |-- setup.py
```

Some highlights:

- The project name, 'becool', is also the name of the `git` repo.
  Inside this repo, the main content a Python package called 'becool'.
  Hence the repo and the package are both named after the project.
  This is a common situation.
  
  Although nothing forbids you from having multiple packages in one project (e.g. having a `behot` parallel to `becool`), it's usually a bad idea. The reason is that if `becool` and `behot` closely interact, then maybe they should be one single package. On the other hand, if they do not closely interact, then they are independently deployable, therefore it's better to have them in separate projects (i.e. repos).

- Maintain a clean project file structure, and do not be afraid to adjust or refactor. [Some reference](http://as.ynchrono.us/2007/12/filesystem-structure-of-python-project_21.html); [some more](http://stackoverflow.com/questions/193161/what-is-the-best-project-structure-for-a-python-application).

- The directory `scripts` should not contain `__init__.py`, but may contain files named like `*_test.py` or `test_*.py` for testing functions in the scripts.

  Ideally, content of `scripts` is mainly short launchers of functions in packages.
  For this reason, the scripts often do not need tests.

  This directory may also be called `bin`.

- `archive` is for any code segments that you have deleted from the "main line" but for some reason still want to keep a visible copy for reference. Do not use a *branch* for this purpose.

### Where to put tests?

There are mainly two approaches. The first is to have `tests` subdirectories within the package `becool`. Specifically, every directory (includeing the top level `becool`) contains a subdirectory named `tests`, which hosts tests for the modules (i.e. `*.py` files) in its parent directory. The second approach uses a `tests` directory separate from the package being tested. This is the approach shown in the digram above. Both approaches are used by major open source projects.

After some time using the first approach, I switched to the second. The switch is prompted by some confusing situations related to running tests against the package that has been "installed" into the system `site-packages`. I can think of two other reasons. First, conceptually, "tests" are *not* the package's intrinsic functionalities. They are supporting the development of the package, just like "doc" is another supporting component. Second, in substantial projects, one may build some utilities to be used by the test code. In the first approach, the location of this testing infrastructure may become awkward.


### Alternative structure with a `src` sub-directory

There are discussions about having a `src` directory.
[This post](https://hynek.me/articles/testing-packaging/) advocates top-level
directories like `src`, `docs`, `tests`, and a main point for this structure is
related to testing.

Without carefully reading the arguments, **I have switched to use this structure**.
I can think of two advantages of this structure:

1. The top-level directories, like `src`, `docs`, `tests`, are more logical and extensible because they are on the same level of abstraction. It feels better.

2. If there are non-Python code, such as Python extensions in C++, this structure can easily grow with the complexity of the codebase. An [experimental package of mine](https://github.com/zpz/experiments.py) is an example.

## Naming and style
<a name="naming"></a>

For the project and packages, single-word names (or two words without hyphen) are more desirable.

For functions and variables, multi-word, verboses names are fine.

Use 'snake-case' names; see [examples](http://google.github.io/styleguide/pyguide.html#316-naming).

Do study the ``PEP 8`` style guide; pick and stick to a decent style.

I use `yapf` to format Python code. This frees the developer from thinking about good style,
and help avoids arguments about style between developers. Usually I use the following command:

```
$ yapf -ir -vv --no-local-style ./
```

## Documentation
<a name="documentation"></a>

Do write documentation to help **yourself**, if not others. 

- In-source documentation: make use of Python's docstring mechanism;
  follow the Google style; [example 1](http://sphinxcontrib-napoleon.readthedocs.org/en/latest/example_google.html), [example 2](http://www.sphinx-doc.org/en/stable/ext/example_google.html#example-google).
- Stand-alone documentation files in source tree (but outside of source code files):
  - `README.md` in the top-level (project root) or second-level (package root) 
    directories.
  - Doc files in `doc` directory under the project root directory.
  - Plain text files are perferred. Preferred formats are 'plain text' (`txt`), 'mark-down' (`md`), or 'reStructuredText' (`rst`) (if you intend to do extensive documentation and generate friendly HTML versions using `Sphinx`).
  - If you use Mac, a decent Markdown editor is `MacDown`.
  - To edit `rst` files with live preview, use the `Atom` editor (by Github) with a couple plug-ins.
  - If you use `Sphinx`, `reStructuredText` markup syntax can be used in in-source docstrings as well.
- Outside of source tree: for documentation aimed at non-developers or users from other groups that require different levels of access control.

### How to generate documentation using Sphinx

1. Basically, let Sphinx take over the `doc` directory in the repo. In `doc/`, run

   ```
   sphinx-quickstart
   ```

   Make sure you make these particular choices:

   ```
   Separate source and build directories (y/n) [n]: y
   autodoc: automatically insert docstrings from modules (y/n) [n]: y
   imgmath: include math, rendered as PNG or SVG images (y/n) [n]: n
   mathjax: include math, rendered in the browser by MathJax (y/n) [n]: y
   viewcode: include links to the source code of documented Python objects (y/n) [n]: y
   Create Makefile? (y/n) [y]: y
   ```

   In `doc/source/conf.py`, make sure the `extensions` list contains at least these items:

   * `sphinx.ext.autodoc`
   * `sphinx.ext.inheritance_diagram`
   * `sphinx.ext.mathjax`
   * `sphinx.ext.napoleon`
   * `sphinx.ext.todo`
   * `sphinx.ext.viewcode`

   Also make sure `html_theme` is set to `nature` (unless you have another preference).
   After that, search for `alabaster` in `conf.py`, and comment out that block,
   because that block is needed for the default `alabaster` theme but could get in the way
   of other thems.

2. Create `*.rst` files as needed in `doc/source/`. These stand-alone doc files along with doc in source code will be used to generate documentation.

3. In `doc`, run `make html` to generate HTML documentation, which will be located in `doc/build/html/`. View the documentation in a web browser, starting with the file `doc/build/html/index.html`.

4. `git commit` the files `doc/Makefile`, `doc/source/conf.py`, as well as the `*.rst` and other files you've created in `doc/source`. DO NOT `git commit` the generated material in `doc/build`.


## Logging
<a name="logging"></a>

Prefer logging to `print` in most cases.

- The [twelve factor app](http://12factor.net/logs) advocates treating log events as an event stream, always sending the stream to standard output, and leaving capture of the stream into files to the execution environment.
- In all modules that need to do logging, always have this, and only this, at the top of the module:

  ```python
  import logging
  logger = logging.getLogger(__name__)
  ```

  then use `logger.info()` etc to create log messages.

  Do not create custom names for the logger. The `__name__` mechanism will create a hierarchy of loggers following your
package structure, e.g. with loggers named

  ```
  package1
  package1.module1
  package1.module1.submodule2
  ```

  Log messages will pass to higher levels in this hierarchy.
Customizing the logger names will disrupt this useful message forwarding.

- In the `__init__.py` file in the top-level directory of your Python package,
include this:

  ```python
  # Set default logging handler to avoid "No handler found" warnings.
  import logging
  logging.getLogger(__name__).addHandler(logging.NullHandler())
  ```

  Do not add any other handler in your package code.

- Do not do any format, handler (e.g. using a file handler), log level,
or other configuration in your package code. These belong in the *launch scripts* and should happen
at exactly **one** place in a running program.

  I basically call something like the following function in the launch script.
  Usually ``level`` is the only argument I need to specify.

  ```python
  import logging
  import os
  import time

  def config_logger(level=None, use_utc=True, datefmt=None, format=None, **kwargs):
    # 'level' is a string form of the logging levels: 'debug', 'info', 'warning', 'error', 'critical'.

    if level is None:
      level = os.environ.get('LOGLEVEL', 'info')

    if level not in [logging.DEBUG, logging.INFO, logging.WARNING, logging.ERROR, logging.CRITICAL]:
      level = getattr(logging, level.upper())

    if use_utc:
      logging.Formatter.converter = time.gmtime
      datefmt = datefmt or '%Y-%m-%d %H:%M:%SZ'
    else:
      logging.Formatter.converter = time.localtime
      datefmt = datefmt or '%Y-%m-%d %H:%M:%S'

    format = format or '[%(asctime)s; %(name)s, %(funcName)s, %(lineno)d; %(levelname)s]    %(message)s'

    logging.basicConfig(format=format, datefmt=datefmt, level=level, **kwargs)
  ```

- Use the old-style string formatting (`%`), not the new-style string formatting (`str.format`).
  The following example should serve most of your fomatting needs:

  ```python
  logger.info('Line %s (%s) has spent %.2f by hour %d', 'asd9123las', 'Huge Sale!', 28.97, 23) 
  ```

  [See here](https://pyformat.info) for more about formatting.


## Testing
<a name="testing"></a>


Write tests, and adopt a testing framework. My recommendation is `py.test`.

You do not need to `import pytest` unless you explicitly use `pytest` in the code.

See [Directory Structure](#directory-structure) above for where to put the test files.

(Lightly revised on December 21, 2018.)