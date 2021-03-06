

Basic patterns and examples
==========================================================

Pass different values to a test function, depending on command line options
----------------------------------------------------------------------------

.. regendoc:wipe

Suppose we want to write a test that depends on a command line option.
Here is a basic pattern to achieve this:

.. code-block:: python

    # content of test_sample.py
    def test_answer(cmdopt):
        if cmdopt == "type1":
            print ("first")
        elif cmdopt == "type2":
            print ("second")
        assert 0 # to see what was printed


For this to work we need to add a command line option and
provide the ``cmdopt`` through a :ref:`fixture function <fixture function>`:

.. code-block:: python

    # content of conftest.py
    import pytest

    def pytest_addoption(parser):
        parser.addoption("--cmdopt", action="store", default="type1",
            help="my option: type1 or type2")

    @pytest.fixture
    def cmdopt(request):
        return request.config.getoption("--cmdopt")

Let's run this without supplying our new option::

    $ pytest -q test_sample.py
    F
    ======= FAILURES ========
    _______ test_answer ________
    
    cmdopt = 'type1'
    
        def test_answer(cmdopt):
            if cmdopt == "type1":
                print ("first")
            elif cmdopt == "type2":
                print ("second")
    >       assert 0 # to see what was printed
    E       assert 0
    
    test_sample.py:6: AssertionError
    --------------------------- Captured stdout call ---------------------------
    first
    1 failed in 0.12 seconds

And now with supplying a command line option::

    $ pytest -q --cmdopt=type2
    F
    ======= FAILURES ========
    _______ test_answer ________
    
    cmdopt = 'type2'
    
        def test_answer(cmdopt):
            if cmdopt == "type1":
                print ("first")
            elif cmdopt == "type2":
                print ("second")
    >       assert 0 # to see what was printed
    E       assert 0
    
    test_sample.py:6: AssertionError
    --------------------------- Captured stdout call ---------------------------
    second
    1 failed in 0.12 seconds

You can see that the command line option arrived in our test.  This
completes the basic pattern.  However, one often rather wants to process
command line options outside of the test and rather pass in different or
more complex objects.

Dynamically adding command line options
--------------------------------------------------------------

.. regendoc:wipe

Through :confval:`addopts` you can statically add command line
options for your project.  You can also dynamically modify
the command line arguments before they get processed:

.. code-block:: python

    # content of conftest.py
    import sys
    def pytest_cmdline_preparse(args):
        if 'xdist' in sys.modules: # pytest-xdist plugin
            import multiprocessing
            num = max(multiprocessing.cpu_count() / 2, 1)
            args[:] = ["-n", str(num)] + args

If you have the `xdist plugin <https://pypi.python.org/pypi/pytest-xdist>`_ installed
you will now always perform test runs using a number
of subprocesses close to your CPU. Running in an empty
directory with the above conftest.py::

    $ pytest
    ======= test session starts ========
    platform linux -- Python 3.5.2, pytest-3.0.6, py-1.4.33, pluggy-0.4.0
    rootdir: $REGENDOC_TMPDIR, inifile: 
    collected 0 items
    
    ======= no tests ran in 0.12 seconds ========

.. _`excontrolskip`:

Control skipping of tests according to command line option
--------------------------------------------------------------

.. regendoc:wipe

Here is a ``conftest.py`` file adding a ``--runslow`` command
line option to control skipping of ``slow`` marked tests:

.. code-block:: python

    # content of conftest.py

    import pytest
    def pytest_addoption(parser):
        parser.addoption("--runslow", action="store_true",
            help="run slow tests")

We can now write a test module like this:

.. code-block:: python

    # content of test_module.py
    import pytest


    slow = pytest.mark.skipif(
        not pytest.config.getoption("--runslow"),
        reason="need --runslow option to run"
    )


    def test_func_fast():
        pass


    @slow
    def test_func_slow():
        pass

and when running it will see a skipped "slow" test::

    $ pytest -rs    # "-rs" means report details on the little 's'
    ======= test session starts ========
    platform linux -- Python 3.5.2, pytest-3.0.6, py-1.4.33, pluggy-0.4.0
    rootdir: $REGENDOC_TMPDIR, inifile: 
    collected 2 items
    
    test_module.py .s
    ======= short test summary info ========
    SKIP [1] test_module.py:13: need --runslow option to run
    
    ======= 1 passed, 1 skipped in 0.12 seconds ========

Or run it including the ``slow`` marked test::

    $ pytest --runslow
    ======= test session starts ========
    platform linux -- Python 3.5.2, pytest-3.0.6, py-1.4.33, pluggy-0.4.0
    rootdir: $REGENDOC_TMPDIR, inifile: 
    collected 2 items
    
    test_module.py ..
    
    ======= 2 passed in 0.12 seconds ========

Writing well integrated assertion helpers
--------------------------------------------------

.. regendoc:wipe

If you have a test helper function called from a test you can
use the ``pytest.fail`` marker to fail a test with a certain message.
The test support function will not show up in the traceback if you
set the ``__tracebackhide__`` option somewhere in the helper function.
Example:

.. code-block:: python

    # content of test_checkconfig.py
    import pytest
    def checkconfig(x):
        __tracebackhide__ = True
        if not hasattr(x, "config"):
            pytest.fail("not configured: %s" %(x,))

    def test_something():
        checkconfig(42)

The ``__tracebackhide__`` setting influences ``pytest`` showing
of tracebacks: the ``checkconfig`` function will not be shown
unless the ``--full-trace`` command line option is specified.
Let's run our little function::

    $ pytest -q test_checkconfig.py
    F
    ======= FAILURES ========
    _______ test_something ________
    
        def test_something():
    >       checkconfig(42)
    E       Failed: not configured: 42
    
    test_checkconfig.py:8: Failed
    1 failed in 0.12 seconds

If you only want to hide certain exceptions, you can set ``__tracebackhide__``
to a callable which gets the ``ExceptionInfo`` object. You can for example use
this to make sure unexpected exception types aren't hidden:

.. code-block:: python

    import operator
    import pytest

    class ConfigException(Exception):
        pass

    def checkconfig(x):
        __tracebackhide__ = operator.methodcaller('errisinstance', ConfigException)
        if not hasattr(x, "config"):
            raise ConfigException("not configured: %s" %(x,))

    def test_something():
        checkconfig(42)

This will avoid hiding the exception traceback on unrelated exceptions (i.e.
bugs in assertion helpers).


Detect if running from within a pytest run
--------------------------------------------------------------

.. regendoc:wipe

Usually it is a bad idea to make application code
behave differently if called from a test.  But if you
absolutely must find out if your application code is
running from a test you can do something like this:

.. code-block:: python

    # content of conftest.py

    def pytest_configure(config):
        import sys
        sys._called_from_test = True

    def pytest_unconfigure(config):
        del sys._called_from_test

and then check for the ``sys._called_from_test`` flag:

.. code-block:: python

    if hasattr(sys, '_called_from_test'):
        # called from within a test run
    else:
        # called "normally"

accordingly in your application.  It's also a good idea
to use your own application module rather than ``sys``
for handling flag.

Adding info to test report header
--------------------------------------------------------------

.. regendoc:wipe

It's easy to present extra information in a ``pytest`` run:

.. code-block:: python

    # content of conftest.py

    def pytest_report_header(config):
        return "project deps: mylib-1.1"

which will add the string to the test header accordingly::

    $ pytest
    ======= test session starts ========
    platform linux -- Python 3.5.2, pytest-3.0.6, py-1.4.33, pluggy-0.4.0
    project deps: mylib-1.1
    rootdir: $REGENDOC_TMPDIR, inifile: 
    collected 0 items
    
    ======= no tests ran in 0.12 seconds ========

.. regendoc:wipe

It is also possible to return a list of strings which will be considered as several
lines of information. You may consider ``config.getoption('verbose')`` in order to
display more information if applicable:

.. code-block:: python

    # content of conftest.py

    def pytest_report_header(config):
        if config.getoption('verbose') > 0:
            return ["info1: did you know that ...", "did you?"]

which will add info only when run with "--v"::

    $ pytest -v
    ======= test session starts ========
    platform linux -- Python 3.5.2, pytest-3.0.6, py-1.4.33, pluggy-0.4.0 -- $PYTHON_PREFIX/bin/python3.5m
    cachedir: .cache
    info1: did you know that ...
    did you?
    rootdir: $REGENDOC_TMPDIR, inifile: 
    collecting ... collected 0 items
    
    ======= no tests ran in 0.12 seconds ========

and nothing when run plainly::

    $ pytest
    ======= test session starts ========
    platform linux -- Python 3.5.2, pytest-3.0.6, py-1.4.33, pluggy-0.4.0
    rootdir: $REGENDOC_TMPDIR, inifile: 
    collected 0 items
    
    ======= no tests ran in 0.12 seconds ========

profiling test duration
--------------------------

.. regendoc:wipe

.. versionadded: 2.2

If you have a slow running large test suite you might want to find
out which tests are the slowest. Let's make an artificial test suite:

.. code-block:: python

    # content of test_some_are_slow.py
    import time

    def test_funcfast():
        pass

    def test_funcslow1():
        time.sleep(0.1)

    def test_funcslow2():
        time.sleep(0.2)

Now we can profile which test functions execute the slowest::

    $ pytest --durations=3
    ======= test session starts ========
    platform linux -- Python 3.5.2, pytest-3.0.6, py-1.4.33, pluggy-0.4.0
    rootdir: $REGENDOC_TMPDIR, inifile: 
    collected 3 items
    
    test_some_are_slow.py ...
    
    ======= slowest 3 test durations ========
    0.20s call     test_some_are_slow.py::test_funcslow2
    0.10s call     test_some_are_slow.py::test_funcslow1
    0.00s setup    test_some_are_slow.py::test_funcfast
    ======= 3 passed in 0.12 seconds ========

incremental testing - test steps
---------------------------------------------------

.. regendoc:wipe

Sometimes you may have a testing situation which consists of a series
of test steps.  If one step fails it makes no sense to execute further
steps as they are all expected to fail anyway and their tracebacks
add no insight.  Here is a simple ``conftest.py`` file which introduces
an ``incremental`` marker which is to be used on classes:

.. code-block:: python

    # content of conftest.py

    import pytest

    def pytest_runtest_makereport(item, call):
        if "incremental" in item.keywords:
            if call.excinfo is not None:
                parent = item.parent
                parent._previousfailed = item

    def pytest_runtest_setup(item):
        if "incremental" in item.keywords:
            previousfailed = getattr(item.parent, "_previousfailed", None)
            if previousfailed is not None:
                pytest.xfail("previous test failed (%s)" %previousfailed.name)

These two hook implementations work together to abort incremental-marked
tests in a class.  Here is a test module example:

.. code-block:: python

    # content of test_step.py

    import pytest

    @pytest.mark.incremental
    class TestUserHandling:
        def test_login(self):
            pass
        def test_modification(self):
            assert 0
        def test_deletion(self):
            pass

    def test_normal():
        pass

If we run this::

    $ pytest -rx
    ======= test session starts ========
    platform linux -- Python 3.5.2, pytest-3.0.6, py-1.4.33, pluggy-0.4.0
    rootdir: $REGENDOC_TMPDIR, inifile: 
    collected 4 items
    
    test_step.py .Fx.
    ======= short test summary info ========
    XFAIL test_step.py::TestUserHandling::()::test_deletion
      reason: previous test failed (test_modification)
    
    ======= FAILURES ========
    _______ TestUserHandling.test_modification ________
    
    self = <test_step.TestUserHandling object at 0xdeadbeef>
    
        def test_modification(self):
    >       assert 0
    E       assert 0
    
    test_step.py:9: AssertionError
    ======= 1 failed, 2 passed, 1 xfailed in 0.12 seconds ========

We'll see that ``test_deletion`` was not executed because ``test_modification``
failed.  It is reported as an "expected failure".


Package/Directory-level fixtures (setups)
-------------------------------------------------------

If you have nested test directories, you can have per-directory fixture scopes
by placing fixture functions in a ``conftest.py`` file in that directory
You can use all types of fixtures including :ref:`autouse fixtures
<autouse fixtures>` which are the equivalent of xUnit's setup/teardown
concept.  It's however recommended to have explicit fixture references in your
tests or test classes rather than relying on implicitly executing
setup/teardown functions, especially if they are far away from the actual tests.

Here is an example for making a ``db`` fixture available in a directory:

.. code-block:: python

    # content of a/conftest.py
    import pytest

    class DB:
        pass

    @pytest.fixture(scope="session")
    def db():
        return DB()

and then a test module in that directory:

.. code-block:: python

    # content of a/test_db.py
    def test_a1(db):
        assert 0, db  # to show value

another test module:

.. code-block:: python

    # content of a/test_db2.py
    def test_a2(db):
        assert 0, db  # to show value

and then a module in a sister directory which will not see
the ``db`` fixture:

.. code-block:: python

    # content of b/test_error.py
    def test_root(db):  # no db here, will error out
        pass

We can run this::

    $ pytest
    ======= test session starts ========
    platform linux -- Python 3.5.2, pytest-3.0.6, py-1.4.33, pluggy-0.4.0
    rootdir: $REGENDOC_TMPDIR, inifile: 
    collected 7 items
    
    test_step.py .Fx.
    a/test_db.py F
    a/test_db2.py F
    b/test_error.py E
    
    ======= ERRORS ========
    _______ ERROR at setup of test_root ________
    file $REGENDOC_TMPDIR/b/test_error.py, line 1
      def test_root(db):  # no db here, will error out
    E       fixture 'db' not found
    >       available fixtures: cache, capfd, capsys, doctest_namespace, monkeypatch, pytestconfig, record_xml_property, recwarn, tmpdir, tmpdir_factory
    >       use 'pytest --fixtures [testpath]' for help on them.
    
    $REGENDOC_TMPDIR/b/test_error.py:1
    ======= FAILURES ========
    _______ TestUserHandling.test_modification ________
    
    self = <test_step.TestUserHandling object at 0xdeadbeef>
    
        def test_modification(self):
    >       assert 0
    E       assert 0
    
    test_step.py:9: AssertionError
    _______ test_a1 ________
    
    db = <conftest.DB object at 0xdeadbeef>
    
        def test_a1(db):
    >       assert 0, db  # to show value
    E       AssertionError: <conftest.DB object at 0xdeadbeef>
    E       assert 0
    
    a/test_db.py:2: AssertionError
    _______ test_a2 ________
    
    db = <conftest.DB object at 0xdeadbeef>
    
        def test_a2(db):
    >       assert 0, db  # to show value
    E       AssertionError: <conftest.DB object at 0xdeadbeef>
    E       assert 0
    
    a/test_db2.py:2: AssertionError
    ======= 3 failed, 2 passed, 1 xfailed, 1 error in 0.12 seconds ========

The two test modules in the ``a`` directory see the same ``db`` fixture instance
while the one test in the sister-directory ``b`` doesn't see it.  We could of course
also define a ``db`` fixture in that sister directory's ``conftest.py`` file.
Note that each fixture is only instantiated if there is a test actually needing
it (unless you use "autouse" fixture which are always executed ahead of the first test
executing).


post-process test reports / failures
---------------------------------------

If you want to postprocess test reports and need access to the executing
environment you can implement a hook that gets called when the test
"report" object is about to be created.  Here we write out all failing
test calls and also access a fixture (if it was used by the test) in
case you want to query/look at it during your post processing.  In our
case we just write some information out to a ``failures`` file:

.. code-block:: python

    # content of conftest.py

    import pytest
    import os.path

    @pytest.hookimpl(tryfirst=True, hookwrapper=True)
    def pytest_runtest_makereport(item, call):
        # execute all other hooks to obtain the report object
        outcome = yield
        rep = outcome.get_result()

        # we only look at actual failing test calls, not setup/teardown
        if rep.when == "call" and rep.failed:
            mode = "a" if os.path.exists("failures") else "w"
            with open("failures", mode) as f:
                # let's also access a fixture for the fun of it
                if "tmpdir" in item.fixturenames:
                    extra = " (%s)" % item.funcargs["tmpdir"]
                else:
                    extra = ""

                f.write(rep.nodeid + extra + "\n")


if you then have failing tests:

.. code-block:: python

    # content of test_module.py
    def test_fail1(tmpdir):
        assert 0
    def test_fail2():
        assert 0

and run them::

    $ pytest test_module.py
    ======= test session starts ========
    platform linux -- Python 3.5.2, pytest-3.0.6, py-1.4.33, pluggy-0.4.0
    rootdir: $REGENDOC_TMPDIR, inifile: 
    collected 2 items
    
    test_module.py FF
    
    ======= FAILURES ========
    _______ test_fail1 ________
    
    tmpdir = local('PYTEST_TMPDIR/test_fail10')
    
        def test_fail1(tmpdir):
    >       assert 0
    E       assert 0
    
    test_module.py:2: AssertionError
    _______ test_fail2 ________
    
        def test_fail2():
    >       assert 0
    E       assert 0
    
    test_module.py:4: AssertionError
    ======= 2 failed in 0.12 seconds ========

you will have a "failures" file which contains the failing test ids::

    $ cat failures
    test_module.py::test_fail1 (PYTEST_TMPDIR/test_fail10)
    test_module.py::test_fail2

Making test result information available in fixtures
-----------------------------------------------------------

.. regendoc:wipe

If you want to make test result reports available in fixture finalizers
here is a little example implemented via a local plugin:

.. code-block:: python

    # content of conftest.py

    import pytest

    @pytest.hookimpl(tryfirst=True, hookwrapper=True)
    def pytest_runtest_makereport(item, call):
        # execute all other hooks to obtain the report object
        outcome = yield
        rep = outcome.get_result()

        # set a report attribute for each phase of a call, which can
        # be "setup", "call", "teardown"

        setattr(item, "rep_" + rep.when, rep)


    @pytest.fixture
    def something(request):
        yield
        # request.node is an "item" because we use the default
        # "function" scope
        if request.node.rep_setup.failed:
            print ("setting up a test failed!", request.node.nodeid)
        elif request.node.rep_setup.passed:
            if request.node.rep_call.failed:
                print ("executing test failed", request.node.nodeid)


if you then have failing tests:

.. code-block:: python

    # content of test_module.py

    import pytest

    @pytest.fixture
    def other():
        assert 0

    def test_setup_fails(something, other):
        pass

    def test_call_fails(something):
        assert 0

    def test_fail2():
        assert 0

and run it::

    $ pytest -s test_module.py
    ======= test session starts ========
    platform linux -- Python 3.5.2, pytest-3.0.6, py-1.4.33, pluggy-0.4.0
    rootdir: $REGENDOC_TMPDIR, inifile: 
    collected 3 items
    
    test_module.py Esetting up a test failed! test_module.py::test_setup_fails
    Fexecuting test failed test_module.py::test_call_fails
    F
    
    ======= ERRORS ========
    _______ ERROR at setup of test_setup_fails ________
    
        @pytest.fixture
        def other():
    >       assert 0
    E       assert 0
    
    test_module.py:6: AssertionError
    ======= FAILURES ========
    _______ test_call_fails ________
    
    something = None
    
        def test_call_fails(something):
    >       assert 0
    E       assert 0
    
    test_module.py:12: AssertionError
    _______ test_fail2 ________
    
        def test_fail2():
    >       assert 0
    E       assert 0
    
    test_module.py:15: AssertionError
    ======= 2 failed, 1 error in 0.12 seconds ========

You'll see that the fixture finalizers could use the precise reporting
information.

Freezing pytest 
---------------

If you freeze your application using a tool like
`PyInstaller <https://pyinstaller.readthedocs.io>`_
in order to distribute it to your end-users, it is a good idea to also package
your test runner and run your tests using the frozen application. This way packaging
errors such as dependencies not being included into the executable can be detected early
while also allowing you to send test files to users so they can run them in their
machines, which can be useful to obtain more information about a hard to reproduce bug.

Fortunately recent ``PyInstaller`` releases already have a custom hook
for pytest, but if you are using another tool to freeze executables 
such as ``cx_freeze`` or ``py2exe``, you can use ``pytest.freeze_includes()``
to obtain the full list of internal pytest modules. How to configure the tools
to find the internal modules varies from tool to tool, however.

Instead of freezing the pytest runner as a separate executable, you can make 
your frozen program work as the pytest runner by some clever
argument handling during program startup. This allows you to 
have a single executable, which is usually more convenient.

.. code-block:: python

    # contents of app_main.py
    import sys

    if len(sys.argv) > 1 and sys.argv[1] == '--pytest':
        import pytest
        sys.exit(pytest.main(sys.argv[2:]))
    else:
        # normal application execution: at this point argv can be parsed
        # by your argument-parsing library of choice as usual
        ...


This allows you to execute tests using the frozen
application with standard ``pytest`` command-line options::

    ./app_main --pytest --verbose --tb=long --junitxml=results.xml test-suite/
