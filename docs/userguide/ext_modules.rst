==========================
Building Extension Modules
==========================

Setuptools can build C/C++ extension modules.  The keyword argument
``ext_modules`` of ``setup()`` should be a list of instances of the
:class:`setuptools.Extension` class.

Overview
========

To understand how this works, let us consider a very simple project with
only one extension module.
This project will be named ``calculate``, and will provide C functions to add
and subtract two integers.
The directory structure of this project could be as follows::

    <project_folder>
    ├── add.c
    ├── calculate
    │   └── __init__.py
    ├── pyproject.toml
    └── subtract.c

Here, ``pyproject.toml`` contains a basic project metadata configuration:

.. code-block:: toml

   # pyproject.toml
   [build-system]
   requires = ["setuptools"]
   build-backend = "setuptools.build_meta"

   [project]
   name = "calculate"  # as it would appear on PyPI
   version = "0.42"

The C code is contained in two files ``add.c`` and ``subtract.c``, and could
be as follows:

.. code-block:: c

    // add.c
    int add(int a, int b) {
        return a + b;
    }

    // subtract.c
    int subtract(int a, int b) {
        return a - b;
    }

Now, this C code needs to be compiled into a dynamic library in order to enable
the Python interpreter to link to it at runtime. A simple way to carry out this
compilation (on Linux systems) is to run:

.. code-block:: bash

   $ gcc -shared -o libcalculate.so add.c subtract.c

This will create a dynamic library ``libcalculate.so`` inside ``<project_folder>``.

Coming to the Python side, let us say all we want to do is import the ``add()``
and ``subtract()`` functions of the ``libcalculate`` library, and make them
available from within the ``calculate`` package namespace, i.e. in
``calculate/__init__.py`` we write:

.. code-block:: python

   from libcalculate import add, subtract

However, this import will not work. And that is because regular C code cannot
integrate with Python code "just like that". Python exposes a certain C API, and
C code has to work with and conform to that API in order to integrate with Python
code. To make our C code integrate with Python code, we can do something like the
following:

.. code-block:: c

   // add.h
   int add(int a, int b);

   // subtract.h
   int subtract(int a, int b);

   // calculate.c

   #include <Python.h>
   #include "add.h"
   #include "subtract.h"
   
   static PyObject *libcalculate_add(PyObject *self, PyObject *args) {
       int a, b;
       if (!PyArg_ParseTuple(args, "ii", &a, &b))
           return NULL;
       int result = add(a, b);
       return PyLong_FromLong(result);
   }
   
   static PyObject *libcalculate_subtract(PyObject *self, PyObject *args) {
       int a, b;
       if (!PyArg_ParseTuple(args, "ii", &a, &b))
           return NULL;
       int result = subtract(a, b);
       return PyLong_FromLong(result);
   }
   
   static PyMethodDef LibcalculateMethods[] = {
       {"add", libcalculate_add, METH_VARARGS, "Add two integers."},
       {"subtract", libcalculate_subtract, METH_VARARGS, "Subtract two integers."},
       {NULL, NULL, 0, NULL},
   };
   
   static struct PyModuleDef libcalculate_module = {
       PyModuleDef_HEAD_INIT,
       "libcalculate",
       "Simple calculation module.",
       -1,
       LibcalculateMethods,
   };
   
   PyMODINIT_FUNC
   PyInit_libcalculate(void) {
       return PyModule_Create(&libcalculate_module);
   }

Basically, along with two header files ``add.h`` and ``subtract.h``, we have created
a new file ``calculate.c`` which wraps around the C functions ``add()`` and ``subtract()``
and registers them as valid methods of a Python module named ``libcalculate``.

.. note::
   Writing C extensions for the CPython interpreter is an involved topic. As such, it is
   beyond the scope of this guide. To understand more about the code in ``calculate.c``
   above, you may refer to the `Python docs about C/C++ extensions`_.

After these changes, the compilation command has to be slightly modified:

.. code-block:: bash

   $ gcc -shared -o libcalculate.so -I/usr/include/python3.10 \
        add.c calculate.c subtract.c

Here ``/usr/include/python3.10`` represents the path to the ``Python.h`` header file
(if you are using a Unix system and the Python version is 3.10), the file exposing
Python's C API.

At this point, after compiling the library, you can open up a Python shell inside
``<project_folder>`` and verify that everything works:

.. code-block:: pycon

   >>> import calculate
   >>> calculate.add(3, 5)
   8
   >>> calculate.subtract(3, 5)
   -2

Our discussion so far illustrates the entire process of integrating C code with
Python code. So where does Setuptools fit into the process? Basically,
Setuptools facilitates an easier process of compiling the C code, i.e. the C
extension module(s), and shipping it along with the Python code.

To register extension modules with Setuptools, they must be configured via a
``setup.py`` file. For our example, we can use the following configuration:

.. code-block:: python

   from setuptools import setup, Extension

   libcalculate_ext = Extension(
       name='libcalculate',
       sources=['add.c', 'calculate.c', 'subtract.c'])

   setup(ext_modules=[libcalculate_ext])

Basically, each extension module in your package must be configured as an
instance of the :class:`setuptools.Extension` class, and then, you must pass to
the ``setup()`` function a keyword argument ``ext_modules`` with a list of
these instances.

.. tip::
   Note that it is currently not possible to configure extension modules via
   ``setup.cfg`` or ``pyproject.toml``, so you must use ``setup.py``. A preferred
   arrangement is to use a minimal ``setup.py`` file: in other words, your
   ``setup.py`` should only contain configuration pertaining to extension
   modules, while all other metadata/configuration pertaining to the project
   should be stored in a ``setup.cfg`` or ``pyproject.toml`` file.

.. note::
   The ``name`` of the extension specifies the name with which it will be
   imported on the Python side. As such, it can include a dot ``.`` to denote
   packages/namespaces. For example

   .. code-block:: python

      libcalculate_ext = Extension(
          name='calculate.libcalculate',
          ...,
      )

   is perfectly valid, and then the code in ``calculate/__init__.py`` will
   correspondingly change to (note the relative import):

   .. code-block:: python

      from .libcalculate import add, subtract

.. note::

   The ``sources`` argument of the extension must list all of your C source
   code files. Note that you must not include any header files i.e. files
   ending in ``.h``, or this will give you an error on building the extension.
   For example if we try to add the header file ``add.h`` as follows:

   .. code-block:: python

      libcalculate_ext = Extension(
          ...,
          sources=['add.c', 'add.h', 'calculate.c', 'subtract.c'])

   then when Setuptools tries to build the extension, it will produce the
   following error::

       error: unknown file type '.h' (from 'add.h')

   Another point is that if you would like to use globs, you must use the
   standard library :mod:`glob` module to expand the globs prior to passing
   them as ``sources``. For example, this will not work:

   .. code-block:: python

      libcalculate_ext = Extension(
          ...,
          sources=['*.c'])

   but this will:

   .. code-block:: python

      import glob
      libcalculate_ext = Extension(
          ...,
          sources=[file for file in glob.glob('*.c')])

Now, with this configuration in place, whenever you build a wheel for your
package:

.. code-block:: bash

   $ python -m build .
   # or
   $ python -m build . --wheel

compilation of the extension module(s) will be automatically triggered, i.e.
you do not have to write out the ``gcc`` command yourself, and the compiled
library will be appropriately inserted into the resulting wheel.

Even during development, when you run an editable install:

.. code-block:: bash

   $ pip install -e .

the extension module(s) will be compiled and placed in an appropriate location
such that it is accessible to the Python interpreter.

.. tip::

   Earlier, when it was standard practice to run commands via ``setup.py``,
   it was possible to run the command:

   .. code-block:: bash

      $ python setup.py build_ext

   to build all extension modules. However, running this command is no longer
   recommended. To recompile extension modules and bring into effect any changes
   made to your non-Python source code, you are advised to simply *re-run* the
   editable install command:

   .. code-block:: bash

      $ pip install -e .

.. seealso::
   You can find more information on the `Python docs about C/C++ extensions`_.
   Alternatively, you might also be interested in learn about `Cython`_.

   If you plan to distribute a package that uses extensions across multiple
   platforms, :pypi:`cibuildwheel` can also be helpful.

.. important::
   All files used to compile your extension need to be available on the system
   when building the package, so please make sure to include some documentation
   on how developers interested in building your package from source
   can obtain operating system level dependencies
   (e.g. compilers and external binary libraries/artifacts).

   You will also need to make sure that all auxiliary files that are contained
   inside your :term:`project` (e.g. C headers authored by you or your team)
   are configured to be included in your :term:`sdist <Source Distribution (or "sdist")>`.
   Please have a look on our section on :ref:`Controlling files in the distribution`.


Compiler and linker options
===========================

The command ``build_ext`` builds C/C++ extension modules.  It creates
a command line for running the compiler and linker by combining
compiler and linker options from various sources:

.. Reference: `test_customize_compiler` in distutils/tests/test_sysconfig.py

* the ``sysconfig`` variables ``CC``, ``CXX``, ``CCSHARED``,
  ``LDSHARED``, and ``CFLAGS``,
* the environment variables ``CC``, ``CPP``,
  ``CXX``, ``LDSHARED`` and ``LDFLAGS``,
  ``CFLAGS``, ``CPPFLAGS``, ``LDFLAGS``,
* the ``Extension`` attributes ``include_dirs``,
  ``library_dirs``, ``extra_compile_args``, ``extra_link_args``,
  ``runtime_library_dirs``.

.. Ignoring AR, ARFLAGS, RANLIB here because they are used by the (obsolete?) build_clib, not build_ext.

Specifically, if the environment variables ``CC``, ``CPP``, ``CXX``, and ``LDSHARED``
are set, they will be used instead of the ``sysconfig`` variables of the same names.

The compiler options appear in the command line in the following order:

.. Reference: "compiler_so" and distutils.ccompiler.gen_preprocess_options, CCompiler.compile, UnixCCompiler._compile

* first, the options provided by the ``sysconfig`` variable ``CFLAGS``,
* then, the options provided by the environment variables ``CFLAGS`` and ``CPPFLAGS``,
* then, the options provided by the ``sysconfig`` variable ``CCSHARED``,
* then, a ``-I`` option for each element of ``Extension.include_dirs``,
* finally, the options provided by ``Extension.extra_compile_args``.

The linker options appear in the command line in the following order:

.. Reference: "linker_so" and CCompiler.link

* first, the options provided by environment variables and ``sysconfig`` variables,
* then, a ``-L`` option for each element of ``Extension.library_dirs``,
* then, a linker-specific option like ``-Wl,-rpath`` for each element of ``Extension.runtime_library_dirs``,
* finally, the options provided by ``Extension.extra_link_args``.

The resulting command line is then processed by the compiler and linker.
According to the GCC manual sections on `directory options`_ and
`environment variables`_, the C/C++ compiler searches for files named in
``#include <file>`` directives in the following order:

* first, in directories given by ``-I`` options (in left-to-right order),
* then, in directories given by the environment variable ``CPATH`` (in left-to-right order),
* then, in directories given by ``-isystem`` options (in left-to-right order),
* then, in directories given by the environment variable ``C_INCLUDE_PATH`` (for C) and ``CPLUS_INCLUDE_PATH`` (for C++),
* then, in standard system directories,
* finally, in directories given by ``-idirafter`` options (in left-to-right order).

The linker searches for libraries in the following order:

* first, in directories given by ``-L`` options (in left-to-right order),
* then, in directories given by the environment variable ``LIBRARY_PATH`` (in left-to-right order).


Distributing Extensions compiled with Cython
============================================

When your :pypi:`Cython` extension modules *are declared using the*
:class:`setuptools.Extension` *class*, ``setuptools`` will detect at build time
whether Cython is installed or not.

If Cython is present, then ``setuptools`` will use it to build the ``.pyx`` files.
Otherwise, ``setuptools`` will try to find and compile the equivalent ``.c`` files
(instead of ``.pyx``). These files can be generated using the
`cython command line tool`_.

You can ensure that Cython is always automatically installed into the build
environment by including it as a :ref:`build dependency <build-requires>` in
your ``pyproject.toml``:

.. code-block:: toml

    [build-system]
    requires = [..., "cython"]

Alternatively, you can include the ``.c`` code that is pre-compiled by Cython
into your source distribution, alongside the original ``.pyx`` files (this
might save a few seconds when building from an ``sdist``).
To improve version compatibility, you probably also want to include current
``.c`` files in your :wiki:`revision control system`, and rebuild them whenever
you check changes in for the ``.pyx`` source files.
This will ensure that people tracking your project will be able to build it
without installing Cython, and that there will be no variation due to small
differences in the generate C files.
Please checkout our docs on :ref:`controlling files in the distribution` for
more information.

----

Extension API Reference
=======================

.. autoclass:: setuptools.Extension


.. _Python docs about C/C++ extensions: https://docs.python.org/3/extending/extending.html
.. _Cython: https://cython.readthedocs.io/en/stable/index.html
.. _directory options: https://gcc.gnu.org/onlinedocs/gcc/Directory-Options.html
.. _environment variables: https://gcc.gnu.org/onlinedocs/gcc/Environment-Variables.html>
.. _cython command line tool: https://cython.readthedocs.io/en/stable/src/userguide/source_files_and_compilation.html
