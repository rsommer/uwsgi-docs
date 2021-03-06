The PyPy plugin
===============

Requires uWSGI >= 2.0.9

Introduction
^^^^^^^^^^^^

Idea/Design: Maciej Fijalkowski

Contributors: Alex Gaynor, Armin Rigo

A new PyPy plugin based on cffi is available since uWSGI 1.9.11. The old slow cpyext-based one has been removed from the tree.

The plugin is currently supported only on Linux systems. Following releases will support other platforms as well.

The plugin loads ``libpypy-c.so`` on startup, sets the home of the PyPy installation and executes a special Python module
implementing the plugin logic. So yes, most of the plugin is implemented in Python, and theoretically this approach will allow
writing uWSGI plugins directly in Python in addition to C, C++ and Objective-C.

As of December 2014 all of the required patches to PyPy have been merged, so you can get an official nightly build (or a stable version released after december 2014)
and use it with uWSGI.


Install uWSGI with PyPy support
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

As always with uWSGI, you have different ways to install uWSGI based on your needs.

If you have installed pip in your PyPy home, you can run

.. code-block:: sh

   pip install uwsgi
  
The uwsgi setup.py file will recognize the PyPy environment and will build a PyPy-only uWSGI binary.

In the same way, you can execute the setup.py supplied in uWSGI sources:

.. code-block:: sh

    pypy setup.py install
    

(this two approaches will hardcode the pypy home in the uWSGI binary, so you will not need to set pypy-home in your options)


Or you can compile manually:

.. code-block:: sh

   UWSGI_PROFILE=pypy make
   
Or you can use the network installer:

.. code-block:: sh

   curl http://uwsgi.it/install | bash -s pypy /tmp/uwsgi
   
This will build a uWSGI + PyPy binary in ``/tmp/uwsgi``.

Or you can build PyPy support as a plugin.

.. code-block:: sh

   uwsgi --build-plugin plugins/pypy

or (old-style)

.. code-block:: sh

   python uwsgiconfig.py --plugin plugins/pypy
   
The PyPy home
^^^^^^^^^^^^^

The uWSGI Python plugin (more exactly the CPython plugin) works by linking in ``libpython``. That means you need to rebuild the plugin for every different version of Python. The PyPy plugin is different, as libpypy-c is loaded on startup and its symbols are resolved at runtime. This allows you to migrate to a different PyPy version on the fly.

The "downside" of this approach is that you need to inform uWSGI where your PyPy installation is at runtime (unless you installed uwsgi via pip or with the setup.py script, in such a case the home will be found automatically)

Supposing your PyPy is in ``/opt/pypy`` you can start uWSGI with:

.. code-block:: sh

   uwsgi --http-socket :9090 --pypy-home /opt/pypy
  
With this command line uWSGI will search for ``/opt/pypy/bin/libpypy-c.so`` and if found, it will set that path as the PyPy home.

If your ``libpypy-c.so`` is outside of the PyPy home (and in a directory not reachable by the dynamic linker), you can use the ``--pypy-lib``option.

.. code-block:: sh

   uwsgi --http-socket :9090 --pypy-home /opt/pypy --pypy-lib /opt/libs/libpypy-c.so
   
With this approach you are able to use the library from a specific PyPy build and the home from another one.

.. note:: Remember to prefix --pypy-lib with ./ if you want to point to a .so file in your current directory!

The PyPy setup file
^^^^^^^^^^^^^^^^^^^

As said before, most of the uWSGI PyPy plugin is written in Python. This code is loaded at runtime, and you can also customize it.

Yes, this does mean you can change the way the plugin works without rebuilding uWSGI.

A default version of the ``pypy_setup.py`` file is embedded in the uWSGI binary, and it is automatically loaded on startup.

If you want to change it, just pass another filename via the ``--pypy-setup`` option.

.. code-block:: sh

   uwsgi --http-socket :9090 --pypy-home /opt/pypy --pypy-lib /opt/libs/libpypy-c.so --pypy-setup /home/foobar/foo.py
   
This Python module implements uWSGI hooks and the virtual ``uwsgi`` python module for accessing the uWSGI API from your apps.

If you want to retrieve the contents of the embedded pypy_setup.py file you can read it from the binary symbols with the ``print-sym`` convenience option.

.. code-block:: sh

   uwsgi --print-sym uwsgi_pypy_setup

WSGI support
^^^^^^^^^^^^

The plugin implements PEP 333 and PEP 3333. You can load both WSGI modules and ``mod_wsgi`` style ``.wsgi`` files.

To load a WSGI module (it must be in your Python path):

.. code-block:: sh

   uwsgi --http-socket :9090 --pypy-home /opt/pypy --pypy-wsgi myapp
   
To load a WSGI file:

.. code-block:: sh

   uwsgi --http-socket :9090 --pypy-home /opt/pypy --pypy-wsgi-file /var/www/myapp/myapp.wsgi
   
RPC support
^^^^^^^^^^^

You can register RPC functions using the :func:`uwsgi.register_rpc` API function, like you would with the vanilla Python plugin.

.. code-block:: py

   import uwsgi
   
   def hello():
       return "Hello World"
       
   uwsgi.register_rpc('hello', hello)
   
To call RPC functions, both :func:`uwsgi.rpc` and :func:`uwsgi.call` are available.

.. code-block:: py

   import uwsgi
   
   uwsgi.rpc('192.168.173.100:3031', 'myfunc', 'myarg')
   uwsgi.call('myfunc', 'myarg')
   uwsgi.call('myfunc@192.168.173.100:3031', 'myarg')
   
   
Integration (with local RPC) has been tested between PyPy and PyPy, PyPy and JVM, and PyPy and Lua. All of these worked flawlessly... so that means you can call Java functions from PyPy.

IPython trick
^^^^^^^^^^^^^

Having a runtime shell for making tests is very nice to have. You can use IPython for this.

.. code-block:: sh

   uwsgi --socket :3031 --pypy-home /opt/pypy --pypy-eval "import IPython; IPython.embed()" --honour-stdin
   
   
uWSGI API status
^^^^^^^^^^^^^^^^

The following API functions, hooks and attributes are supported as of 20130526.

* :py:data:`uwsgi.opt`
* :py:data:`uwsgi.post_fork_hook`
* :func:`uwsgi.add_cron()`
* :func:`uwsgi.setprocname()`
* :func:`uwsgi.alarm()`
* :func:`uwsgi.signal_registered()`
* :func:`uwsgi.mule_id()`
* :func:`uwsgi.worker_id()`
* :func:`uwsgi.masterpid()`
* :func:`uwsgi.lock()`
* :func:`uwsgi.unlock()`
* :func:`uwsgi.add_file_monitor()`
* :func:`uwsgi.add_timer()`
* :func:`uwsgi.add_rb_timer()`
* :func:`uwsgi.cache_get()`
* :func:`uwsgi.cache_set()`
* :func:`uwsgi.cache_update()`
* :func:`uwsgi.cache_del()`
* :func:`uwsgi.signal()`
* :func:`uwsgi.call()`
* :func:`uwsgi.rpc()`
* :func:`uwsgi.register_rpc()`
* :func:`uwsgi.register_signal()`
  
Options
^^^^^^^


* ``pypy-lib`` - load the specified libpypy-s.so
* ``pypy-setup`` - load the specified pypy_setup script file
* ``pypy-home`` - set the pypy home
* ``pypy-wsgi`` - load a WSGI module
* ``pypy-wsgi-file`` - load a mod_wsgi compatible .wsgi file
* ``pypy-eval`` - execute the specified string before ``fork()``
* ``pypy-eval-post-fork`` - execute the specified string after each ``fork()``
* ``pypy-exec`` - execute the specified python script before ``fork()``
* ``pypy-exec-post-fork`` - execute the specified python script after each ``fork()``
* ``pypy-pp/pypy-python-path/pypy-pythonpath`` - add the specified item to the pythonpath
* ``pypy-paste`` - load a paste.deploy .ini configuration
* ``pypy-ini-paste`` - load a paste.deploy .ini configuration and use its [uwsgi] section


Notes
^^^^^

* Mixing libpython with libpypy-c is explicitly forbidden. A check in the pypy plugin prevents you from doing such a hellish thing.
* The PyPy plugin is generally somewhat more "orthodox" from a Python programmer point of view, while the CPython one may be a little blasphemous in many areas. We have been able to make that choice as we do not need backward compatibility with older uWSGI releases.
* The uWSGI API is still incomplete.
* The WSGI loader does not update the uWSGI internal application list, so things like ``--need-app`` will not work. The server will report "dynamic mode" on startup even if the app has been successfully loaded. This will be fixed soon.
