===================
Unified Test Format
===================

:Spec Title: Unified Test Format
:Spec Version: 1.0.0
:Author: Jeremy Mikola
:Advisors: Prashant Mital, Isabel Atkinson
:Status: Draft
:Type: Standards
:Minimum Server Version: N/A
:Last Modified: 2020-08-25

.. contents::

--------

Abstract
========

This project defines a unified schema for YAML and JSON specification tests,
which run operations against a MongoDB deployment. By conforming various spec
tests to a single schema, drivers can implement a single test runner to execute
acceptance tests for multiple specifications, thereby reducing maintenance of
existing specs and implementation time for new specifications.

META
====

The keywords "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD",
"SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be
interpreted as described in `RFC 2119 <https://www.ietf.org/rfc/rfc2119.txt>`__.

Specification
=============


Terms
-----

Entity
  Any object or value that is indexed by a unique name and stored in the
  `Entity Map`_. This will typically be a driver object (e.g. client, session)
  defined in `createEntities`_ but may also be a
  `saved operation result <operation_saveResultAsEntity_>`_. Entities are
  referenced throughout the test file (e.g. `Entity Test Operations`_).

Internal MongoClient
  A MongoClient created specifically for use with internal test operations, such
  as inserting collection data before a test, configuring fail points during a
  test, or asserting collection data after a test.


Schema Version
--------------

This specification and the `Test Format`_ follow
`semantic versioning <https://semver.org/>`__. Backwards breaking changes (e.g.
removing a field, introducing a required field) will warrant a new major
version. Backwards compatible changes (e.g. introducing an optional field) will
warrant a new minor version. Small fixes and internal changes (e.g. grammar,
adding clarifying text to the spec) will warrant a new patch version; however,
patch versions SHOULD NOT alter the structure of the test format and thus SHOULD
NOT be relevant to test files.

Each test file defines a `schemaVersion`_, which test runners will use to
determine compatibility (i.e. whether and how the test file will be
interpreted). Test files are considered compatible with a test runner if their
`schemaVersion`_ is less than or equal to a supported version in the test
runner, given the same major version component. For example:

- A test runner supporting version 1.5.1 could execute test files with versions
  1.0 and 1.5 but *not* 1.6 and 2.0.
- A test runner supporting version 2.1 it could execute test files with versions
  2.0 and 2.1 but *not* 1.0 and 1.5.
- A test runner supporting *both* versions 1.5.1 and 2.0 could execute test
  files with versions 1.4, 1.5, and 2.0, but *not* 1.6, 2.1, or 3.0.
- A test runner supporting version 2.0.1 could execute test files with versions
  2.0 and 2.0.1 but *not* 2.0.2 or 2.1. This example is provided for
  completeness, but test files SHOULD NOT need to refer to patch versions (as
  previously mentioned).

Test runners MUST NOT process incompatible files but MAY determine how to handle
such files (e.g. skip and log a notice, fail and raise an error). Test runners
MAY support multiple schema versions (as demonstrated in the example above).

Each major version of this specification will have a corresponding JSON schema
(e.g. `schema-1.json <schema-1.json>`__), which may be used to programmatically
validate YAML and JSON files using a tool such as `Ajv <https://ajv.js.org/>`__.

The latest JSON schema MUST remain consistent with the `Test Format`_ section.
If and when a new major version is introduced, the `Breaking Changes`_ section
must be updated and JSON schema(s) for any previous major version(s) MUST remain
available so that older test files can still be validated. New tests files
SHOULD always be written using the latest version of this specification.


Entity Map
----------

The entity map indexes arbitrary objects and values by unique names, so that
they can be referenced from test constructs (e.g.
`operation.object <operation_object_>`_). To ensure each test is executed in
isolation, test runners MUST NOT share entity maps between tests. Most entities
will be driver objects created by the `createEntities`_ directive during test
setup, but the entity map may also be modified during test execution via the
`operation.saveResultAsEntity <operation_saveResultAsEntity_>`_ directive.

Test runners MAY choose to implement the entity map in a fashion most suited to
their language, but implementations MUST enforce both uniqueness of entity names
and referential integrity when fetching an entity. Test runners MUST raise an
error if an attempt is made to store an entity with a name that already exists
in the map and MUST raise an error if an entity is not found for a name or is
found but has an unexpected type.

Consider the following examples::

    # Error due to a duplicate name (client0 was already defined)
    createEntities:
      - client: { id: client0 }
      - client: { id: client0 }

    # Error due to a missing entity (client1 is undefined)
    createEntities:
      - client: { id: client0 }
      - session: { id: session0, client: client1 }

    # Error due to an unexpected entity type (session instead of client)
    createEntities:
      - client: { id: client0 }
      - session: { id: session0, client: client0 }
      - session: { id: session1, client: session0 }


Test Format
-----------

Each specification test file can define one or more tests, which inherit some
top-level configuration (e.g. namespace, initial data). YAML and JSON test files
are parsed as a document by the test runner. This section defines the top-level
keys for that document and links to various sub-sections for definitions of
nested structures (e.g. individual `test`_, `operation`_).

Although test runners are free to process YAML or JSON files, YAML is the
canonical format for writing tests. YAML files may be converted to JSON using a
tool such as `js-yaml <https://github.com/nodeca/js-yaml>`__ .


Top-level Fields
~~~~~~~~~~~~~~~~

The top-level fields of a test file are as follows:

.. _schemaVersion:

- ``schemaVersion``: Required string. Version of this specification to which the
  test file complies. Test runners will use this to determine compatibility
  (i.e. whether and how the test file will be interpreted). The format of this
  string is defined in `Version String`_; however, test files SHOULD NOT need to
  refer to specific patch versions since patch-level changes SHOULD NOT alter
  the structure of the test format (as previously noted in `Schema Version`_).

.. _runOn:

- ``runOn``: Optional array of documents. List of server version and/or topology
  requirements for which the tests in this file can be run. If no requirements
  are met, the test runner MUST skip this test file.

  If set, the array should contain at least one document. The structure of each
  document is defined in `runOnRequirement`_.

.. _createEntities:

- ``createEntities``: Optional array of documents. List of entities (e.g.
  client, collection, session objects) that should be created before each test
  case is executed. The structure of each document is defined in `entity`_.

.. _initialData:

- ``initialData``: Optional array of documents. Data that should exist in
  collections before each test case is executed.

  If set, the array should contain at least one document. The structure of each
  document is defined in `collectionData`_. Before each test and for each
  `collectionData`_, the test runner MUST drop and the collection and insert the
  specified documents (if any) using a "majority" write concern. If no documents
  are specified, the test runner MUST create the collection with a "majority"
  write concern.

  The behavior to explicitly create a collection when no documents are specified
  is primarily used for testing transactions, since collections cannot be
  created within transactions.

.. _tests:

- ``tests``: Required array of documents. List of test cases to be executed
  independently of each other.

  The array should contain at least one document. The structure of each
  document is defined in `test`_.


runOnRequirement
~~~~~~~~~~~~~~~~

A combination of server version and/or topology requirements for running the
test(s).

The structure of this document is as follows:

- ``minServerVersion``: Optional string. The minimum server version (inclusive)
  required to successfully run the tests. If this field is omitted, it should be
  assumed that there is no lower bound on the required server version. The
  format of this string is defined in `Version String`_.

- ``maxServerVersion``: Optional string. The maximum server version (inclusive)
  against which the tests can be run successfully. If this field is omitted, it
  should be assumed that there is no upper bound on the required server version.
  The format of this string is defined in `Version String`_.

- ``topologies``: Optional string or array of strings. One or more of server
  topologies against which the tests can be run successfully. Valid topologies
  are "single", "replicaset", "sharded", and "sharded-replicaset" (i.e. sharded
  cluster backed by replica sets). If this field is omitted, it should be
  assumed that there is no topology requirement for the test.

  When matching a "sharded-replicaset" topology, test runners MUST ensure that
  all shards are backed by a replica set. The process for doing so is described
  in `Determining if a Sharded Cluster Uses Replica Sets`_.


entity
~~~~~~

An entity (e.g. client, collection, session object) that will be created in the
`Entity Map`_ before each test is executed.

This document MUST contain **exactly one** top-level key that identifies the
entity type and maps to a nested document, which specifies a unique name for the
entity (``id`` key) and any other parameters necessary for its construction.
Tests SHOULD use sequential names based on the entity type (e.g. "session0",
"session1").

When defining an entity document in YAML, a `node anchor`_ SHOULD be created on
the entity's ``id`` key. This anchor will allow the unique name to be referenced
with an `alias node`_ later in the file (e.g. from another entity or
`operation`_ document) and also leverage YAML's parser for reference validation.

.. _node anchor: https://yaml.org/spec/1.2/spec.html#id2785586
.. _alias node: https://yaml.org/spec/1.2/spec.html#id2786196

The structure of this document is as follows:

.. _entity_client:

- ``client``: Optional document. Corresponds with a MongoClient object.

  The structure of this document is as follows:

  - ``id``: Required string. Unique name for this entity. The YAML file SHOULD
    define a `node anchor`_ for this field (e.g. ``id: &client0 client0``).

  - ``uriOptions``: Optional document. Additional URI options to apply to the
    test suite's connection string that is used to create this client. Any keys
    in this document MUST override conflicting keys in the connection string.

    Documentation for supported options may be found in the
    `URI Options <../uri-options/uri-options.rst>`__ spec, with one notable
    exception: if ``readPreferenceTags`` is specified in this document, the key
    will map to an array of strings, each representing a tag set, since it is
    not feasible to define multiple ``readPreferenceTags`` keys in the document.

  .. _entity_client_useMultipleMongoses:

  - ``useMultipleMongoses``: Optional boolean. If true and the topology is a
    sharded cluster, the test runner MUST assert that this MongoClient connects
    to multiple mongos hosts (e.g. by inspecting the connection string). If
    false and the topology is a sharded cluster, the test runner MUST ensure
    that this MongoClient connects to only a single mongos host (e.g. by
    modifying the connection string). This option has no effect for non-sharded
    topologies.

  .. _entity_client_observeEvents:

  - ``observeEvents``: Optional string or array of strings. One or more types of
    events that can be observed for this client. Unspecified event types MUST
    be ignored by this client's event listeners and SHOULD NOT be included in
    `test.expectedEvents <test_expectedEvents_>`_ for this client. Supported
    types correspond to those documented in `expectedEvent`_ and are as follows:

    - `commandStartedEvent <expectedEvent_commandStartedEvent_>`_

    - `commandSucceededEvent <expectedEvent_commandSucceededEvent_>`_

    - `commandFailedEvent <expectedEvent_commandFailedEvent_>`_

.. _entity_database:

- ``database``: Optional document. Corresponds with a Database object.

  The structure of this document is as follows:

  - ``id``: Required string. Unique name for this entity. The YAML file SHOULD
    define a `node anchor`_ for this field (e.g. ``id: &database0 database0``).

  - ``client``: Required string. Client entity from which this database will be
    created. The YAML file SHOULD use an `alias node`_ for a client entity's
    ``id`` field (e.g. ``client: *client0``).

  - ``databaseName``: Required string. Database name. The YAML file SHOULD
    define a `node anchor`_ for this field (e.g.
    ``databaseName: &database0Name foo``).

  - ``databaseOptions``: Optional document. See `collectionOrDatabaseOptions`_.

.. _entity_collection:

- ``collection``: Optional document. Corresponds with a Collection object.

  The structure of this document is as follows:

  - ``id``: Required string. Unique name for this entity. The YAML file SHOULD
    define a `node anchor`_ for this field (e.g.
    ``id: &collection0 collection0``).

  - ``database``: Required string. Database entity from which this collection
    will be created. The YAML file SHOULD use an `alias node`_ for a database
    entity's ``id`` field (e.g. ``database: *database0``).

  - ``collectionName``: Required string. Collection name. The YAML file SHOULD
    define a `node anchor`_ for this field (e.g.
    ``collectionName: &collection0Name foo``).

  - ``collectionOptions``: Optional document. See
    `collectionOrDatabaseOptions`_.

.. _entity_session:

- ``session``: Optional document. Corresponds with an explicit ClientSession
  object.

  The structure of this document is as follows:

  - ``id``: Required string. Unique name for this entity. The YAML file SHOULD
    define a `node anchor`_ for this field (e.g. ``id: &session0 session0``).

  - ``client``: Required string. Client entity from which this session will be
    created. The YAML file SHOULD use an `alias node`_ for a client entity's
    ``id`` field (e.g. ``client: *client0``).

  - ``sessionOptions``: Optional document. Map of parameters to pass to
    `MongoClient.startSession <../source/sessions/driver-sessions.rst#startsession>`__
    when creating the session. Supported options are defined in the following
    specifications:

    - `Causal Consistency <../causal-consistency/causal-consistency.rst#sessionoptions-changes>`__
    - `Transactions <../transactions/transactions.rst#sessionoptions-changes>`__

    When specifying TransactionOptions for ``defaultTransactionOptions``, the
    transaction options MUST remain nested under ``defaultTransactionOptions``
    and MUST NOT be flattened into ``sessionOptions``.

- ``bucket``: Optional document. Corresponds with a GridFS Bucket object.

  The structure of this document is as follows:

  - ``id``: Required string. Unique name for this entity. The YAML file SHOULD
    define a `node anchor`_ for this field (e.g. ``id: &bucket0 bucket0``).

  - ``database``: Required string. Database entity from which this bucket will
    be created. The YAML file SHOULD use an `alias node`_ for a database
    entity's ``id`` field (e.g. ``database: *database0``).

  - ``bucketOptions``: Optional document. Additional options used to construct
    the bucket object. Supported options are defined in the
    `GridFS <../source/gridfs/gridfs-spec.rst#configurable-gridfsbucket-class>`__
    specification. The ``readConcern``, ``readPreference``, and ``writeConcern``
    options use the same structure as defined in `Common Options`_.


collectionData
~~~~~~~~~~~~~~

List of documents that should correspond to the contents of a collection. This
structure is used by both `initialData`_ and `test.outcome <test_outcome_>`_,
which insert and read documents, respectively.

The structure of this document is as follows:

- ``collectionName``: Required string. See `commonOptions_collectionName`_.

- ``databaseName``: Required string. See `commonOptions_databaseName`_.

- ``documents``: Required array of documents. List of documents corresponding to
  the contents of the collection. This list may be empty.


test
~~~~

Test case consisting of a sequence of operations to be executed.

The structure of each document is as follows:

- ``description``: Required string. The name of the test.

.. _test_runOn:

- ``runOn``: Optional array of documents. List of server version and/or topology
  requirements for which the tests in this file can be run. If specified, these
  requirements are evaluated independently and in addition to any top-level
  `runOn`_ requirements. If no requirements in this array are met, the test
  runner MUST skip this test.

  These requirements SHOULD be more restrictive than those specified in the
  top-level `runOn`_ requirements (if any). They SHOULD NOT be more permissive.
  This is advised because both sets of requirements MUST be satisified in order
  for a test to be executed and more permissive requirements at the test-level
  could be taken out of context on their own.

  If set, the array should contain at least one document. The structure of each
  document is defined in `runOnRequirement`_.

.. _test_skipReason:

- ``skipReason``: Optional string. If set, the test will be skipped. The string
  SHOULD explain the reason for skipping the test (e.g. JIRA ticket).

.. _test_operations:

- ``operations``: Required array of documents. List of operations to be executed
  for the test case.

  The array should contain at least one document. The structure of each
  document is defined in `operation`_.

.. _test_expectedEvents:

- ``expectedEvents``: Optional array of documents. Each document will specify a
  client entity and a list of events that are expected to be observed (in that
  order) on that client while executing `operations <test_operations_>`_.

  If a driver only supports configuring event listeners globally (for all
  clients), the test runner SHOULD associate each observed event with a client
  in order to perform these assertions.

  The array should contain at least one document. The structure of each document
  is as follows:

  - ``client``: Required string. Client entity on which the events are expected
    to be observed. See `commonOptions_client`_.

  - ``events``: Required array of documents. List of events, which are expected
    to be observed (in this order) on the corresponding client while executing
    `operations`_. If the array is empty, the test runner MUST assert that no
    events were observed on the client.

    The structure of each document is defined in `expectedEvent`_.

.. _test_outcome:

- ``outcome``: Optional array of documents. Each document will specify expected
  contents of a collection after all operations have been executed. The list of
  documents therein SHOULD be sorted ascendingly by the ``_id`` field to allow
  for deterministic comparisons.

  If set, the array should contain at least one document. The structure of each
  document is defined in `collectionData`_.


operation
~~~~~~~~~

An operation to be executed as part of the test.

The structure of this document is as follows:

.. _operation_name:

- ``name``: Required string. Name of the operation (e.g. method) to perform on
  the object.

.. _operation_object:

- ``object``: Required string. Name of the object on which to perform the
  operation. This should correspond to either an `entity`_ name (for
  `Entity Test Operations`_) or "testRunner" (for `Special Test Operations`_).
  If the object is an entity, The YAML file SHOULD use an `alias node`_ for its
  ``id`` field (e.g. ``object: *collection0``).

.. _operation_arguments:

- ``arguments``: Optional document. Map of parameter names and values for the
  operation. The structure of this document will vary based on the operation.
  See `Entity Test Operations`_ and `Special Test Operations`_.

  The ``session`` parameter is handled specially (see `commonOptions_session`_).

.. _operation_expectedError:

- ``expectedError``: Optional document. One or more assertions for an expected
  error raised by the operation. The structure of this document is
  defined in `expectedError`_.

  This field is mutually exclusive with
  `expectedResult <operation_expectedResult_>`_ and
  `saveResultAsEntity <operation_saveResultAsEntity_>`_.

.. _operation_expectedResult:

- ``expectedResult``: Optional mixed type. A value corresponding to the expected
  result of the operation. This field may be a scalar value, a single document,
  or an array of documents in the case of a multi-document read.  Test runners
  MUST follow the rules in `Evaluating Matches`_ when processing this assertion.

  This field is mutually exclusive with
  `expectedError <operation_expectedError_>`_.

.. _operation_saveResultAsEntity:

- ``saveResultAsEntity``: Optional string. If specified, the actual result
  returned by the operation (if any) will be saved with this name in the
  `Entity Map`_.  The test runner MUST raise an error if the name is already in
  use. If the operation does not return a value (e.g. void method), the test
  runner MAY choose to store an empty value (e.g. ``null``) or do nothing and
  leave the entity name undefined.

  This field is mutually exclusive with
  `expectedError <operation_expectedError_>`_.

  **TODO**: This is primarily used for change streams. Once an operation for
  iterating a change stream is added, it should link to ``saveResultAsEntity``
  as this will be the only way to add a change stream object to the entity map.


expectedError
~~~~~~~~~~~~~

One or more assertions for an error/exception, which is expected to be raised by
an executed operation. At least one key is required in this document.

The structure of this document is as follows:

- ``type``: Optional string or array of strings. One or more classifications of
  errors, at least one of which should apply to the expected error.

  Valid types are as follows:

  - ``client``: client-generated error (e.g. parameter validation error before
    a command is sent to the server).

  - ``server``: server-generated error (e.g. error derived from a server
    response).

- ``errorContains``: Optional string. A substring of the expected error message.
  The test runner MUST assert that the error message contains this string using
  a case-insensitive match.

  See `bulkWrite`_ for special considerations for BulkWriteExceptions.

- ``errorCodeName``: Optional string. The expected "codeName" field in the
  server-generated error response. The test runner MUST assert that the error
  includes a server-generated response whose "codeName" field equals this value
  using a case-insensitive comparison.

  See `bulkWrite`_ for special considerations for BulkWriteExceptions.

- ``errorLabelsContain``: Optional array of strings. A list of error label
  strings that the error is expected to have. The test runner MUST assert that
  the error contains all of the specified labels (e.g. using the
  ``hasErrorLabel`` method).

- ``errorLabelsOmit``: Optional array of strings. A list of error label strings
  that the error is expected not to have. The test runner MUST assert that
  the error does not contain any of the specified labels (e.g. using the
  ``hasErrorLabel`` method).

.. _expectedError_expectedResult:

- ``expectedResult``: Optional mixed type. This field follows the same rules as
  `operation.expectedResult <operation_expectedResult_>`_ and is only used in
  cases where the error includes a result (e.g. `bulkWrite`_). If specified, the
  test runner MUST assert that the error includes a result and that it matches
  this value.


expectedEvent
~~~~~~~~~~~~~

An event (e.g. APM), which is expected to be observed while executing the test's
operations.

This document MUST contain **exactly one** top-level key that identifies the
event type and maps to a nested document, which contains one or more assertions
for the event's properties.

The structure of this document is as follows:

.. _expectedEvent_commandStartedEvent:

- ``commandStartedEvent``: Optional document. Assertions for a one or more
  `CommandStartedEvent <../command-monitoring/command-monitoring.rst#api>`__
  fields.

  The structure of this document is as follows:

  - ``command``: Optional document. Test runners MUST follow the rules in
    `Evaluating Matches`_ when processing this assertion.

  - ``commandName``: Optional string. Test runners MUST assert that the actual
    command name matches this value using a case-insensitive comparison.

  - ``databaseName``: Optional string. Test runners MUST assert that the actual
    command name matches this value using a case-insensitive comparison. THe
    YAML file SHOULD use an `alias node`_ for this value (e.g.
    ``databaseName: *database0Name``).

.. _expectedEvent_commandSucceededEvent:

- ``commandSucceededEvent``: Optional document. Assertions for a one or more
  `CommandSucceededEvent <../command-monitoring/command-monitoring.rst#api>`__
  fields.

  The structure of this document is as follows:

  - ``reply``: Optional document. Test runners MUST follow the rules in
    `Evaluating Matches`_ when processing this assertion.

  - ``commandName``: Optional string. Test runners MUST assert that the actual
    command name matches this value using a case-insensitive comparison.

.. _expectedEvent_commandFailedEvent:

- ``commandFailedEvent``: Optional document. Assertions for a one or more
  `CommandFailedEvent <../command-monitoring/command-monitoring.rst#api>`__
  fields.

  The structure of this document is as follows:

  - ``commandName``: Optional string. Test runners MUST assert that the actual
    command name matches this value using a case-insensitive comparison.


collectionOrDatabaseOptions
~~~~~~~~~~~~~~~~~~~~~~~~~~~

Map of parameters used to construct a collection or database object.

The structure of this document is as follows:

  - ``readConcern``: Optional document. See `commonOptions_readConcern`_.

  - ``readPreference``: Optional document. See `commonOptions_readPreference`_.

  - ``writeConcern``: Optional document. See `commonOptions_writeConcern`_.


Common Options
--------------

This section defines the structure of common options that are referenced from
various contexts in the test format. Comprehensive documentation for some of
these types and their parameters may be found in the following specifications:

- `Read and Write Concern <../read-write-concern/read-write-concern.rst>`__.
- `Server Selection: Read Preference <../server-selection/server-selection.rst#read-preference>`__.

The structure of these common options is as follows:

.. _commonOptions_collectionName:

- ``collectionName``: String. Collection name. The YAML file SHOULD use an
  `alias node`_ for a collection entity's ``collectionName`` field (e.g.
  ``collectionName: *collection0Name``).

.. _commonOptions_databaseName:

- ``databaseName``: String. Database name. The YAML file SHOULD use an
  `alias node`_ for a database entity's ``databaseName`` field (e.g.
  ``databaseName: *database0Name``).

.. _commonOptions_readConcern:

- ``readConcern``: Document. Map of parameters to construct a read concern.

  The structure of this document is as follows:

  - ``level``: Required string.

.. _commonOptions_readPreference:

- ``readPreference``: Document. Map of parameters to construct a read
  preference.

  The structure of this document is as follows:

  - ``mode``: Required string.

  - ``tagSets``: Optional array of documents.

  - ``maxStalenessSeconds``: Optional integer.

  - ``hedge``: Optional document.

.. _commonOptions_client:

- ``client``: String. Client entity name, which the test runner MUST resolve
  to a MongoClient object. The YAML file SHOULD use an `alias node`_ for a
  client entity's ``id`` field (e.g. ``client: *client0``).

.. _commonOptions_session:

- ``session``: String. Session entity name, which the test runner MUST resolve
  to a ClientSession object. The YAML file SHOULD use an `alias node`_ for a
  session entity's ``id`` field (e.g. ``session: *session0``).

.. _commonOptions_writeConcern:

- ``writeConcern``: Document. Map of parameters to construct a write concern.

  The structure of this document is as follows:

  - ``journal``: Optional boolean.

  - ``w``: Optional integer or string.

  - ``wtimeoutMS``: Optional integer.


Version String
--------------

Version strings, which are used for `schemaVersion`_ and `runOn`_, MUST conform
to one of the following formats, where each component is an integer:

- ``<major>.<minor>.<patch>``
- ``<major>.<minor>`` (``<patch>`` is assumed to be zero)
- ``<major>`` (``<minor>`` and ``<patch>`` are assumed to be zero)


Entity Test Operations
----------------------

Entity operations correspond to an API method on a driver object. If
`operation.object <operation_object_>`_ refers to an `entity`_ name (e.g.
"collection0") then `operation.name <operation_name_>`_ is expected to reference
an API method on that class.

Some specifications group optional parameters for API methods under an
``options`` parameter (e.g. ``options: Optional<UpdateOptions>`` in the CRUD
spec). While this improves readability of the spec document(s) by allowing
option documentation to be expanded and reused, it would add unnecessary
verbosity to test files. Therefore, test files SHALL declare all required and
optional parameters for an API method directly within
`operation.arguments <operation_arguments_>`_ (e.g. ``upsert`` for ``updateOne``
is *not* nested under an ``options`` key), unless otherwise stated below.

If ``session`` is specified in `operation.arguments`_, it is defined according
to `commonOptions_session`_. Test runners MUST resolve the ``session`` argument
to session entity *before* passing it as a parameter to any API method.

If ``readConcern``, ``readPreference``, or ``writeConcern`` are specified in
`operation.arguments`_, test runners MUST interpret them according to the
corresponding definition in `Common Options`_ and MUST convert the value into
the appropriate object *before* passing it as a parameter to any API method.

This spec does not provide exhaustive documentation for all possible API methods
that may appear in a test; however, the following sections discuss all supported
entities and their operations in some level of detail. Special handling for
certain operations is also discussed below.


client
~~~~~~

These operations and their arguments may be documented in the following
specifications:

- `Change Streams <../change-streams/change-streams.rst>`__
- `Enumerating Databases <../enumerate-databases.rst>`__


database
~~~~~~~~

These operations and their arguments may be documented in the following
specifications:

- `Change Streams <../change-streams/change-streams.rst>`__
- `CRUD <../crud/crud.rst>`__
- `Enumerating Collections <../enumerate-collections.rst>`__

Other database operations not documented by an existing specification follow.


runCommand
``````````

Generic command runner.

This method does not inherit a read concern or write concern (per the
`Read and Write Concern <../read-write-concern/read-write-concern.rst#generic-command-method>`__
spec), nor does it inherit a read preference (per the
`Server Selection <../server-selection/server-selection.rst#use-of-read-preferences-with-commands>`__
spec); however, they may be specified as arguments.

The following arguments are supported:

- ``command``: Required document. The command to be executed.

- ``commandName``: Required string. The name of the command to run. This is used
  by languages that are unable preserve the order of keys in the ``command``
  argument when parsing YAML/JSON.

- ``readConcern``: Optional document. See `commonOptions_readConcern`_.

- ``readPreference``: Optional document. See `commonOptions_readPreference`_.

- ``session``: Optional string. See `commonOptions_session`_.

- ``writeConcern``: Optional document. See `commonOptions_writeConcern`_.


collection
~~~~~~~~~~

These operations and their arguments may be documented in the following
specifications:

- `Change Streams <../change-streams/change-streams.rst>`__
- `CRUD <../crud/crud.rst>`__
- `Enumerating Indexes <../enumerate-indexes.rst>`__
- `Index Management <../index-management.rst>`__

Collection operations that require special handling are described below.


bulkWrite
`````````

The ``requests`` parameter for ``bulkWrite`` is documented as a list of
WriteModel interfaces. Each WriteModel implementation (e.g. InsertOneModel)
provides important context to the method, but that type information is not
easily expressed in YAML and JSON. To account for this, test files MUST nest
each WriteModel document in a single-key document, where the key identifies the
request type (e.g. "insertOne"), as in the following example::

    arguments:
      requests:
        - insertOne:
            document: { _id: 1, x: 1 }
        - replaceOne:
            filter: { _id: 2 }
            replacement: { x: 2 }
            upsert: true
        - updateOne:
            filter: { _id: 3 }
            update: { $set: { x: 3 } }
            upsert: true
        - updateMany:
            filter: { }
            update: { $inc: { x: 1 } }
        - deleteOne:
            filter: { x: 2 }
        - deleteMany:
            filter: { x: { $gt: 2 } }
      ordered: true

While operations typically raise an error *or* return a result, the
``bulkWrite`` operation is unique in that it may report both via the
``writeResult`` property of a BulkWriteException. In this case, the intermediary
write result may be matched with `expectedError_expectedResult`_. Because
``writeResult`` is optional for drivers to implement, such assertions should
utilize the `$$unsetOrMatches`_ operator.

Additionally, BulkWriteException is unique in that it aggregates one or more
server errors in its ``writeConcernError`` and ``writeErrors`` properties.
When test runners evaluate `expectedError`_ assertions for ``errorContains`` and
``errorCodeName``, they MUST examine the aggregated errors and consider any
match therein to satisfy the assertion(s). Drivers that concatenate all write
and write concern error messages into the BulkWriteException message MAY
optimize the check for ``errorContains`` by examining the concatenated message.
Drivers that expose ``code`` but not ``codeName`` through BulkWriteException MAY
translate the expected code name to a number (see:
`error_codes.yml <https://github.com/mongodb/mongo/blob/master/src/mongo/base/error_codes.yml>`_)
and compare with ``code`` instead, but MUST raise an error if the comparison
cannot be attempted (e.g. ``code`` is also not available, translation fails).


session
~~~~~~~

These operations and their arguments may be documented in the following
specifications:

- `Convenient API for Transactions <../transactions-convenient-api/transactions-convenient-api.rst>`__
- `Driver Sessions <../sessions/driver-sessions.rst>`__

Session operations that require special handling are described below.


withTransaction
```````````````

The ``withTransaction`` operation is unique in that its ``callback`` parameter
is a function and not easily expressed in YAML/JSON. For ease of testing, this
parameter is defined as an array of `operation`_ documents (analogous to
`test.operations <test_operations>`_). Test runners MUST evaluate error and
result assertions when executing these operations in the callback.


bucket
~~~~~~

These operations and their arguments may be documented in the following
specifications:

- `GridFS <../gridfs/gridfs-spec.rst>`__


Special Test Operations
-----------------------

Certain operations do not correspond to API methods but instead represent
special test operations (e.g. assertions). These operations are distinguished by
`operation.object <operation_object_>`_ having a value of "testRunner". The
`operation.name <operation_name_>`_ field will correspond to an operation
defined below.


failPoint
~~~~~~~~~

The ``failPoint`` operation instructs the test runner to configure a fail point
using a ``primary`` read preference using the specified client.

The following arguments are supported:

- ``failPoint``: Required document. The ``configureFailPoint`` command to be
  executed.

- ``client``: Required string. See `commonOptions_client`_.

  The client entity SHOULD specify false for
  `useMultipleMongoses <entity_client_useMultipleMongoses_>`_ if this operation
  could be executed on a sharded topology (according to `runOn`_ or
  `test.runOn <test_runOn_>`_). This is advised because server selection rules
  for mongos could lead to unpredictable behavior if different servers were
  selected for configuring the fail point and executing subsequent operations.

When executing this operation, the test runner MUST keep a record of the fail
point so that it can be disabled after the test. The test runner MUST also
ensure that the ``configureFailPoint`` command is excluded from the list of
observed command monitoring events for this client (if applicable).

An example of this operation follows::

    # Enable the fail point on the server selected with a primary read preference
    - name: failPoint
      object: testRunner
      arguments:
        client: *client0
        failPoint:
          configureFailPoint: failCommand
          mode: { times: 1 }
          data:
            failCommands: ["insert"]
            closeConnection: true


targetedFailPoint
~~~~~~~~~~~~~~~~~

The ``targetedFailPoint`` operation instructs the test runner to configure a
fail point on a specific mongos.

The following arguments are supported:

- ``failPoint``: Required document. The ``configureFailPoint`` command to be
  executed.

- ``session``: Required string. See `commonOptions_session`_.

  The client entity associated with this session SHOULD specify true for
  `useMultipleMongoses <entity_client_useMultipleMongoses_>`_. This is advised
  because multiple mongos connections are necessary to test session pinning.

The MongoClient and mongos on which to run the ``configureFailPoint`` command is
determined by the ``session`` argument (after resolution to a session entity).
Test runners MUST error if the session is not pinned to a mongos server at the
time this operation is executed.

When executing this operation, the test runner MUST keep a record of both the
fail point and session (or pinned mongos server) so that the fail point can be
disabled on the same mongos server after the test. The test runner MUST also
ensure that the ``configureFailPoint`` command is excluded from the list of
observed command monitoring events for this client (if applicable).

An example of this operation follows::

    # Enable the fail point on the mongos to which session0 is pinned
    - name: targetedFailPoint
      object: testRunner
      arguments:
        session: *session0
        failPoint:
          configureFailPoint: failCommand
          mode: { times: 1 }
          data:
            failCommands: ["commitTransaction"]
            closeConnection: true


assertSessionTransactionState
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

The ``assertSessionTransactionState`` operation instructs the test runner to
assert that the given session has a particular transaction state.

The following arguments are supported:

- ``session``: Required string. See `commonOptions_session`_.

- ``state``: Required string. Expected transaction state for the session.
  Possible values are as follows: ``none``, ``starting``, ``in_progress``,
  ``committed``, and ``aborted``.

An example of this operation follows::

    - name: assertSessionTransactionState
      object: testRunner
      arguments:
        session: *session0
        state: in_progress


assertSessionPinned
~~~~~~~~~~~~~~~~~~~

The ``assertSessionPinned`` operation instructs the test runner to assert that
the given session is pinned to a mongos server.

The following arguments are supported:

- ``session``: Required string. See `commonOptions_session`_.

An example of this operation follows::

    - name: assertSessionPinned
      object: testRunner
      arguments:
        session: *session0


assertSessionUnpinned
~~~~~~~~~~~~~~~~~~~~~

The ``assertSessionUnpinned`` operation instructs the test runner to assert that
the given session is not pinned to a mongos server.

The following arguments are supported:

- ``session``: Required string. See `commonOptions_session`_.

An example of this operation follows::

    - name: assertSessionUnpinned
      object: testRunner
      arguments:
        session: *session0


assertDifferentLsidOnLastTwoCommands
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

The ``assertDifferentLsidOnLastTwoCommands`` operation instructs the test runner
to assert that the last two CommandStartedEvents observed on the client have
different ``lsid`` fields. This assertion is primarily used to test that dirty
server sessions are discarded from the pool.

The following arguments are supported:

- ``client``: Required string. See `commonOptions_client`_.

  The client entity SHOULD include "commandStartedEvent" in
  `observeEvents <entity_client_observeEvents_>`_.

The test runner MUST fail this assertion if fewer than two CommandStartedEvents
have been observed on the client or if either command does not include an
``lsid`` field.

An example of this operation follows::

    - name: assertDifferentLsidOnLastTwoCommands
      object: testRunner
      arguments:
        client: *client0


assertSameLsidOnLastTwoCommands
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

The ``assertSameLsidOnLastTwoCommands`` operation instructs the test runner to
assert that the last two CommandStartedEvents observed on the client have
identical ``lsid`` fields. This assertion is primarily used to test that
non-dirty server sessions are not discarded from the pool.

The following arguments are supported:

- ``client``: Required string. See `commonOptions_client`_.

  The client entity SHOULD include "commandStartedEvent" in
  `observeEvents <entity_client_observeEvents_>`_.

The test runner MUST fail this assertion if fewer than two CommandStartedEvents
have been observed on the client or if either command does not include an
``lsid`` field.

An example of this operation follows::

    - name: assertSameLsidOnLastTwoCommands
      object: testRunner
      arguments:
        client: *client0


assertSessionDirty
~~~~~~~~~~~~~~~~~~

The ``assertSessionDirty`` operation instructs the test runner to assert that
the given session is marked dirty.

The following arguments are supported:

- ``session``: Required string. See `commonOptions_session`_.

An example of this operation follows::

    - name: assertSessionDirty
      object: testRunner
      arguments:
        session: *session0


assertSessionNotDirty
~~~~~~~~~~~~~~~~~~~~~

The ``assertSessionNotDirty`` operation instructs the test runner to assert that
the given session is not marked dirty.

The following arguments are supported:

- ``session``: Required string. See `commonOptions_session`_.

An example of this operation follows::

    - name: assertSessionNotDirty
      object: testRunner
      arguments:
        session: *session0


assertCollectionExists
~~~~~~~~~~~~~~~~~~~~~~

The ``assertCollectionExists`` operation instructs the test runner to assert
that the given collection exists in the database. The test runner MUST use the
internal MongoClient for this operation.

The following arguments are supported:

- ``collectionName``: Required string. See `commonOptions_collectionName`_.

- ``databaseName``: Required string. See `commonOptions_databaseName`_.

An example of this operation follows::

    - name: assertCollectionExists
      object: testRunner
      arguments:
        collectionName: *collection0Name
        databaseName:  *database0Name

Use a ``listCollections`` command to check whether the collection exists. Note
that it is currently not possible to run ``listCollections`` from within a
transaction.


assertCollectionNotExists
~~~~~~~~~~~~~~~~~~~~~~~~~

The ``assertCollectionNotExists`` operation instructs the test runner to assert
that the given collection does not exist in the database. The test runner MUST
use the internal MongoClient for this operation.

The following arguments are supported:

- ``collectionName``: Required string. See `commonOptions_collectionName`_.

- ``databaseName``: Required string. See `commonOptions_databaseName`_.

An example of this operation follows::

    - name: assertCollectionNotExists
      object: testRunner
      arguments:
        collectionName: *collection0Name
        databaseName:  *database0Name

Use a ``listCollections`` command to check whether the collection exists. Note
that it is currently not possible to run ``listCollections`` from within a
transaction.


assertIndexExists
~~~~~~~~~~~~~~~~~

The ``assertIndexExists`` operation instructs the test runner to assert that an
index with the given name exists on the collection. The test runner MUST use the
internal MongoClient for this operation.

The following arguments are supported:

- ``collectionName``: Required string. See `commonOptions_collectionName`_.

- ``databaseName``: Required string. See `commonOptions_databaseName`_.

- ``indexName``: Required string. Index name.

An example of this operation follows::

    - name: assertIndexExists
      object: testRunner
      arguments:
        collectionName: *collection0Name
        databaseName:  *database0Name
        indexName: t_1

Use a ``listIndexes`` command to check whether the index exists. Note that it is
currently not possible to run ``listIndexes`` from within a transaction.


assertIndexNotExists
~~~~~~~~~~~~~~~~~~~~

The ``assertIndexNotExists`` operation instructs the test runner to assert that
an index with the given name does not exist on the collection. The test runner
MUST use the internal MongoClient for this operation.

- ``collectionName``: Required string. See `commonOptions_collectionName`_.

- ``databaseName``: Required string. See `commonOptions_databaseName`_.

- ``indexName``: Required string. Index name.

An example of this operation follows::

    - name: assertIndexNotExists
      object: testRunner
      arguments:
        collectionName: *collection0Name
        databaseName:  *database0Name
        indexName: t_1

Use a ``listIndexes`` command to check whether the index exists. Note that it is
currently not possible to run ``listIndexes`` from within a transaction.


Evaluating Matches
------------------

Expected values in tests (e.g.
`operation.expectedResult <operation_expectedResult_>`_) are expressed as either
relaxed or canonical `Extended JSON <../extended-json.rst>`_.

The algorithm for matching expected and actual values is specified with the
following pseudo-code::

    function match (expected, actual):
      if expected is a document:
        if first key of expected starts with "$$":
          assert that the special operator (identified by key) matches
          return

        assert that actual is a document

        for every key/value in expected:
          assert that actual[key] exists
          assert that actual[key] matches value

        return

      if expected is an array:
        assert that actual is an array
        assert that actual and expected have the same number of elements

        for every index/value in expected:
          assert that actual[index] matches value

        return

      // expected is neither a document nor array
      assert that actual and expected are the same type
      assert that actual and expected are equal

The rules for comparing documents and arrays are discussed in more detail in
subsequent sections. When comparing types *other* than documents and arrays,
test runners MAY adopt any of the following approaches to compare expected and
actual values, as long as they are consistent:

- Convert both values to relaxed or canonical `Extended JSON`_ and compare
  strings
- Convert both values to BSON, and compare bytes
- Convert both values to native representations, and compare accordingly

When comparing numeric types (excluding Decimal128), test runners MUST consider
32-bit, 64-bit, and floating point numbers to be equal if their values are
numerically equivalent. For example, an expected value of ``1`` would match an
actual value of ``1.0`` (e.g. ``ok`` field in a server response) but would not
match ``1.5``.

When comparing types that may contain documents (e.g. CodeWScope), test runners
MUST follow standard document matching rules when comparing those properties.


Extra Fields in Actual Documents
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

When matching expected and actual *documents*, test runners MUST permit the
actual documents to contain additional fields not present in the expected
document. For example, the following documents match::

    expected: { x: 1 }
    actual: { x: 1, y: 1 }

The inverse is not true. For example, the following documents would not match::

    expected: { x: 1, y: 1 }
    actual: { x: 1 }

It may be helpful to think of expected documents as a form of query criteria.
The intention behind this rule is that it is not always feasible or relevant for
a test to exhaustively specify all fields in an expected document (e.g. cluster
time in a `CommandStartedEvent <expectedEvent_commandStartedEvent_>`_ command).

Note that this rule for allowing extra fields in actual values only applies when
matching documents. When comparing arrays, expected and actual values MUST
contain the same number of elements. For example, the following arrays
corresponding to a ``distinct`` operation result would not match::

    expected: [ 1, 2, 3 ]
    actual: [ 1, 2, 3, 4 ]

That said, any individual documents *within* an array or list (e.g. result of a
``find`` operation) MAY be matched according to the rules in this section. For
example, the following arrays would match::

    expected: [ { x: 1 }, { x: 2 } ]
    actual: [ { x: 1, y: 1 }, { x: 2, y: 2 } ]


Special Operators for Matching Assertions
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

When matching expected and actual values, an equality comparison is not always
sufficient. For instance, a test file cannot anticipate what a session ID will
be at runtime, but may still want to analyze the contents of an ``lsid`` field
in a command document. To address this need, special operators can be used.

These operators are documents with a single key identifying the operator. Such
keys are prefixed with ``$$`` to ease in detecting an operator (test runners
need only inspect the first key of each document) and differentiate the document
from MongoDB query operators, which use a single `$` prefix. The key will map to
some value that influences the operator's behavior (if applicable).

When examining the structure of an expected value during a comparison, test
runners MUST examine the first key of any document for a ``$$`` prefix and, if
present, defer to the special logic defined in this section.


$$exists
````````

Syntax::

    { field: { $$exists: <boolean> } }

This operator can be used anywhere the value for a key might be specified in an
expected dcoument. If true, the test runner MUST assert that the key exists in
the actual document, irrespective of its value (e.g. a key with a ``null`` value
would match). If false, the test runner MUST assert that the key does not exist
in the actual document. This operator is modeled after the
`$exists <https://docs.mongodb.com/manual/reference/operator/query/exists/>`__
query operator.

An example of this operator checking for a field's presence follows::

    command:
      getMore: { $$exists: true }
      collection: *collectionName,
      batchSize: 5

An example of this operator checking for a field's absence follows::

    command:
      update: *collectionName
      updates: [ { q: {}, u: { $set: { x: 1 } } } ]
      ordered: true
      writeConcern: { $$exists: false }


$$type
``````

Syntax, where ``bsonType`` is a string or integer::

    { $$type: <bsonType> }
    { $$type: [ <bsonType>, <bsonType>, ... ] }

This operator can be used anywhere a matched value is expected (including an
`expectedResult <operation_expectedResult_>`_). The test runner MUST assert that
the actual value exists and matches one of the expected types, which correspond
to the documented types for the
`$type <https://docs.mongodb.com/manual/reference/operator/query/type/>`__
query operator.

An example of this operator follows::

    command:
      getMore: { $$type: [ int, long ] }
      collection: { $$type: 2 } # string

When the actual value is an array, test runners MUST NOT examine types of the
array's elements. Only the type of actual field should be checked. This is
admittedly inconsistent with the behavior of the
`$type <https://docs.mongodb.com/manual/reference/operator/query/type/>`__
query operator, but there is presently no need for this behavior in tests.


$$unsetOrMatches
````````````````

Syntax::

    { $$unsetOrMatches: <anything> }

This operator can be used anywhere a matched value is expected (including an
`expectedResult <operation_expectedResult_>`_). The test runner MUST assert that
actual value either does not exist or matches the expected value. Matching the
expected value should use the standard rules in `Evaluating Matches`_, which
means that it may contain special operators.

This operator is primarily used to assert driver-optional fields from the CRUD
spec (e.g. ``insertedId`` for InsertOneResult, ``writeResult`` for
BulkWriteException).

An example of this operator used for a result's field follows::

    expectedResult:
      insertedId: { $$unsetOrMatches: 2 }

An example of this operator used for an entire result follows::

    expectedError:
      expectedResult:
        $$unsetOrMatches:
          deletedCount: 0
          insertedCount: 2
          matchedCount: 0
          modifiedCount: 0
          upsertedCount: 0
          upsertedIds: { }


$$sessionLsid
`````````````

Syntax::

    { $$sessionLsid: <string> }

This operation is used for matching any value with the logical session ID of a
`session entity <entity_session_>`_. The value will refer to a unique name of a
session entity. The YAML file SHOULD use an `alias node`_ for a session entity's
``id`` field (e.g. ``session: *session0``).

An example of this operator follows::

    command:
      ping: 1
      lsid: { $$sessionLsid: *session0 }


Test Runner Implementation
--------------------------

The sections below describe instructions for instantiating the test runner,
loading each test file, and executing each test within a test file. Test runners
SHOULD NOT share state created by processing a test file with the processing of
subsequent test files, and likewise for tests within a test file.


Initializing the Test Runner
~~~~~~~~~~~~~~~~~~~~~~~~~~~~

The test runner MUST be configurable with a connection string (or equivalent
configuration), which will be used to initialize the internal MongoClient and
any `client entities <entity_client_>`_ (in combination with other URI options).
This specification is not prescriptive about how this information is provided.
For example, it may be read from an environment variable or configuration file.

Create a new MongoClient, which will be used for internal operations (e.g.
processing `initialData`_ and `test.outcome <test_outcome_>`_). This is referred
to elsewhere in the specification as the internal MongoClient.

Determine the server version and topology type using the internal MongoClient.
This information will be used to evaluate any future `runOnRequirement`_ checks.

The test runner SHOULD terminate any open transactions (see:
`Terminating Open Transactions`_) using the internal MongoClient before
executing any tests.


Executing a Test File
~~~~~~~~~~~~~~~~~~~~~

The instructions in this section apply for each test file loaded by the test
runner. After processing a test file, test runners SHOULD reset any internal
state that resulted from doing so. For example, an internal MongoClient created
for one test file SHOULD NOT be shared with another.

Test files, which may be YAML or JSON files, MUST be interpreted using an
`Extended JSON`_ parser. The parser MUST accept relaxed and canonical Extended
JSON (per `Extended JSON: Parsers <../extended-json.rst#parsers>`__), as test
files may use either.

Upon loading a file, the test runner MUST read the `schemaVersion`_ field and
determine if the test file can be processed further. Test runners MAY support
multiple versions and MUST NOT process incompatible files (as discussed in
`Schema Version`_).

If `runOn`_ is specified, the test runner MUST skip the test file unless one or
more `runOnRequirement`_ documents are satisfied.

For each element in `tests`_, follow the process in `Executing a Test`_.


Executing a Test
~~~~~~~~~~~~~~~~

The instructions in this section apply for each `test`_ occuring in a test file
loaded by the test runner. After processing a test, test runners SHOULD reset
any internal state that resulted from doing so. For example, the `Entity Map`_
created for one test SHOULD NOT be shared with another.

If at any point while executing this test an unexpected error is encountered or
an assertion fails, the test runner MUST consider this test to have failed and
SHOULD continue with the instructions in this section to ensure that the test
environment is cleaned up (e.g. disable fail points, kill sessions) while also
forgoing any additional assertions.

If `test.skipReason <test_skipReason_>`_ is specified, the test runner MUST skip
this test and MAY use the string value to log a message.

If `test.runOn <test_runOn_>`_ is specified, the test runner MUST skip the test
unless one or more `runOnRequirement`_ documents are satisfied.

If `initialData`_ is specified, for each `collectionData`_ therein the test
runner MUST drop the collection and insert the specified documents (if any)
using a "majority" write concern. If no documents are specified, the test runner
MUST create the collection with a "majority" write concern. The test runner
MUST use the internal MongoClient for these operations.

Create a new `Entity Map`_ that will be used for this test. If `createEntities`_
is specified, the test runner MUST create each `entity`_ accordingly and add it
to the map. If the topology is a sharded cluster, the test runner MUST handle
`useMultipleMongoses`_ accordingly if it is specified for any client entities.

If the test might execute a ``distinct`` command within a sharded transaction,
for each target collection the test runner SHOULD execute a non-transactional
``distinct`` command on each mongos server using the internal MongoClient. See
`StaleDbVersion Errors on Sharded Clusters`_ for more information.

If `test.expectedEvents <test_expectedEvents_>`_ is specified, for each client
entity the test runner MUST enable all event listeners necessary to collect the
event types specified in `observeEvents <entity_client_observeEvents_>`_. Test
runners MAY leave event listeners disabled for tests and/or clients that do not
assert expected events.

Test runners MUST ensure that ``configureFailPoint`` commands executed for
`failPoint`_ and `targetedFailPoint`_ operations are excluded from the list of
observed command monitoring events (if applicable). This may require manually
filtering out ``configureFailPoint`` command monitoring events from the list of
observed events. Test runners MUST also ensure that any commands containing
sensitive information are excluded (per the
`Command Monitoring <../command-monitoring/command-monitoring.rst#security>`__
spec).

For each element in `test.operations <test_operations_>`_, follow the process
in `Executing an Operation`_. If an unexpected error is encountered or an
assertion fails, the test runner MUST consider this test to have failed.

If any event listeners were enabled on any client entities, the test runner MUST
now disable those event listeners.

If any fail points were configured, the test runner MUST now disable those fail
points (on the same server) to avoid spurious failures in subsequent tests. For
any fail points configured using `targetedFailPoint`_, the test runner MUST
disable the fail point on the same mongos server on which it was originally
configured. See `Disabling Fail Points`_ for more information.

If `test.expectedEvents <test_expectedEvents_>`_ is specified, for each document
therein the test runner MUST assert that the number and sequence of expected
events match the number and sequence of actual events observed on the specified
client. If the list of expected events is empty, the test runner MUST assert
that no events were observed on the client. The process for matching events is
described in `expectedEvent`_.

If `test.outcome <test_outcome_>`_ is specified, for each `collectionData`_
therein the test runner MUST assert that the collection contains exactly the
expected data. The test runner MUST query each collection using an ascending
sort order on the ``_id`` field (i.e. ``{ _id: 1 }``), a ``primary`` read
preference, a ``local`` read concern, and the internal MongoClient. If the list
of documents is empty, the test runner MUST assert that the collection is empty.

Clear the entity map for this test. For each ClientSession in the entity map,
the test runner MUST end the session (e.g. call
`endSession <../sessions/driver-sessions.rst#endsession>`_).

If the test failed, the test runner MUST terminate any open transactions (see:
`Terminating Open Transactions`_).

Proceed to the subsequent test.


Executing an Operation
~~~~~~~~~~~~~~~~~~~~~~

The instructions in this section apply for each `operation`_ occuring in a
`test`_ contained within a test file loaded by the test runner.

If at any point while executing an operation an unexpected error is encountered
or an assertion fails, the test runner MUST consider the parent test to have
failed and proceed from `Executing a Test`_ accordingly.

If `operation.object <operation_object_>`_ is "testRunner", this is a special
operation. If `operation.name <operation_name_>`_ is defined in
`Special Test Operations`_, the test runner MUST execute the operation
accordingly and, if successful, proceed to the next operation in the test;
otherwise, the test runner MUST raise an error for an undefined operation. The
test runner MUST keep a record of any fail points configured by special
operations so that they may be disabled after the current test.

If `operation.object`_ is not "testRunner", this is an entity operation. If
`operation.object`_ is defined in the current test's `Entity Map`_, the test
runner MUST fetch that entity and note its type; otherwise, the test runner
MUST raise an error for an undefined entity. If `operation.name`_ does not
correspond to an operation for the entity type (per `Entity Test Operations`_),
the test runner MUST raise an error for an undefined operation.

Proceed with preparing the operation's arguments. If ``session`` is specified in
`operation.arguments <operation_arguments_>`_, the test runner MUST resolve it
to a session entity and MUST raise an error if the name is undefined or maps to
an unexpected type. 

Before executing the operation, the test runner MUST be prepared to catch a
potential error from the operation (e.g. enter a ``try`` block). Proceed with
executing the operation and capture its result or error.

If `operation.expectedError <operation_expectedError_>`_ is specified, the test
runner MUST assert that the operation yielded an error; otherwise, the test
runner MUST assert that the operation did not yield an error. If an error was
expected, the test runner MUST evaluate any assertions in `expectedError`_
accordingly.

If `operation.expectedResult <operation_expectedError_>`_ is specified, the test
MUST assert that it matches the actual result of the operation according to the
rules outlined in `Evaluating Matches`_.

If `operation.saveResultAsEntity <operation_saveResultAsEntity_>`_ is specified,
the test runner MUST store the result (if any) in the current test's entity map
using the specified name. If the operation did not return a result (e.g.
``void`` method), the test runner MAY decide to store an empty value (e.g.
``null``) or do nothing and leave the entity name undefined.

After asserting the operation's error and/or result and optionally saving the
result, proceed to the subsequent operation.


Special Procedures
~~~~~~~~~~~~~~~~~~

This section describes some procedures that may be referenced from earlier
sections.


Terminating Open Transactions
`````````````````````````````

Open transactions can cause tests to block indiscriminately. Test runners SHOULD
terminate all open transactions at the start of a test suite and after each
failed test by killing all sessions in the cluster. Using the internal
MongoClient, execute the ``killAllSessions`` command on either the primary or,
if connected to a sharded cluster, all mongos servers.

For example::

    db.adminCommand({
      killAllSessions: []
    });

The test runner MAY ignore any command failure with error Interrupted(11601) to
work around `SERVER-38335`_.

.. _SERVER-38335: https://jira.mongodb.org/browse/SERVER-38335


StaleDbVersion Errors on Sharded Clusters
`````````````````````````````````````````

When a shard receives its first command that contains a ``databaseVersion``, the
shard returns a StaleDbVersion error and mongos retries the operation. In a
sharded transaction, mongos does not retry these operations and instead returns
the error to the client. For example::

    Command distinct failed: Transaction aa09e296-472a-494f-8334-48d57ab530b6:1 was aborted on statement 0 due to: an error from cluster data placement change :: caused by :: got stale databaseVersion response from shard sh01 at host localhost:27217 :: caused by :: don't know dbVersion.

To workaround this limitation, a test runners MUST execute a non-transactional
``distinct`` command on each mongos server before running any test that might
execute ``distinct`` within a transaction. To ease the implementation, test
runners MAY execute ``distinct`` before *every* test.

Test runners can remove this workaround once `SERVER-39704`_ is resolved, after
which point mongos should retry the operation transparently. The ``distinct``
command is the only command allowed in a sharded transaction that uses the
``databaseVersion`` concept so it is the only command affected.

.. _SERVER-39704: https://jira.mongodb.org/browse/SERVER-39704


Server Fail Points
------------------

Many tests utilize the ``configureFailPoint`` command to trigger server-side
errors such as dropped connections or command errors. Tests can configure fail
points using the special `failPoint`_ or `targetedFailPoint`_ opertions.

This internal command is not documented in the MongoDB manual (pending
`DOCS-10784`_); however, there is scattered documentation available on the
server wiki (`The "failCommand" Fail Point <failpoint-wiki_>`_) and employee blogs
(e.g. `Intro to Fail Points <failpoint-blog1_>`_,
`Testing Network Errors with MongoDB <failpoint-blog2_>`_). Documentation can
also be gleaned from JIRA tickets (e.g. `SERVER-35004`_, `SERVER-35083`_). This
specification does not aim to provide comprehensive documentation for all fail
points available for driver testing, but some fail points are documented in
`Fail Points Commonly Used in Tests`_.

.. _failpoint-wiki: https://github.com/mongodb/mongo/wiki/The-%22failCommand%22-fail-point
.. _failpoint-blog1: https://kchodorow.com/2013/01/15/intro-to-fail-points/
.. _failpoint-blog2: https://emptysqua.re/blog/mongodb-testing-network-errors/
.. _DOCS-10784: https://jira.mongodb.org/browse/DOCS-10784
.. _SERVER-35004: https://jira.mongodb.org/browse/SERVER-35004
.. _SERVER-35083: https://jira.mongodb.org/browse/SERVER-35083


Configuring Fail Points
~~~~~~~~~~~~~~~~~~~~~~~

The ``configureFailPoint`` command should be executed on the ``admin`` database
and has the following structure::

    db.adminCommand({
        configureFailPoint: <string>,
        mode: <string|document>,
        data: <document>
    });

The value of ``configureFailPoint`` is a string denoting the fail point to be
configured (e.g. "failCommand").

The ``mode`` option is a generic fail point option and may be assigned a string
or document value. The string values "alwaysOn" and "off" may be used to
enable or disable the fail point, respectively. A document may be used to
specify either ``times`` or ``skip``, which are mutually exclusive:

- ``{ times: <integer> }`` may be used to limit the number of times the fail
  point may trigger before transitioning to "off".
- ``{ skip: <integer> }`` may be used to defer the first trigger of a fail
  point, after which it will transition to "alwaysOn".

The ``data`` option is a document that may be used to specify any options that
control the particular fail point's behavior.

In order to use ``configureFailPoint``, the undocumented ``enableTestCommands``
`server parameter <https://docs.mongodb.com/manual/reference/parameters/>`_ must
be enabled by either the configuration file or command line option (e.g.
``--setParameter enableTestCommands=1``). It cannot be enabled at runtime via
the `setParameter <https://docs.mongodb.com/manual/reference/command/setParameter/>`_
command). This parameter should already be enabled for most configuration files
in `mongo-orchestration <https://github.com/10gen/mongo-orchestration>`_.


Disabling Fail Points
~~~~~~~~~~~~~~~~~~~~~

A fail point may be disabled like so::

    db.adminCommand({
        configureFailPoint: <string>,
        mode: "off"
    });


Fail Points Commonly Used in Tests
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~


failCommand
```````````

The ``failCommand`` fail point allows the client to force the server to return
an error for commands listed in the ``data.failCommands`` field. Additionally,
this fail point is documented in server wiki:
`The failCommand Fail Point <https://github.com/mongodb/mongo/wiki/The-%22failCommand%22-fail-point>`__.

The ``failCommand`` fail point may be configured like so::

    db.adminCommand({
        configureFailPoint: "failCommand",
        mode: <string|document>,
        data: {
          failCommands: ["commandName", "commandName2"],
          closeConnection: <true|false>,
          errorCode: <Number>,
          writeConcernError: <document>,
          appName: <string>,
          blockConnection: <true|false>,
          blockTimeMS: <Number>,
        }
    });

``failCommand`` supports the following ``data`` options, which may be combined
if desired:

* ``failCommands``: Required array of strings. Lists the command names to fail.
* ``closeConnection``: Optional boolean, which defaults to ``false``. If
  ``true``, the command will not be executed, the connection will be closed, and
  the client will see a network error.
* ``errorCode``: Optional integer, which is unset by default. If set, the
  command will not be executed and the specified command error code will be
  returned as a command error.
* ``appName``: Optional string, which is unset by default. If set, the fail
  point will only apply to connections for MongoClients created with this
  ``appname``. New in server 4.4.0-rc2 (`SERVER-47195 <https://jira.mongodb.org/browse/SERVER-47195>`_).
* ``blockConnection``: Optional boolean, which defaults to ``false``. If
  ``true``, the server should block the affected commands for ``blockTimeMS``.
  New in server 4.3.4 (`SERVER-41070 <https://jira.mongodb.org/browse/SERVER-41070>`_).
* ``blockTimeMS``: Integer, which is required when ``blockConnection`` is
  ``true``. The number of milliseconds for which the affected commands should be
  blocked. New in server 4.3.4 (`SERVER-41070 <https://jira.mongodb.org/browse/SERVER-41070>`_).


Determining if a Sharded Cluster Uses Replica Sets
--------------------------------------------------

When connected to a mongos server, the test runner can query the
`config.shards <https://docs.mongodb.com/manual/reference/config-database/#config.shards>`__
collection. Each shard in the cluster is represented by a document in this
collection. If the shard is backed by a single server, the ``host`` field will
contain a single host. If the shard is backed by a replica set, the ``host``
field contain the name of the replica followed by a forward slash and a
comma-delimited list of hosts.


Design Rationale
================

This specification was primarily derived from the test formats used by the
`Transactions <../transactions/transactions.rst>`__ and
`CRUD <../crud/crud.rst>`__ specs, which have served models or other specs.


Breaking Changes
================

This section is reserved for future use. Any breaking changes to the test format
should be described here in detail for historical reference, in addition to any
shorter description that may be added to the `Change Log`_.


Open Questions
==============

Note: these questions should be resolved and this section removed before
publishing version 1.0 of the spec.


General
-------


Interaction between top-level and test-level runOn
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

There are at least two ways to handle runOn requirements defined at both the
test and file level:

* If `test.runOn`_ is specified, those requirements are checked in addition to
  any top-level `runOn`_. This behavior is the most straightforward as test
  runners will not have to consider inheritance or overriding anything (similar
  to reasons top-level ``databaseName`` and ``collectionName`` fields were
  ultimately removed). If the top-level `runOn`_ fails, the test runner can skip
  the entire file outright (i.e. short-circuit the logical-and of both checks).

* If `test.runOn`_ is specified, those requirements are checked *instead* of any
  top-level `runOn`_. This would mean that a test runner cannot immediately skip
  an entire file based on evaluation of the top-level `runOn`_; however, it also
  means that humans editing a test file can no longer trust that the top-level
  `runOn`_ applies to all tests in the file.

During review, it was asked if a test runner could enforce that `test.runOn`_ is
only more restrictive (and never more permissive) than the top-level `runOn`_.
While this is technically feasible, it would add considerable complexity to a
test runner to make such comparisons.


Mixing event types in expectedEvent
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Can different event types (e.g. command monitoring, SDAM) appear within the
``events`` array within `test.expectedEvents`_? To date, no specs (including
SDAM) expect events multiple types of events in the same test. `observeEvents`_
does allow tests to filter events per client, so a test could conceivably expect
mixed types (e.g. CommandStartedEvent and ServerDescriptionChangedEvent).

Perhaps the underlying question is: when collecting observed events on a client,
should the test runner keep them in one ordered list or group them into separate
lists (e.g. one list for command monitoring events and another for SDAM events)?


Should failPoint support a read preference?
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

The `failPoint`_ operation currently uses a primary read preference. Should it
solicit a ``readPreference`` argument to target nodes other than the primary?
To date, no spec needs this behavior and this change could easily be deferred to
a future minor version of the test format.


Representing options in operation.arguments
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Existing spec tests are inconsistent in how they represent optional
parameters for operations. In some cases, optional parameters appear directly
in `operation.arguments`_ alongside required parameters (e.g. CRUD). In other
cases, they are nested under an ``options`` key (e.g. Sessions).

Most specs *are* consistent about how API methods are defined in spec documents.
The CRUD spec was on the first documents to define API methods and can be
credited with introducing the ``options`` parameter (e.g.
``options: Optional<UpdateOptions>``); however, this was likely done for
readability of the spec (avoiding very long method declarations and/or embedding
``Optional`` syntax therein). Additionally, documenting options in a separate
type (e.g. UpdateOptions) allows the declarations to be shared and reused across
multiple methods.

That said, this syntax is in no way prescriptive for language implementations.
While some drivers to model options as a struct/object, others solicit them
alongside required parameters (e.g. Python's keyword/named arguments).

Should this spec require that optional parameters be nested under an ``options``
key or solicit them directly in `operation.arguments`_? Both are technically
possible, since test runners handle both forms today.

With regard to CRUD, this issue dates back to
`SPEC-1158 <https://jira.mongodb.org/browse/SPEC-1158>`__. Therein, Jeremy
originally argued in favor of nesting under an ``options`` key because it was
most consistent the text in existing spec documents.

Note: the CRUD spec WriteModels (e.g. UpdateOneModel) for ``bulkWrite`` to not
use ``options`` keys in either the spec document or test files. As such, the
resolution of this question would not impact how WriteModels are expressed in
test files (see: `bulkWrite`_).

Note: this question does not pertain to TransactionOptions in SessionOptions,
which is always nested under ``defaultTransactionOptions``. This is a special
case where all drivers represent TransactionOptions as a separate struct/object,
even if they do not do so for ``startTransaction``.


GridFS Tests
------------


Error Assertions
~~~~~~~~~~~~~~~~

The GridFS tests assert for the following error types: "FileNotFound",
"ChunkIsMissing", "ChunkIsWrongSize", and "RevisionNotFound". These types are
not explicitly defined in the spec and only appear in test files. The spec also
does not specify how messages for these errors should be constructed, so it may
not be feasible to use ``errorContains`` assertions.

Do we care about differentiating these types of errors? If so, we may need to
add them to ``expectedError.type``.


Outcome Assertions
~~~~~~~~~~~~~~~~~~

In the GridFS tests, ``assert.data`` is similar to `test.outcome`_ but it
uses ``*result`` and ``*actual`` to match document data with a saved result
(e.g. returned file ID) or an arbitrary value (e.g. date). Presently,
`test.outcome`_ uses an exact match and does **not** follow the rules in
`Evaluating Matches`_ (e.g. allow extra fields in actual documents, process
special operators). Should we change `test.outcome`_ to follow the same
comparison rules as we use for `expectedResult`_?

If so, `test.outcome`_ and `initialData`_ would no longer share the same
`collectionData`_ structure. This is not problematic, but worth noting.

Alternatively, we could leave `test.outcome`_ as-is and create a new special
operation that *matches* data within a collection.


Human-readable binary data
~~~~~~~~~~~~~~~~~~~~~~~~~~

In `SPEC-1216 <https://jira.mongodb.org/browse/SPEC-1216>`__, Robert recommended
against changing the GridFS tests' ``$hex`` syntax to ``$binary`` Extended JSON,
as the base64-encoded strings in the latter format would not be human-readable.
This spec could introduce a custom operator in
`Special Operators for Matching Assertions`_ to compare an expected hexidecimal
string with an actual binary value, but we would still need to use ``$binary``
syntax in `collectionData`_ for both `initialData`_ and `test.outcome`_.

Another possible work-around is for this spec to discuss or link to a utility
that can easily convert between hex and base64. For example, we can include a
Javascript or Python script in the spec repository that can convert between the
two and assist those editing GridFS test files.


Related Issues
==============

Note: this section should be removed before publishing version 1.0 of the spec.

The following SPEC tickets are associated with
`DRIVERS-709 <https://jira.mongodb.org/browse/DRIVERS-709>`__. This section will
record whether or not each issue is addressed by this spec.

The following tickets are addressed by the test format:

* `SPEC-1158 <https://jira.mongodb.org/browse/SPEC-1158>`__: Spec tests should use a consistent format for CRUD options

  Open question: `Representing options in operation.arguments`_

* `SPEC-1215 <https://jira.mongodb.org/browse/SPEC-1215>`__: Introduce spec test syntax for meta assertions (e.g. any value, not exists)

  See: `Special Operators for Matching Assertions`_

* `SPEC-1216 <https://jira.mongodb.org/browse/SPEC-1216>`__: Update GridFS YAML tests to use newer format

  Open question: `Human-readable binary data`_

* `SPEC-1229 <https://jira.mongodb.org/browse/SPEC-1229>`__: Standardize spec-test syntax for topology assertions

  See `runOnRequirement`_, which is used by both `runOn`_ and `test.runOn`_.

* `SPEC-1254 <https://jira.mongodb.org/browse/SPEC-1254>`__: Rename topology field in spec tests to topologies

  See `runOnRequirement`_.

* `SPEC-1439 <https://jira.mongodb.org/browse/SPEC-1439>`__: Inconsistent error checking in spec tests

  This may still require test format syntax to assert that an error occurred
  without caring about any assertions on the error itself.

* `SPEC-1713 <https://jira.mongodb.org/browse/SPEC-1713>`__: Allow runOn to be defined per-test in addition to per-file

  See `runOn`_ and `test.runOn`_ and related open question:
  `Interaction between top-level and test-level runOn`_

* `SPEC-1723 <https://jira.mongodb.org/browse/SPEC-1723>`__: Introduce test file syntax to disable dropping of collection under test

  Only collections in `initialData`_ are dropped, so this can be achieved by
  omitting the collection from `initialData`_. Additionally, the format supports
  creating an empty collection without inserting any documents (needed by
  transaction tests).

The following tickets can be resolved after the unified test format is approved
and/or other specs begin porting their tests to the format:

* `SPEC-1102 <https://jira.mongodb.org/browse/SPEC-1102>`__: Add "object: collection" to command monitoring tests
* `SPEC-1133 <https://jira.mongodb.org/browse/SPEC-1133>`__: Use APM to assert outgoing commands in CRUD spec tests
* `SPEC-1144 <https://jira.mongodb.org/browse/SPEC-1144>`__: CRUD tests improvements
* `SPEC-1193 <https://jira.mongodb.org/browse/SPEC-1193>`__: Convert change stream spec tests to runOn format
* `SPEC-1230 <https://jira.mongodb.org/browse/SPEC-1230>`__: Rewrite APM spec tests to use common test format
* `SPEC-1238 <https://jira.mongodb.org/browse/SPEC-1238>`__: Convert retryable write spec tests to multi-operation format
* `SPEC-1261 <https://jira.mongodb.org/browse/SPEC-1261>`__: Use runOn syntax to specify APM test requirements
* `SPEC-1375 <https://jira.mongodb.org/browse/SPEC-1375>`__: changeStream spec tests should be run on sharded clusters


Change Log
==========

Note: this will be cleared when publishing version 1.0 of the spec

2020-08-27:

* rename runOn.topology to topologies (SPEC-1254)

* clarify rules for comparing schema versions and note that test files should
  not need to refer to patch versions.

* note that lsid assertions require actually having observed events and lsid
  fields to compare. also remove note about "collecting observed events" after
  executing operations, since events will already need to be accessible while
  running operations in order to evaluate some assertions.

* open question about human-readable binary data for GridFS

* rename runOn.topology to topologies

* create section for related SPEC tickets and explain which are addressed by
  this spec or suitable to be completed after the format is approved (e.g. those
  that pertain to porting over other spec tests to the new format).

2020-08-26:

* special operations from sessions spec tests

* clarify interaction between runOn and test.runOn, which are evaluated
  independently and must both be met for a test to be executed. Also add a note
  that test.runOn requirements should only be more restrictive for readability.

* create open questions from GridFS spec and internal TODO items

2020-08-25:

* note that drivers with global event listeners will need to associate events
  with clients for executing assertions (copied from change streams spec).

* require databaseName and collectionName when creating database and collection
  entities, respectively. also require those options in initialData, outcome,
  and special operations (YAML aliases may be used). remove top-level
  databaseName and collectionName fields (again) and any language for test
  runners generate their own names.

* remove createOnServer option for collection entities. initialData will now
  explicitly create a collection if the list of documents is empty.

* note handling of read concern, read preference, and write concern options for
  entity operations.

* document arguments for all special operations

* define observeEvents option for client entities to filter observed events

* revise docs for configuring event listeners and remove wording that assumed
  a test runner might only collect command monitoring events.

* failPoint now takes a client entity and no longer uses internal MongoClient.
  this is related to moving the option for multi-mongos and single-mongos to the
  client entity.

* replace top-level allowMultipleMongoses option, which previously applied to
  all clients, with a useMultipleMongoses client entity option. This option can
  be used to require multiple mongos hosts (desirable for targetedFailPoint) or
  modify a connection string to a single host (desirable for failPoint).

* construct internal MongoClient when initializing the test runner and allow it
  to be shared throughout. This is possible because it no longer cares about the
  number of mongos hosts (formerly allowMultipleMongoses).

2020-08-23:

* describe how to determine if sharded clusters use replica sets

* use flexible comparisons for numeric types (to help with ``ok`` matches)

* copy special procedures from transaction spec

* document test runner implementation, including loading test files, executing
  tests, and executing operations

* clarify interactions with fail points and APM

* better describe argument handling for entity operations and document edge
  cases for bulkWrite ops

* document command started, succeeded, and failed events (per APM spec)

* clarify logic for expectedError assertions

* restructure expectedEvents to list events per client

* createOnServer option for collection entities (mainly for transactions)

2020-08-21:

* clarify error assertions for BulkWriteException

* note that YAML is the canonical format and discuss js-yaml

* note that configureFailPoint must be excluded from APM

* reformat external links to YAML spec and fail point docs

* add schemaVersion field and document how the spec will handle versions

* move client.allowMultipleMongoses to top-level option, since it should apply
  to all clients (internal and entities). also note that targetedFailPoint and
  failPoint should not be used in the same test file, since the latter requires
  allowMultipleMongoses:false and would not provide meaningful test coverage of
  mongos pinning for sessions.

* add terms and define Entity and Internal MongoClient

* note that failPoint always uses internal MongoClient and targetedFailPoint
  uses the client of its session argument

* start writing steps for test execution

2020-08-19:

* added test.runOn and clarified that it can override top-level runOn requirements

* runOn.topology can be a single string in addition to array of strings

* added "sharded-replicaset" topology type, which will be relevant for change
  streams, transactions, retryable writes.

* removed top-level collectionName and databaseName fields, since they can be
  specified when creating collection and database entities.

* removed test.clientOptions, since client entities can specify their own options

* moved operation.failPoint to failPoint special operation

* operation.object is now required and takes either an entity name (e.g.
  "collection0") or "testRunner"

* operation.commandName moved to an argument of the runCommand database
  operation. Since that method is documented entirely in this spec, I didn't
  see an issue with consolidating.

* renamed operation.result to expectedResult and noted that it may co-exist with
  error assertions in special cases (e.g. BulkWriteException).

* remove error assertions from operation.result. These are now specified under
  operation.expectedError, which replaces the error boolean and requires at
  least one assertion. Added a type assertion (e.g. client, server), which
  should be useful for discerning client-side and server-side errors (currently
  achieved with APM assertions).

* added operation.saveResultAsEntity to capture a result in the entity map
  (primarily for use with change streams)

* consolidated documentation for ignoring configureFailPoint commands in APM and
  also disabling fail points after a test, which is now referenced from the
  failPoint and targetedFailPoint operations

* removed $$assert nesting in favor of $$<operator>, since test runners can
  easily check the first document key for a ``$$`` prefix.

* completed section on evaluating matches and added pseudo-code
