:mod:`unittest` --- Unit testing framework
==========================================

.. module:: unittest
   :synopsis: Unit testing framework for Python.
.. moduleauthor:: Steve Purcell <stephen_purcell@yahoo.com>
.. sectionauthor:: Steve Purcell <stephen_purcell@yahoo.com>
.. sectionauthor:: Fred L. Drake, Jr. <fdrake@acm.org>
.. sectionauthor:: Raymond Hettinger <python@rcn.com>


.. versionchanged:: 3.1
   Added test :ref:`skipping and expected failures <unittest-skipping>`.

The Python unit testing framework, sometimes referred to as "PyUnit," is a
Python language version of JUnit, by Kent Beck and Erich Gamma. JUnit is, in
turn, a Java version of Kent's Smalltalk testing framework.  Each is the de
facto standard unit testing framework for its respective language.

:mod:`unittest` supports test automation, sharing of setup and shutdown code for
tests, aggregation of tests into collections, and independence of the tests from
the reporting framework.  The :mod:`unittest` module provides classes that make
it easy to support these qualities for a set of tests.

To achieve this, :mod:`unittest` supports some important concepts:

test fixture
   A :dfn:`test fixture` represents the preparation needed to perform one or more
   tests, and any associate cleanup actions.  This may involve, for example,
   creating temporary or proxy databases, directories, or starting a server
   process.

test case
   A :dfn:`test case` is the smallest unit of testing.  It checks for a specific
   response to a particular set of inputs.  :mod:`unittest` provides a base class,
   :class:`TestCase`, which may be used to create new test cases.

test suite
   A :dfn:`test suite` is a collection of test cases, test suites, or both.  It is
   used to aggregate tests that should be executed together.

test runner
   A :dfn:`test runner` is a component which orchestrates the execution of tests
   and provides the outcome to the user.  The runner may use a graphical interface,
   a textual interface, or return a special value to indicate the results of
   executing the tests.

The test case and test fixture concepts are supported through the
:class:`TestCase` and :class:`FunctionTestCase` classes; the former should be
used when creating new tests, and the latter can be used when integrating
existing test code with a :mod:`unittest`\ -driven framework. When building test
fixtures using :class:`TestCase`, the :meth:`~TestCase.setUp` and
:meth:`~TestCase.tearDown` methods can be overridden to provide initialization
and cleanup for the fixture.  With :class:`FunctionTestCase`, existing functions
can be passed to the constructor for these purposes.  When the test is run, the
fixture initialization is run first; if it succeeds, the cleanup method is run
after the test has been executed, regardless of the outcome of the test.  Each
instance of the :class:`TestCase` will only be used to run a single test method,
so a new fixture is created for each test.

Test suites are implemented by the :class:`TestSuite` class.  This class allows
individual tests and test suites to be aggregated; when the suite is executed,
all tests added directly to the suite and in "child" test suites are run.

A test runner is an object that provides a single method,
:meth:`~TestRunner.run`, which accepts a :class:`TestCase` or :class:`TestSuite`
object as a parameter, and returns a result object.  The class
:class:`TestResult` is provided for use as the result object. :mod:`unittest`
provides the :class:`TextTestRunner` as an example test runner which reports
test results on the standard error stream by default.  Alternate runners can be
implemented for other environments (such as graphical environments) without any
need to derive from a specific class.


.. seealso::

   Module :mod:`doctest`
      Another test-support module with a very different flavor.

   `Simple Smalltalk Testing: With Patterns <http://www.XProgramming.com/testfram.htm>`_
      Kent Beck's original paper on testing frameworks using the pattern shared by
      :mod:`unittest`.

   `Nose <http://code.google.com/p/python-nose/>`_ and `py.test <http://pytest.org>`_
      Third-party unittest frameworks with a lighter-weight syntax
      for writing tests.  For example, ``assert func(10) == 42``.

   `python-mock <http://python-mock.sourceforge.net/>`_ and `minimock <http://blog.ianbicking.org/minimock.html>`_
      Tools for creating mock test objects (objects simulating external resources).

.. _unittest-minimal-example:

Basic example
-------------

The :mod:`unittest` module provides a rich set of tools for constructing and
running tests.  This section demonstrates that a small subset of the tools
suffice to meet the needs of most users.

Here is a short script to test three functions from the :mod:`random` module::

   import random
   import unittest

   class TestSequenceFunctions(unittest.TestCase):

       def setUp(self):
           self.seq = list(range(10))

       def test_shuffle(self):
           # make sure the shuffled sequence does not lose any elements
           random.shuffle(self.seq)
           self.seq.sort()
           self.assertEqual(self.seq, list(range(10)))

       def test_choice(self):
           element = random.choice(self.seq)
           self.assertIn(element, self.seq)

       def test_sample(self):
           self.assertRaises(ValueError, random.sample, self.seq, 20)
           for element in random.sample(self.seq, 5):
               self.assertIn(element, self.seq)

   if __name__ == '__main__':
       unittest.main()

A testcase is created by subclassing :class:`unittest.TestCase`.  The three
individual tests are defined with methods whose names start with the letters
``test``.  This naming convention informs the test runner about which methods
represent tests.

The crux of each test is a call to :meth:`~TestCase.assertEqual` to check for an
expected result; :meth:`~TestCase.assert_` to verify a condition; or
:meth:`~TestCase.assertRaises` to verify that an expected exception gets raised.
These methods are used instead of the :keyword:`assert` statement so the test
runner can accumulate all test results and produce a report.

When a :meth:`~TestCase.setUp` method is defined, the test runner will run that
method prior to each test.  Likewise, if a :meth:`~TestCase.tearDown` method is
defined, the test runner will invoke that method after each test.  In the
example, :meth:`~TestCase.setUp` was used to create a fresh sequence for each
test.

The final block shows a simple way to run the tests. :func:`unittest.main`
provides a command line interface to the test script.  When run from the command
line, the above script produces an output that looks like this::

   ...
   ----------------------------------------------------------------------
   Ran 3 tests in 0.000s

   OK

Instead of :func:`unittest.main`, there are other ways to run the tests with a
finer level of control, less terse output, and no requirement to be run from the
command line.  For example, the last two lines may be replaced with::

   suite = unittest.TestLoader().loadTestsFromTestCase(TestSequenceFunctions)
   unittest.TextTestRunner(verbosity=2).run(suite)

Running the revised script from the interpreter or another script produces the
following output::

   test_choice (__main__.TestSequenceFunctions) ... ok
   test_sample (__main__.TestSequenceFunctions) ... ok
   test_shuffle (__main__.TestSequenceFunctions) ... ok

   ----------------------------------------------------------------------
   Ran 3 tests in 0.110s

   OK

The above examples show the most commonly used :mod:`unittest` features which
are sufficient to meet many everyday testing needs.  The remainder of the
documentation explores the full feature set from first principles.


.. _organizing-tests:

Organizing test code
--------------------

The basic building blocks of unit testing are :dfn:`test cases` --- single
scenarios that must be set up and checked for correctness.  In :mod:`unittest`,
test cases are represented by instances of :mod:`unittest`'s :class:`TestCase`
class. To make your own test cases you must write subclasses of
:class:`TestCase`, or use :class:`FunctionTestCase`.

An instance of a :class:`TestCase`\ -derived class is an object that can
completely run a single test method, together with optional set-up and tidy-up
code.

The testing code of a :class:`TestCase` instance should be entirely self
contained, such that it can be run either in isolation or in arbitrary
combination with any number of other test cases.

The simplest :class:`TestCase` subclass will simply override the
:meth:`~TestCase.runTest` method in order to perform specific testing code::

   import unittest

   class DefaultWidgetSizeTestCase(unittest.TestCase):
       def runTest(self):
           widget = Widget('The widget')
           self.assertEqual(widget.size(), (50, 50), 'incorrect default size')

Note that in order to test something, we use the one of the :meth:`assert\*`
methods provided by the :class:`TestCase` base class.  If the
test fails, an exception will be raised, and :mod:`unittest` will identify the
test case as a :dfn:`failure`.  Any other exceptions will be treated as
:dfn:`errors`. This helps you identify where the problem is: :dfn:`failures` are
caused by incorrect results - a 5 where you expected a 6. :dfn:`Errors` are
caused by incorrect code - e.g., a :exc:`TypeError` caused by an incorrect
function call.

The way to run a test case will be described later.  For now, note that to
construct an instance of such a test case, we call its constructor without
arguments::

   testCase = DefaultWidgetSizeTestCase()

Now, such test cases can be numerous, and their set-up can be repetitive.  In
the above case, constructing a :class:`Widget` in each of 100 Widget test case
subclasses would mean unsightly duplication.

Luckily, we can factor out such set-up code by implementing a method called
:meth:`~TestCase.setUp`, which the testing framework will automatically call for
us when we run the test::

   import unittest

   class SimpleWidgetTestCase(unittest.TestCase):
       def setUp(self):
           self.widget = Widget('The widget')

   class DefaultWidgetSizeTestCase(SimpleWidgetTestCase):
       def runTest(self):
           self.assertEqual(self.widget.size(), (50,50),
                            'incorrect default size')

   class WidgetResizeTestCase(SimpleWidgetTestCase):
       def runTest(self):
           self.widget.resize(100,150)
           self.assertEqual(self.widget.size(), (100,150),
                            'wrong size after resize')

If the :meth:`~TestCase.setUp` method raises an exception while the test is
running, the framework will consider the test to have suffered an error, and the
:meth:`~TestCase.runTest` method will not be executed.

Similarly, we can provide a :meth:`~TestCase.tearDown` method that tidies up
after the :meth:`~TestCase.runTest` method has been run::

   import unittest

   class SimpleWidgetTestCase(unittest.TestCase):
       def setUp(self):
           self.widget = Widget('The widget')

       def tearDown(self):
           self.widget.dispose()
           self.widget = None

If :meth:`~TestCase.setUp` succeeded, the :meth:`~TestCase.tearDown` method will
be run whether :meth:`~TestCase.runTest` succeeded or not.

Such a working environment for the testing code is called a :dfn:`fixture`.

Often, many small test cases will use the same fixture.  In this case, we would
end up subclassing :class:`SimpleWidgetTestCase` into many small one-method
classes such as :class:`DefaultWidgetSizeTestCase`.  This is time-consuming and
discouraging, so in the same vein as JUnit, :mod:`unittest` provides a simpler
mechanism::

   import unittest

   class WidgetTestCase(unittest.TestCase):
       def setUp(self):
           self.widget = Widget('The widget')

       def tearDown(self):
           self.widget.dispose()
           self.widget = None

       def test_default_size(self):
           self.assertEqual(self.widget.size(), (50,50),
                            'incorrect default size')

       def test_resize(self):
           self.widget.resize(100,150)
           self.assertEqual(self.widget.size(), (100,150),
                            'wrong size after resize')

Here we have not provided a :meth:`~TestCase.runTest` method, but have instead
provided two different test methods.  Class instances will now each run one of
the :meth:`test_\*` methods, with ``self.widget`` created and destroyed
separately for each instance.  When creating an instance we must specify the
test method it is to run.  We do this by passing the method name in the
constructor::

   defaultSizeTestCase = WidgetTestCase('test_default_size')
   resizeTestCase = WidgetTestCase('test_resize')

Test case instances are grouped together according to the features they test.
:mod:`unittest` provides a mechanism for this: the :dfn:`test suite`,
represented by :mod:`unittest`'s :class:`TestSuite` class::

   widgetTestSuite = unittest.TestSuite()
   widgetTestSuite.addTest(WidgetTestCase('test_default_size'))
   widgetTestSuite.addTest(WidgetTestCase('test_resize'))

For the ease of running tests, as we will see later, it is a good idea to
provide in each test module a callable object that returns a pre-built test
suite::

   def suite():
       suite = unittest.TestSuite()
       suite.addTest(WidgetTestCase('test_default_size'))
       suite.addTest(WidgetTestCase('test_resize'))
       return suite

or even::

   def suite():
       tests = ['test_default_size', 'test_resize']

       return unittest.TestSuite(map(WidgetTestCase, tests))

Since it is a common pattern to create a :class:`TestCase` subclass with many
similarly named test functions, :mod:`unittest` provides a :class:`TestLoader`
class that can be used to automate the process of creating a test suite and
populating it with individual tests. For example, ::

   suite = unittest.TestLoader().loadTestsFromTestCase(WidgetTestCase)

will create a test suite that will run ``WidgetTestCase.test_default_size()`` and
``WidgetTestCase.test_resize``. :class:`TestLoader` uses the ``'test'`` method
name prefix to identify test methods automatically.

Note that the order in which the various test cases will be run is
determined by sorting the test function names with respect to the
built-in ordering for strings.

Often it is desirable to group suites of test cases together, so as to run tests
for the whole system at once.  This is easy, since :class:`TestSuite` instances
can be added to a :class:`TestSuite` just as :class:`TestCase` instances can be
added to a :class:`TestSuite`::

   suite1 = module1.TheTestSuite()
   suite2 = module2.TheTestSuite()
   alltests = unittest.TestSuite([suite1, suite2])

You can place the definitions of test cases and test suites in the same modules
as the code they are to test (such as :file:`widget.py`), but there are several
advantages to placing the test code in a separate module, such as
:file:`test_widget.py`:

* The test module can be run standalone from the command line.

* The test code can more easily be separated from shipped code.

* There is less temptation to change test code to fit the code it tests without
  a good reason.

* Test code should be modified much less frequently than the code it tests.

* Tested code can be refactored more easily.

* Tests for modules written in C must be in separate modules anyway, so why not
  be consistent?

* If the testing strategy changes, there is no need to change the source code.


.. _legacy-unit-tests:

Re-using old test code
----------------------

Some users will find that they have existing test code that they would like to
run from :mod:`unittest`, without converting every old test function to a
:class:`TestCase` subclass.

For this reason, :mod:`unittest` provides a :class:`FunctionTestCase` class.
This subclass of :class:`TestCase` can be used to wrap an existing test
function.  Set-up and tear-down functions can also be provided.

Given the following test function::

   def testSomething():
       something = makeSomething()
       assert something.name is not None
       # ...

one can create an equivalent test case instance as follows::

   testcase = unittest.FunctionTestCase(testSomething)

If there are additional set-up and tear-down methods that should be called as
part of the test case's operation, they can also be provided like so::

   testcase = unittest.FunctionTestCase(testSomething,
                                        setUp=makeSomethingDB,
                                        tearDown=deleteSomethingDB)

To make migrating existing test suites easier, :mod:`unittest` supports tests
raising :exc:`AssertionError` to indicate test failure. However, it is
recommended that you use the explicit :meth:`TestCase.fail\*` and
:meth:`TestCase.assert\*` methods instead, as future versions of :mod:`unittest`
may treat :exc:`AssertionError` differently.

.. note::

   Even though :class:`FunctionTestCase` can be used to quickly convert an existing
   test base over to a :mod:`unittest`\ -based system, this approach is not
   recommended.  Taking the time to set up proper :class:`TestCase` subclasses will
   make future test refactorings infinitely easier.

In some cases, the existing tests may have been written using the :mod:`doctest`
module.  If so, :mod:`doctest` provides a :class:`DocTestSuite` class that can
automatically build :class:`unittest.TestSuite` instances from the existing
:mod:`doctest`\ -based tests.


.. _unittest-skipping:

Skipping tests and expected failures
------------------------------------

Unittest supports skipping individual test methods and even whole classes of
tests.  In addition, it supports marking a test as a "expected failure," a test
that is broken and will fail, but shouldn't be counted as a failure on a
:class:`TestResult`.

Skipping a test is simply a matter of using the :func:`skip` :term:`decorator`
or one of its conditional variants.

Basic skipping looks like this: ::

   class MyTestCase(unittest.TestCase):

       @unittest.skip("demonstrating skipping")
       def test_nothing(self):
           self.fail("shouldn't happen")

       @unittest.skipIf(mylib.__version__ < (1, 3), "not supported in this library version")
       def test_format(self):
           # Tests that work for only a certain version of the library.
           pass

       @unittest.skipUnless(sys.platform.startswith("win"), "requires Windows")
       def test_windows_support(self):
           # windows specific testing code
           pass

This is the output of running the example above in verbose mode: ::

   test_format (__main__.MyTestCase) ... skipped 'not supported in this library version'
   test_nothing (__main__.MyTestCase) ... skipped 'demonstrating skipping'
   test_windows_support (__main__.MyTestCase) ... skipped 'requires Windows'

   ----------------------------------------------------------------------
   Ran 3 tests in 0.005s

   OK (skipped=3)

Classes can be skipped just like methods: ::

   @skip("showing class skipping")
   class MySkippedTestCase(unittest.TestCase):
       def test_not_run(self):
           pass

:meth:`TestCase.setUp` can also skip the test.  This is useful when a resource
that needs to be set up is not available.

Expected failures use the :func:`expectedFailure` decorator. ::

   class ExpectedFailureTestCase(unittest.TestCase):
       @unittest.expectedFailure
       def test_fail(self):
           self.assertEqual(1, 0, "broken")

It's easy to roll your own skipping decorators by making a decorator that calls
:func:`skip` on the test when it wants it to be skipped.  This decorator skips
the test unless the passed object has a certain attribute: ::

   def skipUnlessHasattr(obj, attr):
       if hasattr(obj, attr):
           return lambda func: func
       return unittest.skip("{0!r} doesn't have {1!r}".format(obj, attr))

The following decorators implement test skipping and expected failures:

.. function:: skip(reason)

   Unconditionally skip the decorated test.  *reason* should describe why the
   test is being skipped.

.. function:: skipIf(condition, reason)

   Skip the decorated test if *condition* is true.

.. function:: skipUnless(condition, reason)

   Skip the decorated test unless *condition* is true.

.. function:: expectedFailure

   Mark the test as an expected failure.  If the test fails when run, the test
   is not counted as a failure.


.. _unittest-contents:

Classes and functions
---------------------

This section describes in depth the API of :mod:`unittest`.


.. _testcase-objects:

Test cases
~~~~~~~~~~

.. class:: TestCase(methodName='runTest')

   Instances of the :class:`TestCase` class represent the smallest testable units
   in the :mod:`unittest` universe.  This class is intended to be used as a base
   class, with specific tests being implemented by concrete subclasses.  This class
   implements the interface needed by the test runner to allow it to drive the
   test, and methods that the test code can use to check for and report various
   kinds of failure.

   Each instance of :class:`TestCase` will run a single test method: the method
   named *methodName*.  If you remember, we had an earlier example that went
   something like this::

      def suite():
          suite = unittest.TestSuite()
          suite.addTest(WidgetTestCase('test_default_size'))
          suite.addTest(WidgetTestCase('test_resize'))
          return suite

   Here, we create two instances of :class:`WidgetTestCase`, each of which runs a
   single test.

   *methodName* defaults to :meth:`runTest`.

   :class:`TestCase` instances provide three groups of methods: one group used
   to run the test, another used by the test implementation to check conditions
   and report failures, and some inquiry methods allowing information about the
   test itself to be gathered.

   Methods in the first group (running the test) are:


   .. method:: setUp()

      Method called to prepare the test fixture.  This is called immediately
      before calling the test method; any exception raised by this method will
      be considered an error rather than a test failure. The default
      implementation does nothing.


   .. method:: tearDown()

      Method called immediately after the test method has been called and the
      result recorded.  This is called even if the test method raised an
      exception, so the implementation in subclasses may need to be particularly
      careful about checking internal state.  Any exception raised by this
      method will be considered an error rather than a test failure.  This
      method will only be called if the :meth:`setUp` succeeds, regardless of
      the outcome of the test method. The default implementation does nothing.


   .. method:: run(result=None)

      Run the test, collecting the result into the test result object passed as
      *result*.  If *result* is omitted or :const:`None`, a temporary result
      object is created (by calling the :meth:`defaultTestResult` method) and
      used. The result object is not returned to :meth:`run`'s caller.

      The same effect may be had by simply calling the :class:`TestCase`
      instance.


   .. method:: skipTest(reason)

      Calling this during a test method or :meth:`setUp` skips the current
      test.  See :ref:`unittest-skipping` for more information.

      .. versionadded:: 3.1


   .. method:: debug()

      Run the test without collecting the result.  This allows exceptions raised
      by the test to be propagated to the caller, and can be used to support
      running tests under a debugger.

   The test code can use any of the following methods to check for and report
   failures.


   .. method:: assertTrue(expr, msg=None)
               assert_(expr, msg=None)
               failUnless(expr, msg=None)

      Signal a test failure if *expr* is false; the explanation for the failure
      will be *msg* if given, otherwise it will be :const:`None`.

      .. deprecated:: 3.1
         :meth:`failUnless`.


   .. method:: assertEqual(first, second, msg=None)
               failUnlessEqual(first, second, msg=None)

      Test that *first* and *second* are equal.  If the values do not compare
      equal, the test will fail with the explanation given by *msg*, or
      :const:`None`.  Note that using :meth:`assertEqual` improves upon
      doing the comparison as the first parameter to :meth:`assertTrue`: the
      default value for *msg* include representations of both *first* and
      *second*.

      In addition, if *first* and *second* are the exact same type and one of
      list, tuple, dict, set, or frozenset or any type that a subclass
      registers :meth:`addTypeEqualityFunc` the type specific equality function
      will be called in order to generate a more useful default error message.

      .. versionchanged:: 3.1
         Added the automatic calling of type specific equality function.

      .. deprecated:: 3.1
         :meth:`failUnlessEqual`.


   .. method:: assertNotEqual(first, second, msg=None)
               failIfEqual(first, second, msg=None)

      Test that *first* and *second* are not equal.  If the values do compare
      equal, the test will fail with the explanation given by *msg*, or
      :const:`None`.  Note that using :meth:`assertNotEqual` improves upon doing
      the comparison as the first parameter to :meth:`assertTrue` is that the
      default value for *msg* can be computed to include representations of both
      *first* and *second*.

      .. deprecated:: 3.1
         :meth:`failIfEqual`.


   .. method:: assertAlmostEqual(first, second, places=7, msg=None)
               failUnlessAlmostEqual(first, second, places=7, msg=None)

      Test that *first* and *second* are approximately equal by computing the
      difference, rounding to the given number of decimal *places* (default 7),
      and comparing to zero.

      Note that comparing a given number of decimal places is not the same as
      comparing a given number of significant digits. If the values do not
      compare equal, the test will fail with the explanation given by *msg*, or
      :const:`None`.

      .. deprecated:: 3.1
         :meth:`failUnlessAlmostEqual`.


   .. method:: assertNotAlmostEqual(first, second, places=7, msg=None)
               failIfAlmostEqual(first, second, places=7, msg=None)

      Test that *first* and *second* are not approximately equal by computing
      the difference, rounding to the given number of decimal *places* (default
      7), and comparing to zero.

      Note that comparing a given number of decimal places is not the same as
      comparing a given number of significant digits. If the values do not
      compare equal, the test will fail with the explanation given by *msg*, or
      :const:`None`.

      .. deprecated:: 3.1
         :meth:`failIfAlmostEqual`.


   .. method:: assertGreater(first, second, msg=None)
               assertGreaterEqual(first, second, msg=None)
               assertLess(first, second, msg=None)
               assertLessEqual(first, second, msg=None)

      Test that *first* is respectively >, >=, < or <= than *second* depending
      on the method name.  If not, the test will fail with an explanation
      or with the explanation given by *msg*::

         >>> self.assertGreaterEqual(3, 4)
         AssertionError: "3" unexpectedly not greater than or equal to "4"

      .. versionadded:: 3.1


   .. method:: assertMultiLineEqual(self, first, second, msg=None)

      Test that the multiline string *first* is equal to the string *second*.
      When not equal a diff of the two strings highlighting the differences
      will be included in the error message.

      If specified *msg* will be used as the error message on failure.

      .. versionadded:: 3.1


   .. method:: assertRegexpMatches(text, regexp, msg=None):

      Verifies that a *regexp* search matches *text*.  Fails with an error
      message including the pattern and the *text*.  *regexp* may be
      a regular expression object or a string containing a regular expression
      suitable for use by :func:`re.search`.

      .. versionadded:: 3.1


   .. method:: assertIn(first, second, msg=None)
               assertNotIn(first, second, msg=None)

      Tests that *first* is or is not in *second* with an explanatory error
      message as appropriate.

      If specified *msg* will be used as the error message on failure.

      .. versionadded:: 3.1


   .. method:: assertSameElements(actual, expected, msg=None)

      Test that sequence *expected* contains the same elements as *actual*,
      regardless of their order. When they don't, an error message listing
      the differences between the sequences will be generated.

      Duplicate elements are ignored when comparing *actual* and *expected*.
      It is the equivalent of ``assertEqual(set(expected), set(actual))``
      but it works with sequences of unhashable objects as well.

      If specified *msg* will be used as the error message on failure.

      .. versionadded:: 3.1


   .. method:: assertSetEqual(set1, set2, msg=None)

      Tests that two sets are equal.  If not, an error message is constructed
      that lists the differences between the sets.

      Fails if either of *set1* or *set2* does not have a :meth:`set.difference`
      method.

      If specified *msg* will be used as the error message on failure.

      .. versionadded:: 3.1


   .. method:: assertDictEqual(expected, actual, msg=None)

      Test that two dictionaries are equal.  If not, an error message is
      constructed that shows the differences in the dictionaries.

      If specified *msg* will be used as the error message on failure.

      .. versionadded:: 3.1


   .. method:: assertDictContainsSubset(expected, actual, msg=None)

      Tests whether the key/value pairs in dictionary *actual* are a
      superset of those in *expected*.  If not, an error message listing
      the missing keys and mismatched values is generated.

      If specified *msg* will be used as the error message on failure.

      .. versionadded:: 3.1


   .. method:: assertListEqual(list1, list2, msg=None)
               assertTupleEqual(tuple1, tuple2, msg=None)

      Tests that two lists or tuples are equal.  If not an error message is
      constructed that shows only the differences between the two.  An error
      is also raised if either of the parameters are of the wrong type.

      If specified *msg* will be used as the error message on failure.

      .. versionadded:: 3.1


   .. method:: assertSequenceEqual(seq1, seq2, msg=None, seq_type=None)

      Tests that two sequences are equal.  If a *seq_type* is supplied, both
      *seq1* and *seq2* must be instances of *seq_type* or a failure will
      be raised.  If the sequences are different an error message is
      constructed that shows the difference between the two.

      If specified *msg* will be used as the error message on failure.

      This method is used to implement :meth:`assertListEqual` and
      :meth:`assertTupleEqual`.

      .. versionadded:: 3.1


   .. method:: assertRaises(exception, callable, *args, **kwds)
               failUnlessRaises(exception, callable, *args, **kwds)
               assertRaises(exception)
               failUnlessRaises(exception)

      Test that an exception is raised when *callable* is called with any
      positional or keyword arguments that are also passed to
      :meth:`assertRaises`.  The test passes if *exception* is raised, is an
      error if another exception is raised, or fails if no exception is raised.
      To catch any of a group of exceptions, a tuple containing the exception
      classes may be passed as *exception*.

      If only the *exception* argument is given, returns a context manager so
      that the code under test can be written inline rather than as a function::

         with self.assertRaises(SomeException):
             do_something()

      .. versionchanged:: 3.1
         Added the ability to use :meth:`assertRaises` as a context manager.

      .. deprecated:: 3.1
         :meth:`failUnlessRaises`.


   .. method:: assertRaisesRegexp(exception, regexp[, callable, ...])

      Like :meth:`assertRaises` but also tests that *regexp* matches
      on the string representation of the raised exception.  *regexp* may be
      a regular expression object or a string containing a regular expression
      suitable for use by :func:`re.search`.  Examples::

         self.assertRaisesRegexp(ValueError, 'invalid literal for.*XYZ$',
                                 int, 'XYZ')

      or::

         with self.assertRaisesRegexp(ValueError, 'literal'):
            int('XYZ')

      .. versionadded:: 3.1


   .. method:: assertIsNone(expr, msg=None)

      This signals a test failure if *expr* is not None.

      .. versionadded:: 3.1


   .. method:: assertIsNotNone(expr, msg=None)

      The inverse of the :meth:`assertIsNone` method.
      This signals a test failure if *expr* is None.

      .. versionadded:: 3.1


   .. method:: assertIs(expr1, expr2, msg=None)

      This signals a test failure if *expr1* and *expr2* don't evaluate to the same
      object.

      .. versionadded:: 3.1


   .. method:: assertIsNot(expr1, expr2, msg=None)

      The inverse of the :meth:`assertIs` method.
      This signals a test failure if *expr1* and *expr2* evaluate to the same
      object.

      .. versionadded:: 3.1


   .. method:: assertFalse(expr, msg=None)
               failIf(expr, msg=None)

      The inverse of the :meth:`assertTrue` method is the :meth:`assertFalse` method.
      This signals a test failure if *expr* is true, with *msg* or :const:`None`
      for the error message.

      .. deprecated:: 3.1
         :meth:`failIf`.


   .. method:: fail(msg=None)

      Signals a test failure unconditionally, with *msg* or :const:`None` for
      the error message.


   .. attribute:: failureException

      This class attribute gives the exception raised by the test method.  If a
      test framework needs to use a specialized exception, possibly to carry
      additional information, it must subclass this exception in order to "play
      fair" with the framework.  The initial value of this attribute is
      :exc:`AssertionError`.


   .. attribute:: longMessage

      If set to True then any explicit failure message you pass in to the
      assert methods will be appended to the end of the normal failure message.
      The normal messages contain useful information about the objects involved,
      for example the message from assertEqual shows you the repr of the two
      unequal objects. Setting this attribute to True allows you to have a
      custom error message in addition to the normal one.

      This attribute defaults to False, meaning that a custom message passed
      to an assert method will silence the normal message.

      The class setting can be overridden in individual tests by assigning an
      instance attribute to True or False before calling the assert methods.

      .. versionadded:: 3.1


   Testing frameworks can use the following methods to collect information on
   the test:


   .. method:: countTestCases()

      Return the number of tests represented by this test object.  For
      :class:`TestCase` instances, this will always be ``1``.


   .. method:: defaultTestResult()

      Return an instance of the test result class that should be used for this
      test case class (if no other result instance is provided to the
      :meth:`run` method).

      For :class:`TestCase` instances, this will always be an instance of
      :class:`TestResult`; subclasses of :class:`TestCase` should override this
      as necessary.


   .. method:: id()

      Return a string identifying the specific test case.  This is usually the
      full name of the test method, including the module and class name.


   .. method:: shortDescription()

      Returns a description of the test, or :const:`None` if no description
      has been provided.  The default implementation of this method
      returns the first line of the test method's docstring, if available,
      along with the method name.

      .. versionchanged:: 3.1
         In earlier versions this only returned the first line of the test
         method's docstring, if available or the :const:`None`.  That led to
         undesirable behavior of not printing the test name when someone was
         thoughtful enough to write a docstring.


   .. method:: addTypeEqualityFunc(typeobj, function)

      Registers a type specific :meth:`assertEqual` equality checking
      function to be called by :meth:`assertEqual` when both objects it has
      been asked to compare are exactly *typeobj* (not subclasses).
      *function* must take two positional arguments and a third msg=None
      keyword argument just as :meth:`assertEqual` does.  It must raise
      ``self.failureException`` when inequality between the first two
      parameters is detected.

      One good use of custom equality checking functions for a type
      is to raise ``self.failureException`` with an error message useful
      for debugging the problem by explaining the inequalities in detail.

      .. versionadded:: 3.1


   .. method:: addCleanup(function, *args, **kwargs)

      Add a function to be called after :meth:`tearDown` to cleanup resources
      used during the test. Functions will be called in reverse order to the
      order they are added (LIFO). They are called with any arguments and
      keyword arguments passed into :meth:`addCleanup` when they are
      added.

      If :meth:`setUp` fails, meaning that :meth:`tearDown` is not called,
      then any cleanup functions added will still be called.

      .. versionadded:: 3.2


   .. method:: doCleanups()

      This method is called unconditionally after :meth:`tearDown`, or
      after :meth:`setUp` if :meth:`setUp` raises an exception.

      It is responsible for calling all the cleanup functions added by
      :meth:`addCleanup`. If you need cleanup functions to be called
      *prior* to :meth:`tearDown` then you can call :meth:`doCleanups`
      yourself.

      :meth:`doCleanups` pops methods off the stack of cleanup
      functions one at a time, so it can be called at any time.

      .. versionadded:: 3.2


.. class:: FunctionTestCase(testFunc, setUp=None, tearDown=None, description=None)

   This class implements the portion of the :class:`TestCase` interface which
   allows the test runner to drive the test, but does not provide the methods which
   test code can use to check and report errors. This is used to create test cases
   using legacy test code, allowing it to be integrated into a :mod:`unittest`\
   -based test framework.


.. _testsuite-objects:

Grouping tests
~~~~~~~~~~~~~~

.. class:: TestSuite(tests=())

   This class represents an aggregation of individual tests cases and test suites.
   The class presents the interface needed by the test runner to allow it to be run
   as any other test case.  Running a :class:`TestSuite` instance is the same as
   iterating over the suite, running each test individually.

   If *tests* is given, it must be an iterable of individual test cases or other
   test suites that will be used to build the suite initially. Additional methods
   are provided to add test cases and suites to the collection later on.

   :class:`TestSuite` objects behave much like :class:`TestCase` objects, except
   they do not actually implement a test.  Instead, they are used to aggregate
   tests into groups of tests that should be run together. Some additional
   methods are available to add tests to :class:`TestSuite` instances:


   .. method:: TestSuite.addTest(test)

      Add a :class:`TestCase` or :class:`TestSuite` to the suite.


   .. method:: TestSuite.addTests(tests)

      Add all the tests from an iterable of :class:`TestCase` and :class:`TestSuite`
      instances to this test suite.

      This is equivalent to iterating over *tests*, calling :meth:`addTest` for each
      element.

   :class:`TestSuite` shares the following methods with :class:`TestCase`:


   .. method:: run(result)

      Run the tests associated with this suite, collecting the result into the
      test result object passed as *result*.  Note that unlike
      :meth:`TestCase.run`, :meth:`TestSuite.run` requires the result object to
      be passed in.


   .. method:: debug()

      Run the tests associated with this suite without collecting the
      result. This allows exceptions raised by the test to be propagated to the
      caller and can be used to support running tests under a debugger.


   .. method:: countTestCases()

      Return the number of tests represented by this test object, including all
      individual tests and sub-suites.


   .. method:: __iter__()

      Tests grouped by a :class:`TestSuite` are always accessed by iteration.
      Subclasses can lazily provide tests by overriding :meth:`__iter__`. Note
      that this method maybe called several times on a single suite
      (for example when counting tests or comparing for equality)
      so the tests returned must be the same for repeated iterations.

      .. versionchanged:: 3.2
         In earlier versions the :class:`TestSuite` accessed tests directly rather
         than through iteration, so overriding :meth:`__iter__` wasn't sufficient
         for providing tests.

   In the typical usage of a :class:`TestSuite` object, the :meth:`run` method
   is invoked by a :class:`TestRunner` rather than by the end-user test harness.


Loading and running tests
~~~~~~~~~~~~~~~~~~~~~~~~~

.. class:: TestLoader()

   The :class:`TestLoader` class is used to create test suites from classes and
   modules.  Normally, there is no need to create an instance of this class; the
   :mod:`unittest` module provides an instance that can be shared as
   ``unittest.defaultTestLoader``. Using a subclass or instance, however, allows
   customization of some configurable properties.

   :class:`TestLoader` objects have the following methods:


   .. method:: loadTestsFromTestCase(testCaseClass)

      Return a suite of all tests cases contained in the :class:`TestCase`\ -derived
      :class:`testCaseClass`.


   .. method:: loadTestsFromModule(module)

      Return a suite of all tests cases contained in the given module. This
      method searches *module* for classes derived from :class:`TestCase` and
      creates an instance of the class for each test method defined for the
      class.

      .. note::

         While using a hierarchy of :class:`TestCase`\ -derived classes can be
         convenient in sharing fixtures and helper functions, defining test
         methods on base classes that are not intended to be instantiated
         directly does not play well with this method.  Doing so, however, can
         be useful when the fixtures are different and defined in subclasses.


   .. method:: loadTestsFromName(name, module=None)

      Return a suite of all tests cases given a string specifier.

      The specifier *name* is a "dotted name" that may resolve either to a
      module, a test case class, a test method within a test case class, a
      :class:`TestSuite` instance, or a callable object which returns a
      :class:`TestCase` or :class:`TestSuite` instance.  These checks are
      applied in the order listed here; that is, a method on a possible test
      case class will be picked up as "a test method within a test case class",
      rather than "a callable object".

      For example, if you have a module :mod:`SampleTests` containing a
      :class:`TestCase`\ -derived class :class:`SampleTestCase` with three test
      methods (:meth:`test_one`, :meth:`test_two`, and :meth:`test_three`), the
      specifier ``'SampleTests.SampleTestCase'`` would cause this method to return a
      suite which will run all three test methods.  Using the specifier
      ``'SampleTests.SampleTestCase.test_two'`` would cause it to return a test suite
      which will run only the :meth:`test_two` test method.  The specifier can refer
      to modules and packages which have not been imported; they will be imported as a
      side-effect.

      The method optionally resolves *name* relative to the given *module*.


   .. method:: loadTestsFromNames(names, module=None)

      Similar to :meth:`loadTestsFromName`, but takes a sequence of names rather
      than a single name.  The return value is a test suite which supports all
      the tests defined for each name.


   .. method:: getTestCaseNames(testCaseClass)

      Return a sorted sequence of method names found within *testCaseClass*;
      this should be a subclass of :class:`TestCase`.

   The following attributes of a :class:`TestLoader` can be configured either by
   subclassing or assignment on an instance:


   .. attribute:: testMethodPrefix

      String giving the prefix of method names which will be interpreted as test
      methods.  The default value is ``'test'``.

      This affects :meth:`getTestCaseNames` and all the :meth:`loadTestsFrom\*`
      methods.


   .. attribute:: sortTestMethodsUsing

      Function to be used to compare method names when sorting them in
      :meth:`getTestCaseNames` and all the :meth:`loadTestsFrom\*` methods.


   .. attribute:: suiteClass

      Callable object that constructs a test suite from a list of tests. No
      methods on the resulting object are needed.  The default value is the
      :class:`TestSuite` class.

      This affects all the :meth:`loadTestsFrom\*` methods.


.. class:: TestResult

   This class is used to compile information about which tests have succeeded
   and which have failed.

   A :class:`TestResult` object stores the results of a set of tests.  The
   :class:`TestCase` and :class:`TestSuite` classes ensure that results are
   properly recorded; test authors do not need to worry about recording the
   outcome of tests.

   Testing frameworks built on top of :mod:`unittest` may want access to the
   :class:`TestResult` object generated by running a set of tests for reporting
   purposes; a :class:`TestResult` instance is returned by the
   :meth:`TestRunner.run` method for this purpose.

   :class:`TestResult` instances have the following attributes that will be of
   interest when inspecting the results of running a set of tests:


   .. attribute:: errors

      A list containing 2-tuples of :class:`TestCase` instances and strings
      holding formatted tracebacks. Each tuple represents a test which raised an
      unexpected exception.

   .. attribute:: failures

      A list containing 2-tuples of :class:`TestCase` instances and strings
      holding formatted tracebacks. Each tuple represents a test where a failure
      was explicitly signalled using the :meth:`TestCase.fail\*` or
      :meth:`TestCase.assert\*` methods.

   .. attribute:: skipped

      A list containing 2-tuples of :class:`TestCase` instances and strings
      holding the reason for skipping the test.

      .. versionadded:: 3.1

   .. attribute:: expectedFailures

      A list containing 2-tuples of :class:`TestCase` instances and strings
      holding formatted tracebacks.  Each tuple represents an expected failure
      of the test case.

   .. attribute:: unexpectedSuccesses

      A list containing :class:`TestCase` instances that were marked as expected
      failures, but succeeded.

   .. attribute:: shouldStop

      Set to ``True`` when the execution of tests should stop by :meth:`stop`.


   .. attribute:: testsRun

      The total number of tests run so far.


   .. method:: wasSuccessful()

      Return :const:`True` if all tests run so far have passed, otherwise returns
      :const:`False`.


   .. method:: stop()

      This method can be called to signal that the set of tests being run should
      be aborted by setting the :attr:`shouldStop` attribute to :const:`True`.
      :class:`TestRunner` objects should respect this flag and return without
      running any additional tests.

      For example, this feature is used by the :class:`TextTestRunner` class to
      stop the test framework when the user signals an interrupt from the
      keyboard.  Interactive tools which provide :class:`TestRunner`
      implementations can use this in a similar manner.

   The following methods of the :class:`TestResult` class are used to maintain
   the internal data structures, and may be extended in subclasses to support
   additional reporting requirements.  This is particularly useful in building
   tools which support interactive reporting while tests are being run.


   .. method:: startTest(test)

      Called when the test case *test* is about to be run.

      The default implementation simply increments the instance's :attr:`testsRun`
      counter.


   .. method:: stopTest(test)

      Called after the test case *test* has been executed, regardless of the
      outcome.

      The default implementation does nothing.


   .. method:: startTestRun(test)

      Called once before any tests are executed.

      .. versionadded:: 3.2


   .. method:: stopTestRun(test)

      Called once before any tests are executed.

      .. versionadded:: 3.2


   .. method:: addError(test, err)

      Called when the test case *test* raises an unexpected exception *err* is a
      tuple of the form returned by :func:`sys.exc_info`: ``(type, value,
      traceback)``.

      The default implementation appends a tuple ``(test, formatted_err)`` to
      the instance's :attr:`errors` attribute, where *formatted_err* is a
      formatted traceback derived from *err*.


   .. method:: addFailure(test, err)

      Called when the test case *test* signals a failure. *err* is a tuple of the form
      returned by :func:`sys.exc_info`:  ``(type, value, traceback)``.

      The default implementation appends a tuple ``(test, formatted_err)`` to
      the instance's :attr:`failures` attribute, where *formatted_err* is a
      formatted traceback derived from *err*.


   .. method:: addSuccess(test)

      Called when the test case *test* succeeds.

      The default implementation does nothing.


   .. method:: addSkip(test, reason)

      Called when the test case *test* is skipped.  *reason* is the reason the
      test gave for skipping.

      The default implementation appends a tuple ``(test, reason)`` to the
      instance's :attr:`skipped` attribute.


   .. method:: addExpectedFailure(test, err)

      Called when the test case *test* fails, but was marked with the
      :func:`expectedFailure` decorator.

      The default implementation appends a tuple ``(test, formatted_err)`` to
      the instance's :attr:`expectedFailures` attribute, where *formatted_err*
      is a formatted traceback derived from *err*.


   .. method:: addUnexpectedSuccess(test)

      Called when the test case *test* was marked with the
      :func:`expectedFailure` decorator, but succeeded.

      The default implementation appends the test to the instance's
      :attr:`unexpectedSuccesses` attribute.


.. data:: defaultTestLoader

   Instance of the :class:`TestLoader` class intended to be shared.  If no
   customization of the :class:`TestLoader` is needed, this instance can be used
   instead of repeatedly creating new instances.


.. class:: TextTestRunner(stream=sys.stderr, descriptions=True, verbosity=1)

   A basic test runner implementation which prints results on standard error.  It
   has a few configurable parameters, but is essentially very simple.  Graphical
   applications which run test suites should provide alternate implementations.

   .. method:: _makeResult()

      This method returns the instance of ``TestResult`` used by :meth:`run`.
      It is not intended to be called directly, but can be overridden in
      subclasses to provide a custom ``TestResult``.


.. function:: main(module='__main__', defaultTest=None, argv=None, testRunner=TextTestRunner, testLoader=unittest.defaultTestLoader, exit=True)

   A command-line program that runs a set of tests; this is primarily for making
   test modules conveniently executable.  The simplest use for this function is to
   include the following line at the end of a test script::

      if __name__ == '__main__':
          unittest.main()

   The *testRunner* argument can either be a test runner class or an already
   created instance of it. By default ``main`` calls :func:`sys.exit` with
   an exit code indicating success or failure of the tests run.

   ``main`` supports being used from the interactive interpreter by passing in the
   argument ``exit=False``. This displays the result on standard output without
   calling :func:`sys.exit`::

      >>> from unittest import main
      >>> main(module='test_module', exit=False)

   Calling ``main`` actually returns an instance of the ``TestProgram`` class.
   This stores the result of the tests run as the ``result`` attribute.

   .. versionchanged:: 3.1
      The ``exit`` parameter was added.
