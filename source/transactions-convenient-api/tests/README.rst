=====================================
Convenient API for Transactions Tests
=====================================

.. contents::

----

Introduction
============

The YAML and JSON files in this directory are platform-independent tests that
drivers can use to prove their conformance to the Convenient API for
Transactions spec.  They are designed with the intention of sharing some
test-runner code with the CRUD, Command Monitoring, and Transaction spec tests.

Several prose tests, which are not easily expressed in YAML, are also presented
in this file. Those tests will need to be manually implemented by each driver.

Server Fail Point
=================

See: `Server Fail Point <../../transactions/tests#server-fail-point>`_ in the
Transactions spec test suite.

Test Format
===========

Each YAML file has the following keys:

- ``database_name`` and ``collection_name``: The database and collection to use
  for testing.

- ``data``: The data that should exist in the collection under test before each
  test run.

- ``minServerVersion`` (optional): The minimum server version (inclusive)
  required to successfully run the test. If this field is not present, it should
  be assumed that there is no lower bound on the required server version.

- ``tests``: An array of tests that are to be run independently of each other.
  Each test will have some or all of the following fields:

  - ``description``: The name of the test.

  - ``skipReason`` (optional): If present, the test should be skipped and the
    string value will specify a reason.

  - ``failPoint`` (optional): The ``configureFailPoint`` command document to run
    to configure a fail point on the primary server.

  - ``clientOptions`` (optional): Names and values of options to pass to
    ``MongoClient()``.

  - ``sessionOptions`` (optional): Names and values of options to pass to
    ``MongoClient.startSession()``.

  - ``operations``: Array of documents, each describing an operation to be
    executed. Each document has the following fields:

    - ``name``: The name of the operation on ``object``.

    - ``object``: The name of the object on which to perform the operation. The
      value will be either "collection" or "session0".

    - ``arguments`` (optional): Names and values of arguments to pass to the
      operation.

    - ``error`` (optional): If ``true``, the test should expect the operation
      to emit an error or exception. If ``false`` or omitted, drivers MUST
      assert that no error occurred.

    - ``result`` (optional): The return value from the operation. This will
      correspond to an operation's result object as defined in the CRUD
      specification. If the operation is expected to return an error, the
      ``result`` is a single document that has one or more of the following
      fields:

      - ``errorContains``: A substring of the expected error message.

      - ``errorCodeName``: The expected "codeName" field in the server
        error response.

      - ``errorLabelsContain``: A list of error label strings that the
        error is expected to have.

      - ``errorLabelsOmit``: A list of error label strings that the
        error is expected not to have.

  - ``expectations`` (optional): List of command-started events.

  - ``outcome``: Document describing the return value and/or expected state of
    the collection after the operation is executed. Contains the following
    fields:

    - ``collection``:

      - ``data``: The data that should exist in the collection after the
        operations have run.

Use as Integration Tests
========================

Testing against a replica set will require server version 4.0.0 or later. The
replica set should include a primary, a secondary, and an arbiter. Including a
secondary ensures that server selection in a transaction works properly.
Including an arbiter helps ensure that no new bugs have been introduced related
to arbiters.

Testing against a sharded cluster will require server version 4.1.6 or later.
A driver that implements support for sharded transactions MUST also run these
tests against a MongoDB sharded cluster with multiple mongoses. Including
multiple mongoses (and initializing the MongoClient with multiple mongos seeds!)
ensures that mongos transaction pinning works properly.

See: `Use as Integration Tests <../../transactions/tests#use-as-integration-tests>`_
in the Transactions spec test suite for instructions on executing each test.

Take note of the following:

- Most tests will consist of a single "withTransaction" operation to be invoked
  on the "session0" object. The ``callback`` argument of that operation will
  resemble the ``operations`` array found in transaction spec tests.

Command-Started Events
``````````````````````

See: `Command-Started Events <../../transactions/tests#command-started-events>`_
in the Transactions spec test suite for instructions on asserting
command-started events.

Prose Tests
===========

Callback Raises a Custom Error
``````````````````````````````

Write a callback that raises a custom exception or error that does not include
either UnknownTransactionCommitResult or TransientTransactionError error labels.
Execute this callback using ``withTransaction`` and assert that the callback's
error bypasses any retry logic within ``withTransaction`` and is propagated to
the caller of ``withTransaction``.

Callback Returns a Value
````````````````````````

Write a callback that returns a custom value (e.g. boolean, string, object).
Execute this callback using ``withTransaction`` and assert that the callback's
return value is propagated to the caller of ``withTransaction``.

Retry Timeout is Enforced
`````````````````````````

Drivers should test that ``withTransaction`` enforces a non-configurable timeout
before retrying both commits and entire transactions. Specifically, three cases
should be checked:

 * If the callback raises an error with the TransientTransactionError label and
   the retry timeout has been exceeded, ``withTransaction`` should propagate the
   error to its caller.
 * If committing raises an error with the UnknownTransactionCommitResult label,
   the error is not a write concern timeout, and the retry timeout has been
   exceeded, ``withTransaction`` should propagate the error to its caller.
 * If committing raises an error with the TransientTransactionError label and
   the retry timeout has been exceeded, ``withTransaction`` should propagate the
   error to its caller. This case may occur if the commit was internally retried
   against a new primary after a failover and the second primary returned a
   NoSuchTransaction error response.
   
 If possible, drivers should implement these tests without requiring the test
 runner to block for the full duration of the retry timeout. This might be done
 by internally modifying the timeout value used by ``withTransaction`` with some
 private API or using a mock timer.
 
