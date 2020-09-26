.. rst-class:: orm_core

=============================
SQLAlchemy 1.4 / 2.0 Tutorial
=============================

.. admonition:: About this document

    The new SQLAlchemy Tutorial is now integrated between Core and ORM and
    serves as a unified introduction to SQLAlchemy as a whole.   In the new
    :term:`2.0 style` of working, fully available in the :ref:`1.4 release
    <migration_14_toplevel>`, the ORM now uses Core-style querying with the
    :func:`_sql.select` construct, and transactional semantics between Core
    connections and ORM sessions are equivalent.   Take note of the blue
    border styles for each section, that will tell you how "ORM-ish" a
    particular topic is!

    Users who are already familiar with SQLAlchemy, and especially those
    looking to migrate existing applications to work under SQLAlchemy 2.0
    within the 1.4 transitional phase should check out the
    :ref:`migration_20_toplevel` document as well.

    For the newcomer, this document has a **lot** of detail.  When you make
    it to the end however you can call yourself **Alchemist**.

SQLAlchemy is presented as two distinct APIs, one building on top of the other.
These APIs are known as **Core** and **ORM**.

.. container:: core-header

    **SQLAlchemy Core** is the foundational architecture for SQLAlchemy as a
    "database toolkit".  The library provides tools for managing connectivity
    to a database, interacting with database queries and results, and
    programmatic construction of SQL statements.

    Sections that have a **dark blue border on the right** will discuss
    concepts that are **primarily Core-only**; when using the ORM, these
    concepts are still in play but are less often explicit in user code.

.. container:: orm-header

    **SQLAlchemy ORM** builds upon the Core to provide optional **object
    relational mapping** capabilities.   The ORM provides an additional
    configuration layer allowing user-defined Python classes to be **mapped**
    to database tables and other constructs, as well as an object persistence
    mechanism known as the **Session**.   It then extends the Core-level
    SQL Expression Language to allow SQL queries to be composed and invoked
    in terms of user-defined objects.

    Sections that have a **light blue border on the left** will discuss
    concepts that are **primarily ORM-only**.  Core-only users
    can skip these.

.. container:: core-header, orm-dependency

    A section that has **both light and dark borders on both sides** will
    discuss a **Core concept that is also used explicitly with the ORM**.

The tutorial will present both concepts in the natural order that they
should be learned, first with a mostly-Core-centric approach and then
spanning out into a more ORM-centric concepts.

.. rst-class:: core-header, orm-dependency

Version Check
=============


A quick check to verify that we are on at least **version 1.4** of SQLAlchemy:

.. sourcecode:: pycon+sql

    >>> import sqlalchemy
    >>> sqlalchemy.__version__  # doctest: +SKIP
    1.4.0

.. rst-class:: core-header, orm-dependency

A Note on the Future
=====================

This tutorial describes a new API that's released in SQLAlchemy 1.4 known
as :term:`2.0 style`.   The purpose of the 2.0-style API is to provide forwards
compatibility with :ref:`SQLAlchemy 2.0 <migration_20_toplevel>`, which is
planned as the next generation of SQLAlchemy.

In order to provide the full 2.0 API, a new flag called ``future`` will be
used, which will be seen as the tutorial describes the :class:`_engine.Engine`
and :class:`_orm.Session` objects.   These flags fully enable 2.0-compatibility
mode and allow the code in the tutorial to proceed fully.  When using the
``future`` flag with the :func:`_sa.create_engine` function, the object
returned is a sublass of :class:`sqlalchemy.engine.Engine` described as
:class:`sqlalchemy.future.Engine`. This tutorial will be referring to
:class:`sqlalchemy.future.Engine`.


.. rst-class:: core-header, orm-dependency

Establishing Connectivity - the Engine
==========================================

The start of any SQLAlchemy application is an object called the
:class:`_future.Engine`.   This object acts as a central source of connections
to a particular database, providing both a factory as well as a holding
space called a :ref:`connection pool <pooling_toplevel>` for these database
connections.   The engine is typically a global object created just
once for a particular database server, and is configured using a URL string
which will describe how it should connect to the database host or backend.

For this tutorial we will use an in-memory-only SQLite database. This is an
easy way to test things without needing to have an actual pre-existing database
set up.  The :class:`_future.Engine` is created by using :func:`_sa.create_engine`, specifying
the :paramref:`_sa.create_engine.future` flag set to ``True`` so that we make full use
of :term:`2.0 style` usage:

.. sourcecode:: pycon+sql

    >>> from sqlalchemy import create_engine
    >>> engine = create_engine("sqlite+pysqlite:///:memory:", echo=True, future=True)

The main argument to :class:`_sa.create_engine`
is a string URL, above passed as the string ``"sqlite+pysqlite:///:memory:"``.
This string indicates to the :class:`_future.Engine` three important
facts:

1. What kind of database are we communicating with?   This is the ``sqlite``
   portion above, which links in SQLAlchemy to an object known as the
   :term:`dialect`.

2. What :term:`DBAPI` are we using?  The Python :term:`DBAPI` is a third party
   driver that SQLAlchemy uses to interact with a particular database.  In
   this case, we're using the name ``pysqlite``, which in modern Python
   use is the `sqlite3 <http://docs.python.org/library/sqlite3.html>`_ standard
   library interface for SQLite.

3. How do we locate the database?   In this case, our URL includes the phrase
   ``/:memory:``, which is an indicator to the ``sqlite3`` module that we
   will be using an **in-memory-only** database.   This kind of database
   is perfect for experimenting as it does not require any server nor does
   it need to create new files.

.. sidebar:: Lazy Connecting

    The :class:`_future.Engine`, when first returned by :func:`_sa.create_engine`,
    has not actually tried to connect to the database yet; that happens
    only the first time it is asked to perform a task against the database.
    This is a software design pattern known as :term:`lazy initialization`.

We have also specified a parameter :paramref:`_sa.create_engine.echo`, which
will instruct the :class:`_future.Engine` to log all of the SQL it emits to a
Python logger that will write to standard out.   This flag is a shorthand way
of setting up
:ref:`Python logging more formally <dbengine_logging>` and is useful for
experimentation in scripts.   Many of the SQL examples will include this
SQL logging output beneath a ``[SQL]`` link that when clicked, will reveal
the full SQL interaction.

.. _tutorial_working_with_transactions:

Working with Transactions and the DBAPI
========================================

With the :class:`_future.Engine` object ready to go, we may now proceed
to dive into the basic operation of an :class:`_future.Engine` and
its primary interactive endpoints, the :class:`_future.Connection` and
:class:`_engine.Result`.   We will additionally introduce the ORM's
:term:`facade` for these objects, known as the :class:`_orm.Session`.

.. container:: orm-header

    **Note to ORM readers**

    When using the ORM, the :class:`_future.Engine` is managed by another
    object called the :class:`_orm.Session`.  The :class:`_orm.Session` in
    modern SQLAlchemy emphasizes a transactional and SQL execution pattern that
    is largely identical to that of the :class:`_future.Connection` discussed
    below, so while this subsection is Core-centric, all of the concepts here
    are essentially relevant to ORM use as well and is recommended for all ORM
    learners.   The execution pattern used by the :class:`_future.Connection`
    will be contrasted with that of the :class:`_orm.Session` at the end
    of this section.

As we have yet to introduce the SQLAlchemy Expression Language that is the
primary feature of SQLAlchemy, we will make use of one simple construct within
this package called the :func:`_sql.text` construct, which allows us to write
SQL statements as **textual SQL**.   Rest assured that textual SQL in
day-to-day SQLAlchemy use is by far the exception rather than the rule for most
tasks, even though it always remains fully available.

.. rst-class:: core-header

.. _tutorial_getting_connection:

Getting a Connection
---------------------

The sole purpose of the :class:`_future.Engine` object from a user-facing
perspective is to provide a unit of
connectivity to the database called the :class:`_future.Connection`.   When
working with the Core directly, the :class:`_future.Connection` object
is how all interaction with the database is done.   As the :class:`_future.Connection`
represents an open resource against the database, we want to always limit
the scope of our use of this object to a specific context, and the best
way to do that is by using Python context manager form, also known as
`the with statement <https://docs.python.org/3/reference/compound_stmts.html#with>`_.
Below we illustrate "Hello World", using a textual SQL statement.  Textual
SQL is emitted using a construct called :func:`_sql.text` that will be discussed
in more detail later:

.. sourcecode:: pycon+sql

    >>> from sqlalchemy import text

    >>> with engine.connect() as conn:
    ...     result = conn.execute(text("select 'hello world'"))
    ...     print(result.all())
    {opensql}BEGIN (implicit)
    select 'hello world'
    [...] ()
    {stop}[('hello world',)]
    {opensql}ROLLBACK{stop}

In the above example, the context manager provided for a database connection
and also framed the operation inside of a transaction. The default behavior of
the Python DBAPI includes that a transaction is always in progress; when the
scope of the connection is :term:`released`, a ROLLBACK is emitted to end the
transaction.   The transaction is **not committed automatically**; when we want
to commit data we normally need to call :meth:`_future.Connection.commit`
as we'll see in the next section.

.. tip::  "autocommit" mode is available for special cases.  The section
   :ref:`dbapi_autocommit` discusses this.

The result of our SELECT was also returned in an object called
:class:`_engine.Result` that will be discussed later, however for the moment
we'll add that it's best to ensure this object is consumed within the
"connect" block, and is not passed along outside of the scope of our connection.

.. rst-class:: core-header

.. _tutorial_committing_data:

Committing Changes
------------------

We just learned that the DBAPI connection is non-autocommitting.  What if
we want to commit some data?   We can alter our above example to create a
table and insert some data, and the transaction is then committed using
the :meth:`_future.Connection.commit` method, invoked **inside** the block
where we acquired the :class:`_future.Connection` object:

.. sourcecode:: pycon+sql

    # "commit as you go"
    >>> with engine.connect() as conn:
    ...     conn.execute(text("CREATE TABLE some_table (x int, y int)"))
    ...     conn.execute(
    ...         text("INSERT INTO some_table (x, y) VALUES (:x, :y)"),
    ...         [{"x": 1, "y": 1}, {"x": 2, "y": 4}]
    ...     )
    ...     conn.commit()
    {opensql}BEGIN (implicit)
    CREATE TABLE some_table (x int, y int)
    [...] ()
    <sqlalchemy.engine.cursor.CursorResult object at 0x...>
    INSERT INTO some_table (x, y) VALUES (?, ?)
    [...] ((1, 1), (2, 4))
    <sqlalchemy.engine.cursor.CursorResult object at 0x...>
    COMMIT

Above, we emitted two SQL statements that are generally transactional, a
"CREATE TABLE" statement [1]_ and an "INSERT" statement that's parameterized
(the parameterization syntax above is discussed a few sections below in
:ref:`tutorial_multiple_parameters`).  As we want the work we've done to be
committed within our block, we invoke the
:meth:`_future.Connection.commit` method which commits the transaction. After
we call this method inside the block, we can continue to run more SQL
statements and if we choose we may call :meth:`_future.Connection.commit`
again for subsequent statements.  SQLAlchemy refers to this style as **commit as
you go**.

There is also another style of committing data, which is that we can declare
our "connect" block to be a transaction block up front.   For this mode of
operation, we use the :meth:`_future.Engine.begin` method to acquire the
connection, rather than the :meth:`_future.Engine.connect` method.  This method
will both manage the scope of the :class:`_future.Connection` and also
enclose everything inside of a transaction with COMMIT at the end, assuming
a successful block, or ROLLBACK in case of exception raise.  This style
may be referred towards as **begin once**:

.. sourcecode:: pycon+sql

    # "begin once"
    >>> with engine.begin() as conn:
    ...     conn.execute(
    ...         text("INSERT INTO some_table (x, y) VALUES (:x, :y)"),
    ...         [{"x": 6, "y": 8}, {"x": 9, "y": 10}]
    ...     )
    {opensql}BEGIN (implicit)
    INSERT INTO some_table (x, y) VALUES (?, ?)
    [...] ((6, 8), (9, 10))
    <sqlalchemy.engine.cursor.CursorResult object at 0x...>
    COMMIT

"Begin once" style is often preferred as it is more succinct and indicates the
intention of the entire block up front.   However, within this tutorial we will
normally use "commit as you go" style as it is more flexible for demonstration
purposes.

.. topic::  What's "BEGIN (implicit)"?

    You might have noticed the log line "BEGIN (implicit)" at the start of a
    transaction block.  "implicit" here means that SQLAlchemy **did not
    actually send any command** to the database; it just considers this to be
    the start of the DBAPI's implicit transaction.   You can register
    :ref:`event hooks <core_sql_events>` to intercept this event, for example.


.. [1] :term:`DDL` such as "CREATE TABLE" is recommended to be within
   a transaction block that ends with COMMIT, as many databases uses transactional DDL.
   However, as we'll see later, we usually let SQLAlchemy run DDL sequences
   for us as part of a higher level operation where we don't generally need
   to worry about the COMMIT.


.. rst-class:: core-header


Basics of Statement Execution
-----------------------------

We have seen a few examples that run SQL statements against a database, making
use of a method called :meth:`_future.Connection.execute`, in conjunction with
an object called :func:`_sql.text`, and returning an object called
:class:`_engine.Result`.  In this section we'll illustrate more closely the
mechanics and interactions of these components.

.. container:: orm-header

  Most of the content in this section applies equally well to modern ORM
  use when using the :meth:`_orm.Session.execute` method, which works
  very similarly to that of :meth:`_future.Connection.execute`, including that
  ORM result rows are delivered using the same :class:`_engine.Result`
  interface used by Core.

.. rst-class:: orm-addin

Fetching Rows
^^^^^^^^^^^^^

We'll first illustrate the :class:`_engine.Result` object more closely by
making use of the rows we've inserted previously, running a textual SELECT
statement on the table we've created:


.. sourcecode:: pycon+sql

    >>> with engine.connect() as conn:
    ...     result = conn.execute(text("SELECT x, y FROM some_table"))
    ...     for row in result:
    ...         print(f"x: {row.x}  y: {row.y}")
    {opensql}BEGIN (implicit)
    SELECT x, y FROM some_table
    [...] ()
    {stop}x: 1  y: 1
    x: 2  y: 4
    x: 6  y: 8
    x: 9  y: 10
    {opensql}ROLLBACK{stop}

Above, the "SELECT" string we executed selected all rows from our table.
The object returned is called :class:`_engine.Result` and represents an
iterable object of result rows.

:class:`_engine.Result` has lots of methods for
fetching and transforming rows, such as the :meth:`_engine.Result.all`
method illustrated previously, which returns a list of all :class:`_engine.Row`
objects.   It also implements the Python iterator interface so that we can
iterate over the collection of :class:`_engine.Row` objects directly.

The :class:`_engine.Row` objects themselves are intended to act like Python
`named tuples
<https://docs.python.org/3/library/collections.html#collections.namedtuple>`_.
Below we illustrate a variety of ways to access rows.

* **Tuple Assignment** - This is the most Python-idiomatic style, which is to assign variables
  to each row positionally as they are received:

  ::

      result = conn.execute(text("select x, y from some_table"))

      for x, y in result:
          # ...

* **Integer Index** - Tuples are Python sequences, so regular integer access is available too:

  ::

      result = conn.execute(text("select x, y from some_table"))

        for row in result:
            x = row[0]

* **Attribute Name** - As these are Python named tuples, the tuples have dynamic attribute names
  matching the names of each column.  These names are normally the names that the
  SQL statement assigns to the columns in each row.  While they are usually
  fairly predictable and can also be controlled by labels, in less defined cases
  they may be subject to database-specific behaviors::

      result = conn.execute(text("select x, y from some_table"))

      for row in result:
          y = row.y

          # illustrate use with Python f-strings
          print(f"Row: {row.x} {row.y}")

  ..

* **Mapping Access** - To receive rows as Python **mapping** objects, which is
  essentially a read-only version of Python's interface to the common ``dict``
  object, the :class:`_engine.Result` may be **transformed** into a
  :class:`_engine.MappingResult` object using the
  :meth:`_engine.Result.mappings` modifier; this is a result object that yields
  dictionary-like :class:`_engine.RowMapping` objects rather than
  :class:`_engine.Row` objects::

      result = conn.execute(text("select x, y from some_table"))

      for dict_row in result.mappings():
          x = dict_row['x']
          y = dict_row['y']

  ..

.. rst-class:: orm-addin

.. _tutorial_sending_parameters:

Sending Parameters
^^^^^^^^^^^^^^^^^^

SQL statements are usually accompanied by data that is to be passed with the
statement itself, as we saw in the INSERT example previously. The
:meth:`_future.Connection.execute` method therefore also accepts parameters,
which are referred towards as :term:`bound parameters`.  A rudimentary example
might be if we wanted to limit our SELECT statement only to rows that meet a
certain criteria, such as rows where the "y" value were greater than a certain
value that is passed in to a function.

In order to achieve this such that the SQL statement can remain fixed and
that the driver can properly sanitize the value, we add a WHERE criteria to
our statement that names a new parameter called "y"; the :func:`_sql.text`
construct accepts these using a colon format "``:y``".   The actual value for
"``:y``" is then passed as the second argument to
:meth:`_future.Connection.execute` in the form of a dictionary:

.. sourcecode:: pycon+sql

    >>> with engine.connect() as conn:
    ...     result = conn.execute(
    ...         text("SELECT x, y FROM some_table WHERE y > :y"),
    ...         {"y": 2}
    ...     )
    ...     for row in result:
    ...        print(f"x: {row.x}  y: {row.y}")
    {opensql}BEGIN (implicit)
    SELECT x, y FROM some_table WHERE y > ?
    [...] (2,)
    {stop}x: 2  y: 4
    x: 6  y: 8
    x: 9  y: 10
    {opensql}ROLLBACK{stop}


In the logged SQL output, we can see that the bound parameter ``:y`` was
converted into a question mark when it was sent to the SQLite database.
This is because the SQLite database driver uses a format called "qmark parameter style",
which is one of six different formats allowed by the DBAPI specification.
SQLAlchemy abstracts these formats into just one, which is the "named" format
using a colon.

.. _tutorial_multiple_parameters:

Sending Multiple Parameters
^^^^^^^^^^^^^^^^^^^^^^^^^^^

In the example at :ref:`tutorial_committing_data`, we executed an INSERT
statement where it appeared that we were able to INSERT multiple rows into the
database at once.  For statements that **operate upon data, but do not return
result sets**, namely :term:`DML` statements such as "INSERT" which don't
include a phrase like "RETURNING", we can send **multi params** to the
:meth:`_future.Connection.execute` method by passing a list of dictionaries
instead of a single dictionary, thus allowing the single SQL statement to
be invoked against each parameter set individually:

.. sourcecode:: pycon+sql

    >>> with engine.connect() as conn:
    ...     conn.execute(
    ...         text("INSERT INTO some_table (x, y) VALUES (:x, :y)"),
    ...         [{"x": 11, "y": 12}, {"x": 13, "y": 14}]
    ...     )
    ...     conn.commit()
    {opensql}BEGIN (implicit)
    INSERT INTO some_table (x, y) VALUES (?, ?)
    [...] ((11, 12), (13, 14))
    <sqlalchemy.engine.cursor.CursorResult object at 0x...>
    COMMIT

Behind the scenes, the :class:`_future.Connection` objects uses a DBAPI feature
known as `cursor.executemany()
<https://www.python.org/dev/peps/pep-0249/#id18>`_. This method performs the
equivalent operation of invoking the given SQL statement against each parameter
set individually.   The DBAPI may optimize this operation in a variety of ways,
by using prepared statements, or by concatenating the parameter sets into a
single SQL statement in some cases.  Some SQLAlchemy dialects may also use
alternate APIs for this case, such as the :ref:`psycopg2 dialect for PostgreSQL
<postgresql_psycopg2>` which uses more performant APIs
for this use case.

.. tip::  you may have noticed this section isn't tagged as an ORM concept.
   That's because the "multiple parameters" use case is **usually** used
   for INSERT statements, which when using the ORM are invoked in a different
   way.   Multiple parameters also may be used with UPDATE and DELETE
   statements to emit distinct UPDATE/DELETE operations on a per-row basis,
   however again when using the ORM, there is a different technique
   generally used for updating or deleting many individual rows separately.

.. rst-class:: orm-addin

.. _tutorial_bundling_parameters:

Bundling Parameters with a Statement
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

The two previous cases illustrate a series of parameters being passed to
accompany a SQL statement.    For single-parameter statement executions,
SQLAlchemy's use of parameters is in fact more often than not done by
**bundling** the parameters with the statement itself, which is a primary
feature of the SQL Expression Language and makes for queries that can be
composed naturally while still making use of parameterization in all cases.
This concept will be discussed in much more detail in the sections that follow;
for a brief preview, the :func:`_sql.text` construct itself being part of the
SQL Expression Language supports this feature by using the
:meth:`_sql.TextClause.bindparams` method; this is a :term:`generative` method that
returns a new copy of the SQL construct with additional state added, in this
case the parameter values we want to pass along:


.. sourcecode:: pycon+sql

    >>> stmt = text("SELECT x, y FROM some_table WHERE y > :y ORDER BY x, y").bindparams(y=6)
    >>> with engine.connect() as conn:
    ...     result = conn.execute(stmt)
    ...     for row in result:
    ...        print(f"x: {row.x}  y: {row.y}")
    {opensql}BEGIN (implicit)
    SELECT x, y FROM some_table WHERE y > ? ORDER BY x, y
    [...] (6,)
    {stop}x: 6  y: 8
    x: 9  y: 10
    x: 11  y: 12
    x: 13  y: 14
    {opensql}ROLLBACK{stop}


The interesting thing to note above is that even though we passed only a single
argument, ``stmt``, to the :meth:`_future.Connection.execute` method, the
execution of the statement illustrated both the SQL string as well as the
separate parameter tuple.

.. rst-class:: orm-addin

.. _tutorial_executing_orm_session:

Executing with an ORM Session
-----------------------------

As mentioned previously, most of the patterns and examples above apply to
use with the ORM as well, so here we will introduce this usage so that
as the tutorial proceeds, we will be able to illustrate each pattern in
terms of Core and ORM use together.

The fundamental transactional / database interactive object when using the
ORM is called the :class:`_orm.Session`.  In modern SQLAlchemy, this object
is used in a manner very similar to that of the :class:`_future.Connection`,
and in fact as the :class:`_orm.Session` is used, it refers to a
:class:`_future.Connection` internally which it uses to emit SQL.

When the :class:`_orm.Session` is used with non-ORM constructs, it
passes through the SQL statements we give it and does not generally do things
much differently from how the :class:`_future.Connection` does directly, so
we can illustrate it here in terms of the simple textual SQL
operations we've already learned.

The :class:`_orm.Session` has a few different creational patterns, but
here we will illustrate the most basic one that tracks exactly with how
the :class:`_future.Connection` is used which is to construct it within
a context manager:

.. sourcecode:: pycon+sql

    >>> from sqlalchemy.orm import Session

    >>> stmt = text("SELECT x, y FROM some_table WHERE y > :y ORDER BY x, y").bindparams(y=6)
    >>> with Session(engine) as session:
    ...     result = session.execute(stmt)
    ...     for row in result:
    ...        print(f"x: {row.x}  y: {row.y}")
    {opensql}BEGIN (implicit)
    SELECT x, y FROM some_table WHERE y > ? ORDER BY x, y
    [...] (6,){stop}
    x: 6  y: 8
    x: 9  y: 10
    x: 11  y: 12
    x: 13  y: 14
    {opensql}ROLLBACK{stop}

The example above can be compared to the example in the preceding section
in :ref:`tutorial_bundling_parameters` - we directly replace the call to
``with engine.connect() as conn`` with ``with Session(engine) as session``,
and then make use of the :meth:`_orm.Session.execute` method just like we
do with the :meth:`_future.Connection.execute` method.

Also, like the :class:`_future.Connection`, the :class:`_orm.Session` features
"commit as you go" behavior using the :meth:`_orm.Session.commit` method,
illustrated below using a textual UPDATE statement to alter some of
our data:

.. sourcecode:: pycon+sql

    >>> with Session(engine) as session:
    ...     result = session.execute(
    ...         text("UPDATE some_table SET y=:y WHERE x=:x"),
    ...         [{"x": 9, "y":11}, {"x": 13, "y": 15}]
    ...     )
    ...     session.commit()
    {opensql}BEGIN (implicit)
    UPDATE some_table SET y=? WHERE x=?
    [...] ((11, 9), (15, 13))
    COMMIT{stop}

Above, we invoked an UPDATE statement using the bound-parameter, "executemany"
style of execution introduced at :ref:`tutorial_multiple_parameters`, ending
the block with a "commit as you go" commit.

.. tip:: The :class:`_orm.Session` doesn't actually hold onto the
   :class:`_future.Connection` object after it ends the transaction.  It
   gets a new :class:`_future.Connection` from the :class:`_future.Engine`
   when executing SQL against the database is next needed.

The :class:`_orm.Session` obviously has a lot more tricks up its sleeve
than that, however understanding that it has an :meth:`_orm.Session.execute`
method that's used the same way as :meth:`_future.Connection.execute` will
get us started with the examples that follow later.


.. _tutorial_working_with_metadata:

Working with Database Metadata
==============================

With engines and SQL execution down, we are ready to begin some Alchemy.
The central element of both SQLAlchemy Core and ORM is the SQL Expression
Language which allows for fluent, composable construction of SQL queries.
The foundation for these queries are Python objects that represent database
concepts like tables and columns.   These objects are known collectively
as :term:`database metadata`.

The most common foundational objects for database metadata in SQLAlchemy are
known as  :class:`_schema.MetaData`, :class:`_schema.Table`, and :class:`_schema.Column`.
The sections below will illustrate how these objects are used in both a
Core-oriented style as well as an ORM-oriented style.

.. container:: orm-header

    **ORM readers, stay with us!**

    As with other sections, Core users can skip the ORM sections, but ORM users
    would best be familiar with these objects from both perspectives.


.. rst-class:: core-header

.. _tutorial_core_metadata:

Setting up MetaData with Table objects
---------------------------------------

When we work with a relational database, the basic structure that we create and
query from is known as a **table**.   In SQLAlchemy, the "table" is represented
by a Python object similarly named :class:`_schema.Table`.

To start using the SQLAlchemy Expression Language,
we will want to have :class:`_schema.Table` objects constructed that represent
all of the database tables we are interested in working with.   Each
:class:`_schema.Table` may be **declared**, meaning we explicitly spell out
in source code what the table looks like, or may be **reflected**, which means
we generate the object based on what's already present in a particular database.
The two approaches can also be blended in many ways.

Whether we will declare or reflect our tables, we start out with a collection
that will be where we place our tables known as the :class:`_schema.MetaData`
object.  This object is essentially a :term:`facade` around a Python dictionary
that stores a series of :class:`_schema.Table` objects keyed to their string
name.   Constructing this object looks like::

    >>> from sqlalchemy import MetaData
    >>> metadata = MetaData()

Having a single :class:`_schema.MetaData` object for an entire application is
the most common case, represented as a module-level variable in a single place
in an application, often in a "models" or "dbschema" type of package.  There
can be multiple :class:`_schema.MetaData` collections as well,  however
it's typically most helpful if a series :class:`_schema.Table` objects that are
related to each other belong to a single :class:`_schema.MetaData` collection.


Once we have a :class:`_schema.MetaData` object, we can declare some
:class:`_schema.Table` objects.  This tutorial will start with the classic
SQLAlchemy tutorial model, that of the table ``user``, which would for
example represent the users of a website, and the table ``address``,
representing a list of email addresses associated with rows in the ``user``
table.   We normally assign each :class:`_schema.Table` object to a variable
that will be how we will refer to the table in application code::

    >>> from sqlalchemy import Table, Column, Integer, String
    >>> user_table = Table(
    ...     "user_account",
    ...     metadata,
    ...     Column('id', Integer, primary_key=True),
    ...     Column('name', String(30)),
    ...     Column('fullname', String)
    ... )

We can observe that the above :class:`_schema.Table` construct looks a lot like
a SQL CREATE TABLE statement; starting with the table name, then listing out
each column, where each column has a name and a datatype.   The objects we
use above are:

* :class:`_schema.Table` - represents a database table and assigns itself
  to a :class:`_schema.MetaData` collection.

* :class:`_schema.Column` - represents a column in a database table, and
  assigns itself to a :class:`_schema.Table` object.   The :class:`_schema.Column`
  usually includes a string name and a type object.   The collection of
  :class:`_schema.Column` objects in terms of the parent :class:`_schema.Table`
  are typically accessed via an associative array located at :attr:`_schema.Table.c`::

    >>> user_table.c.name
    Column('name', String(length=30), table=<user_account>)

    >>> user_table.c.keys()
    ['id', 'name', 'fullname']

* :class:`_types.Integer`, :class:`_types.String` - these classes represent
  SQL datatypes and can be passed to a :class:`_schema.Column` with or without
  necessarily being instantiated.  Above, we want to give a length of "30" to
  the "name" column, so we instantiated ``String(30)``.  But for "id" and
  "fullname" we did not specify these, so we can send the class itself.

.. seealso::

    The reference and API documentation for :class:`_schema.MetaData`,
    :class:`_schema.Table` and :class:`_schema.Column` is at :ref:`metadata_toplevel`.
    The reference documentation for datatypes is at :ref:`types_toplevel`.

In an upcoming section, we will illustrate one of the fundamental
functions of :class:`_schema.Table` which
is to generate :term:`DDL` on a particular database connection.  But first
we will declare a second :class:`_schema.Table`.

.. rst-class:: core-header

Declaring Simple Constraints
-----------------------------

The first :class:`_schema.Column` in the above ``user_table`` includes the
:paramref:`_schema.Column.primary_key` parameter which is a shorthand technique
of indicating that this :class:`_schema.Column` should be part of the primary
key for this table.  The primary key itself is normally declared implicitly
and is represented by the :class:`_schema.PrimaryKeyConstraint` construct,
which we can see on the :attr:`_schema.Table.primary_key`
attribute on the :class:`_schema.Table` object::

    >>> user_table.primary_key
    PrimaryKeyConstraint(Column('id', Integer(), table=<user_account>, primary_key=True, nullable=False))

The constraint that is most typically declared explicitly is the
:class:`_schema.ForeignKeyConstraint` object that corresponds to a database
:term:`foreign key constraint`.  When we declare tables that are related to
each other, SQLAlchemy uses the presence of these foreign key constraint
declarations not only so that they are emitted within CREATE statements to
the database, but also to assist in constructing SQL expressions.

A :class:`_schema.ForeignKeyConstraint` that involves only a single column
on the target table is typically declared using a column-level shorthand notation
via the :class:`_schema.ForeignKey` object.  Below we declare a second table
``address`` that will have a foreign key constraint referring to the ``user``
table::

    >>> from sqlalchemy import ForeignKey
    >>> address_table = Table(
    ...     "address",
    ...     metadata,
    ...     Column('id', Integer, primary_key=True),
    ...     Column('user_id', None, ForeignKey('user_account.id')),
    ...     Column('email_address', String, nullable=False)
    ... )

The table above also features a third kind of constraint, which in SQL is the
"NOT NULL" constraint, indicated above using the :paramref:`_schema.Column.nullable`
parameter.

In the next section we will emit the completed DDL for the ``user`` and
``address`` table to see the completed result.

.. rst-class:: core-header, orm-dependency


.. _tutorial_emitting_ddl:

Emitting DDL to the Database
----------------------------

We've constructed a fairly elaborate object hierarchy to represent
two database tables, starting at the root :class:`_schema.MetaData`
object, then into two :class:`_schema.Table` objects, each of which hold
onto a collection of :class:`_schema.Column` and :class:`_schema.Constraint`
objects.   This object structure will be at the center of most operations
we perform with both Core and ORM going forward.

The first useful thing we can do with this structure will be to emit CREATE
TABLE statements, or :term:`DDL`, to our SQLite database so that we can insert
and query data from them.   We have already all the tools needed to do so, by
invoking the
:meth:`_schema.MetaData.create_all` method on our :class:`_schema.MetaData`,
sending it the :class:`_future.Engine` that refers to the target database:

.. sourcecode:: pycon+sql

    >>> metadata.create_all(engine)
    {opensql}BEGIN (implicit)
    PRAGMA main.table_info("user_account")
    ...
    PRAGMA main.table_info("address")
    ...
    CREATE TABLE user_account (
        id INTEGER NOT NULL,
        name VARCHAR(30),
        fullname VARCHAR,
        PRIMARY KEY (id)
    )
    ...
    CREATE TABLE address (
        id INTEGER NOT NULL,
        user_id INTEGER,
        email_address VARCHAR NOT NULL,
        PRIMARY KEY (id),
        FOREIGN KEY(user_id) REFERENCES user_account (id)
    )
    ...
    COMMIT

The DDL create process by default includes some SQLite-specific PRAGMA statements
that test for the existence of each table before emitting a CREATE.   The full
series of steps are also included within a BEGIN/COMMIT pair to accommodate
for transactional DDL (SQLite does actually support transactional DDL, however
the ``sqlite3`` database driver historically runs DDL in "autocommit" mode).

The create process also takes care of emitting CREATE statements in the correct
order; above, the FOREIGN KEY constraint is dependent on the ``user`` table
existing, so the ``address`` table is created second.   In more complicated
dependency scenarios the FOREIGN KEY constraints may also be applied to tables
after the fact using ALTER.

The :class:`_schema.MetaData` object also features a
:meth:`_schema.MetaData.drop_all` method that will emit DROP statements in the
reverse order as it would emit CREATE in order to drop schema elements.

.. topic:: Migration tools are usually appropriate

    Overall, the CREATE / DROP feature of :class:`_schema.MetaData` is useful
    for test suites, small and/or new applications, and applications that use
    short-lived databases.  For management of an application database schema
    over the long term however, a schema management tool such as `Alembic
    <https://alembic.sqlalchemy.org>`_, which builds upon SQLAlchemy, is likely
    a better choice, as it can manage and orchestrate the process of
    incrementally altering a fixed database schema over time as the design of
    the application changes.


.. rst-class:: orm-header

.. _tutorial_orm_table_metadata:

Defining Table Metadata with the ORM
------------------------------------

This ORM-only section will provide an example of the declaring the
same database structure illustrated in the previous section, using a more
ORM-centric configuration paradigm.   When using
the ORM, the process by which we declare :class:`_schema.Table` metadata
is usually combined with the process of declaring :term:`mapped` classes.
The mapped class is any Python class we'd like to create, which will then
have attributes on it that will be linked to the columns in a database table.
While there are a few varieties of how this is achieved, the most common
style is known as
:ref:`declarative <orm_declarative_mapper_config_toplevel>`, and allows us
to declare our user-defined classes and :class:`_schema.Table` metadata
at once.

Setting up the Registry
^^^^^^^^^^^^^^^^^^^^^^^

When using the ORM, the :class:`_schema.MetaData` collection remains present,
however it itself is contained within an ORM-only object known as the
:class:`_orm.registry`.   We create a :class:`_orm.registry` by constructing
it::

    >>> from sqlalchemy.orm import registry
    >>> mapper_registry = registry()

The above :class:`_orm.registry`, when constructed, automatically includes
a :class:`_schema.MetaData` object that will store a collection of
:class:`_schema.Table` objects::

    >>> mapper_registry.metadata
    MetaData()

Instead of declaring :class:`_schema.Table` objects directly, we will now
declare them indirectly through directives applied to our mapped classes. In
the most common approach, each mapped class descends from a common base class
known as the **declarative base**.   We get a new declarative base from the
:class:`_orm.registry` using the :meth:`_orm.registry.generate_base` method::

    >>> Base = mapper_registry.generate_base()

.. tip::

    The steps of creating the :class:`_orm.registry` and "declarative base"
    classes can be combined into one step using the historically familiar
    :func:`_orm.declarative_base` function::

        from sqlalchemy.orm import declarative_base
        Base = declarative_base()

    ..

.. _tutorial_declaring_mapped_classes:

Declaring Mapped Classes
^^^^^^^^^^^^^^^^^^^^^^^^

The ``Base`` object above is a Python class which will serve as the base class
for the ORM mapped classes we declare.  We can now define ORM mapped classes
for the ``user`` and ``address`` table in terms of new classes ``User`` and
``Address``::

    >>> from sqlalchemy.orm import relationship
    >>> class User(Base):
    ...     __tablename__ = 'user_account'
    ...
    ...     id = Column(Integer, primary_key=True)
    ...     name = Column(String(30))
    ...     fullname = Column(String)
    ...
    ...     addresses = relationship("Address", back_populates="user")
    ...
    ...     def __repr__(self):
    ...        return f"User(id={self.id!r}, name={self.name!r}, fullname={self.fullname!r})"

    >>> class Address(Base):
    ...     __tablename__ = 'address'
    ...
    ...     id = Column(Integer, primary_key=True)
    ...     email_address = Column(String, nullable=False)
    ...     user_id = Column(Integer, ForeignKey('user_account.id'))
    ...
    ...     user = relationship("User", back_populates="addresses")
    ...
    ...     def __repr__(self):
    ...         return f"Address(id={self.id!r}, email_address={self.email_address!r})"

The above two classes are now our mapped classes, and are available for use in
ORM persistence and query operations, which will be described later. But they
also include :class:`_schema.Table` objects that were generated as part of the
declarative mapping process, and are equivalent to the ones that we declared
directly in the previous Core section.   We can see these
:class:`_schema.Table` objects from a declarative mapped class using the
``.__table__`` attribute::

    >>> User.__table__
    Table('user_account', MetaData(),
        Column('id', Integer(), table=<user_account>, primary_key=True, nullable=False),
        Column('name', String(length=30), table=<user_account>),
        Column('fullname', String(), table=<user_account>), schema=None)

This :class:`_schema.Table` object was generated from the declarative process
based on the ``.__tablename__`` attribute defined on each of our classes,
as well as through the use of :class:`_schema.Column` objects assigned
to class-level attributes within the classes.   These :class:`_schema.Column`
objects can usually be declared without an explicit "name" field inside
the constructor, as the Declarative process will name them automatically
based on the attribute name that was used.

Other Mapped Class Details
^^^^^^^^^^^^^^^^^^^^^^^^^^^

For a few quick explanations for the classes above, note the following
attributes:

* **the classes have an automatically generated __init__() method** - both classes by default
  receive an ``__init__()`` method that allows for parameterized construction
  of the objects.  We are free to provide our own ``__init__()`` method as well.
  The ``__init__()`` allows us to create instances of ``User`` and ``Address``
  passing attribute names, most of which above are linked directly to
  :class:`_schema.Column` objects, as parameter names::

    >>> sandy = User(name="sandy", fullname="Sandy Cheeks")

  More detail on this method is at :ref:`mapped_class_default_constructor`.

  ..

* **we provided a __repr__() method** - this is **fully optional**, and is
  strictly so that our custom classes have a descriptive string representation
  and is not otherwise required::

    >>> sandy
    User(id=None, name='sandy', fullname='Sandy Cheeks')

  ..

  An interesting thing to note above is that the ``id`` attribute automatically
  returns ``None`` when accessed, rather than raising ``AttributeError`` as
  would be the usual Python behavior for missing attributes.

* **we also included a bidirectional relationship** - this  is another **fully optional**
  construct, where we made use of an ORM construct called
  :func:`_orm.relationship` on both classes, which indicates to the ORM that
  these ``User`` and ``Address`` classes refer to each other in a :term:`one to
  many` / :term:`many to one` relationship.  The use of
  :func:`_orm.relationship` above is so that we may demonstrate its behavior
  later in this tutorial; it is  **not required** in order to define the
  :class:`_schema.Table` structure.


Emitting DDL to the database
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

This section is named the same as the section :ref:`tutorial_emitting_ddl`
discussed in terms of Core.   This is because emitting DDL with our
ORM mapped classes is not any different.  If we wanted to emit DDL
for the :class:`_schema.Table` objects we've created as part of
our declaratively mapped classes, we still can use
:meth:`_schema.MetaData.create_all` as before.

In our case, we have already generated the ``user`` and ``address`` tables
in our SQLite database.   If we had not done so already, we would be free to
make use of the :class:`_schema.MetaData` associated with our
:class:`_orm.registry` and ORM declarative base class in order to do so,
using :meth:`_schema.MetaData.create_all`::

    # emit CREATE statements given ORM registry
    mapper_registry.metadata.create_all(engine)

    # the identical MetaData object is also present on the
    # declarative base
    Base.metadata.create_all(engine)


Combining Core Table Declarations with ORM Declarative
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

As an alternative approach to the mapping process shown previously
at :ref:`tutorial_declaring_mapped_classes`, we may also make
use of the :class:`_schema.Table` objects we created directly in the section
:ref:`tutorial_core_metadata` in conjunction with
declarative mapped classes from a :func:`_orm.declarative_base` generated base
class.

This form is called  :ref:`hybrid table <orm_imperative_table_configuration>`,
and it consists of assigning to the ``.__table__`` attribute directly, rather
than having the declarative process generate it::

    class User(Base):
        __table__ = user_table

         addresses = relationship("Address", back_populates="user")

         def __repr__(self):
            return f"User({self.name!r}, {self.fullname!r})"

    class Address(Base):
        __table__ = address_table

         user = relationship("User", back_populates="addresses")

         def __repr__(self):
             return f"Address({self.email_address!r})"

The above two classes are equivalent to those which we declared in the
previous mapping example.

The traditional "declarative base" approach using ``__tablename__`` to
automatically generate :class:`_schema.Table` objects remains the most popular
method to declare table metadata.  However, disregarding the ORM mapping
functionality it achieves, as far as table declaration it's merely a syntactical
convenience on top of the :class:`_schema.Table` constructor.

We will next refer to our ORM mapped classes above when we talk about data
manipulation in terms of the ORM, in the section :ref:`tutorial_inserting_orm`.


.. rst-class:: core-header

.. _tutorial_table_reflection:

Table Reflection
-------------------------------

To round out the section on working with table metadata, we will illustrate
another operation that was mentioned at the beginning of the section,
that of **table reflection**.   Table reflection refers to the process of
generating :class:`_schema.Table` and related objects by reading the current
state of a database.   Whereas in the previous sections we've been declaring
:class:`_schema.Table` objects in Python and then emitting DDL to the database,
the reflection process does it in reverse.

As an example of reflection, we will create a new :class:`_schema.Table`
object which represents the ``some_table`` object we created manually in
the earler sections of this document.  There are again some varieties of
how this is performed, however the most basic is to construct a
:class:`_schema.Table` object, given the name of the table and a
:class:`_schema.MetaData` collection to which it will belong, then
instead of indicating individual :class:`_schema.Column` and
:class:`_schema.Constraint` objects, pass it the target :class:`_future.Engine`
using the :paramref:`_schema.Table.autoload_with` parameter:

.. sourcecode:: pycon+sql

    >>> some_table = Table("some_table", metadata, autoload_with=engine)
    {opensql}BEGIN (implicit)
    PRAGMA main.table_info("some_table")
    [raw sql] ()
    SELECT sql FROM  (SELECT * FROM sqlite_master UNION ALL   SELECT * FROM sqlite_temp_master) WHERE name = ? AND type = 'table'
    [raw sql] ('some_table',)
    PRAGMA main.foreign_key_list("some_table")
    ...
    PRAGMA main.index_list("some_table")
    ...
    ROLLBACK{stop}

At the end of the process, the ``some_table`` object now contains the
information about the :class:`_schema.Column` objects present in the table, and
the object is usable in exactly the same way as a :class:`_schema.Table` that
we declared explicitly.::

    >>> some_table
    Table('some_table', MetaData(),
        Column('x', INTEGER(), table=<some_table>),
        Column('y', INTEGER(), table=<some_table>),
        schema=None)

.. seealso::

    Read more about table and schema reflection at :ref:`metadata_reflection_toplevel`.

    For ORM-related variants of table reflection, the section
    :ref:`orm_declarative_reflected` includes an overview of the available
    options.

.. _tutorial_working_with_data:

Working with Data
==================

In :ref:`tutorial_working_with_transactions` we learned the basics of how
to interact with the Python DBAPI and its transactional state.  Then,
in :ref:`tutorial_working_with_metadata` we learned how to represent
database tables, columns, and constraints within SQLAlchemy using the
:class:`_schema.MetaData` and related objects.

In this section we will combine both concepts above to create, select
and manipulate data within a relational database.   As always, our interaction
with the database is **always** in terms of a transaction, even if we've
set our database driver to use :ref:`autocommit <dbapi_autocommit>` behind the scenes.

The four fundamental operations in SQL can be defined as the :term:`DML`
INSERT, UPDATE and DELETE constructs and the :term:`DQL` SELECT construct.

This section will discuss data selection and manipulation primarily from
a Core perspective, using the SQL Expression Language.  This API represents
the largest part of SQLAlchemy's front-facing API, allowing programmatic techniques of
generating all four of SELECT, INSERT, UPDATE, DELETE.   Whereas in the
section at :ref:`tutorial_working_with_transactions` briefly introduced us to
the :func:`_sql.text` construct for creating SQL statements from strings,
typical use of the SQL Expression Language builds on constructs that are
typically composed by passing structures based on :class:`_schema.Table` and
:class:`_schema.Column` objects.

The following section, :ref:`tutorial_orm_data_manipulation` will focus on the ORM, filling out the major usage patterns of the :class:`_orm.Session` object and discussing additional details of how the SQL constructs
introduced here are integrated.

.. rst-class:: core-header

.. _tutorial_core_insert:

Core Insert
-----------

When using Core, a SQL INSERT statement is generated using the
:func:`_sql.insert` function - this function generates a new instance of
:class:`_sql.Insert` which represents an INSERT statement in SQL, that adds
new data into a table.

.. container:: orm-header

    **ORM Readers** - The way that rows are INSERTed into the database from an ORM
    perspective makes use of object-centric APIs on the :class:`_orm.Session` object known as the
    :term:`unit of work` process,
    and is fairly different from the Core-only approach described here.
    The more ORM-focused sections later starting at :ref:`tutorial_inserting_orm`
    subsequent to the Expression Language sections introduce this.

The insert() SQL Expression Construct
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

A simple example of :class:`_sql.Insert` illustrates the target table
and the VALUES clause at once::

    >>> from sqlalchemy import insert
    >>> stmt = insert(user_table).values(name='spongebob', fullname="Spongebob Squarepants")

The above ``stmt`` variable is an instance of :class:`_sql.Insert`.  Most
SQL expressions can be stringified in place as a means to see the general
form of what's being produced::

    >>> print(stmt)
    INSERT INTO user_account (name, fullname) VALUES (:name, :fullname)

The stringified form is created by producing a :class:`_engine.Compiled` form
of the object which includes a database-specific string SQL representation of
the statement; we can acquire this object directly using the
:meth:`_sql.ClauseElement.compile` method::

    >>> compiled = stmt.compile()

Our :class:`_sql.Insert` construct is an example of a "parameterized"
construct, illustrated previously at :ref:`tutorial_sending_parameters`; to
view the ``name`` and ``fullname`` :term:`bound parameters`, these are
available from the :class:`_engine.Compiled` construct as well::

    >>> compiled.params
    {'name': 'spongebob', 'fullname': 'Spongebob Squarepants'}

Executing the Statement
^^^^^^^^^^^^^^^^^^^^^^^

Invoking the statement we can INSERT a row into ``user_table``.
The INSERT SQL as well as the bundled parameters can be seen in the
SQL logging:

.. sourcecode:: pycon+sql

    >>> with engine.connect() as conn:
    ...     result = conn.execute(stmt)
    ...     conn.commit()
    {opensql}BEGIN (implicit)
    INSERT INTO user_account (name, fullname) VALUES (?, ?)
    [...] ('spongebob', 'Spongebob Squarepants')
    COMMIT

In its simple form above, the INSERT statement does not return any rows, and if
only a single row is inserted, it will usually include the ability to return
information about column-level default values that were generated during the
INSERT of that row, most commonly an integer primary key value.  In the above
case the first row in a SQLite database will normally return ``1`` for the
first integer primary key value, which we can acquire using the
:attr:`_engine.CursorResult.inserted_primary_key` accessor:

.. sourcecode:: pycon+sql

    >>> result.inserted_primary_key
    (1,)

.. tip:: :attr:`_engine.CursorResult.inserted_primary_key` returns a tuple
   because a primary key may contain multiple columns.  This is known as
   a :term:`composite primary key`.  The :attr:`_engine.CursorResult.inserted_primary_key`
   is intended to always contain the complete primary key of the record just
   inserted, not just a "cursor.lastrowid" kind of value, and is also intended
   to be populated regardless of whether or not "autoincrement" were used, hence
   to express a complete primary key it's a tuple.

The INSERT construct features many variations to its general form and behavior,
including that the INSERT supports returning rows via RETURNING, that
multiple VALUES clauses may be rendered at once, and that some
database backends can return information about column defaults for more than
one row at a time.  The reference guide at :ref:`queryguide_insert` details
these other features.

INSERT usually generates the "values" clause automatically
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

The example above made use of the :meth:`_sql.Insert.values` method to
explicitly create the VALUES clause of the SQL INSERT statement.   This method
in fact has some variants that allow for special forms such as multiple rows in
one statement and insertion of SQL expressions.   However the usual way that
:class:`_sql.Insert` is used is such that the VALUES clause is generated
automatically from the parameters passed to the
:meth:`_future.Connection.execute` method; below we INSERT two more rows to
illustrate this:

.. sourcecode:: pycon+sql

    >>> with engine.connect() as conn:
    ...     result = conn.execute(
    ...         insert(user_table),
    ...         [
    ...             {"name": "sandy", "fullname": "Sandy Cheeks"},
    ...             {"name": "patrick", "fullname": "Patrick Star"}
    ...         ]
    ...     )
    ...     conn.commit()
    {opensql}BEGIN (implicit)
    INSERT INTO user_account (name, fullname) VALUES (?, ?)
    [...] (('sandy', 'Sandy Cheeks'), ('patrick', 'Patrick Star'))
    COMMIT{stop}

The execution above features "executemany" form first illustrated at
:ref:`tutorial_multiple_parameters`, however unlike when using the
:func:`_sql.text` construct, we didn't have to spell out any SQL.
By passing a dictionary or list of dictionaries to the :meth:`_future.Connection.execute`
method in conjunction with the :class:`_sql.Insert` construct, the
:class:`_future.Connection` ensures that the column names which are passed
will be expressed in the VALUES clause of the :class:`_sql.Insert`
construct automatically.

.. deepalchemy::

    Hi, welcome to the first edition of **Deep Alchemy**.   The person on the
    left is known as **The Alchemist**, and you'll note they are **not** a wizard,
    as the pointy hat is not sticking upwards.   The Alchemist comes around to
    describe things that are generally **more advanced and/or tricky** and
    additionally **not usually needed**, but for whatever reason they feel you
    should know about this thing that SQLAlchemy can do.

    In this edition, towards the goal of having some interesting data in the
    ``address_table`` as well, below is a more advanced example illustrating
    how the :meth:`_sql.Insert.values` method may be used explicitly while at
    the same time including for additional VALUES generated from the
    parameters.    A :term:`scalar subquery` is constructed, making use of the
    :func:`_sql.select` construct introduced in the next section, and the
    parameters used in the subquery are set up using an explicit bound
    parameter name, established using the :func:`_sql.bindparam` construct.

    This is some slightly **deeper** alchemy just so that we can add related
    rows without fetching the primary key identifiers from the ``user_table``
    operation into the application.   Most Alchemists will simply use the ORM
    which takes care of things like this for us.

    .. sourcecode:: pycon+sql

        >>> from sqlalchemy import select, bindparam
        >>> scalar_subquery = (
        ...     select(user_table.c.id).
        ...     where(user_table.c.name==bindparam('username')).
        ...     scalar_subquery()
        ... )

        >>> with engine.connect() as conn:
        ...     result = conn.execute(
        ...         insert(address_table).values(user_id=scalar_subquery),
        ...         [
        ...             {"username": 'spongebob', "email_address": "spongebob@sqlalchemy.org"},
        ...             {"username": 'sandy', "email_address": "sandy@sqlalchemy.org"},
        ...             {"username": 'sandy', "email_address": "sandy@squirrelpower.org"},
        ...             {"username": 'patrick', "email_address": "pat999@aol.com"},
        ...         ]
        ...     )
        ...     conn.commit()
        {opensql}BEGIN (implicit)
        INSERT INTO address (user_id, email_address) VALUES ((SELECT user_account.id
        FROM user_account
        WHERE user_account.name = ?), ?)
        [...] (('spongebob', 'spongebob@sqlalchemy.org'), ('sandy', 'sandy@sqlalchemy.org'),
        ('sandy', 'sandy@squirrelpower.org'), ('patrick', 'pat999@aol.com'))
        COMMIT{stop}


.. _tutorial_selecting_data:

.. rst-class:: core-header, orm-dependency

Selecting Data
--------------

For both Core and ORM, the :func:`_sql.select` function generates a
:class:`_sql.Select` construct which is used for all SELECT queries.
Passed to methods like :meth:`_future.Connection.execute` in Core and
:meth:`_orm.Session.execute` in ORM, a SELECT statement is emitted in the
current transaction and the result rows available via the returned
:class:`_engine.Result` object.


The select() SQL Expression Construct
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

The :func:`_sql.select` construct builds up a statement in the same way
as that of :func:`_sql.insert`, using a :term:`generative` approach where
each method builds more state onto the object.  Like the other SQL constructs,
it can be stringified in place::

    >>> from sqlalchemy import select
    >>> stmt = select(user_table).where(user_table.c.name == 'spongebob')
    >>> print(stmt)
    SELECT user_account.id, user_account.name, user_account.fullname
    FROM user_account
    WHERE user_account.name = :name_1

Also in the same manner as all other statement-level SQL constructs, to
actually run the statement we pass it to an execution method.
Since a SELECT statement returns
rows we can always iterate the result object to get :class:`_engine.Row`
objects back:

.. sourcecode:: pycon+sql

    >>> with engine.connect() as conn:
    ...     for row in conn.execute(stmt):
    ...         print(row)
    {opensql}BEGIN (implicit)
    SELECT user_account.id, user_account.name, user_account.fullname
    FROM user_account
    WHERE user_account.name = ?
    [...] ('spongebob',){stop}
    (1, 'spongebob', 'Spongebob Squarepants')
    {opensql}ROLLBACK{stop}

When using the ORM, particularly with a :func:`_sql.select` construct that's
composed against ORM entities, we will want to execute it using the
:meth:`_orm.Session.execute` method on the :class:`_orm.Session`; using
this approach, we continue to get :class:`_engine.Row` objects from the
result, however these rows are now capable of including
complete entities, such as instances of the ``User`` class, as column values:

.. sourcecode:: pycon+sql

    >>> stmt = select(User).where(User.name == 'spongebob')
    >>> with Session(engine) as session:
    ...     for row in session.execute(stmt):
    ...         print(row)
    {opensql}BEGIN (implicit)
    SELECT user_account.id, user_account.name, user_account.fullname
    FROM user_account
    WHERE user_account.name = ?
    [...] ('spongebob',){stop}
    (User(id=1, name='spongebob', fullname='Spongebob Squarepants'),)
    {opensql}ROLLBACK{stop}

The following sections will discuss the SELECT construct in more detail.


Setting the COLUMNS and FROM clause
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

The :func:`_sql.select` function accepts positional elements representing any
number of :class:`_schema.Column` and/or :class:`_schema.Table` expressions, as
well as a wide range of compatible objects, which are resolved into a list of SQL
expressions to be SELECTed from that will be returned as columns in the result
set.  These elements also serve in simpler cases to create the FROM clause,
which is inferred from the columns and table-like expressions passed::

    >>> print(select(user_table))
    SELECT user_account.id, user_account.name, user_account.fullname
    FROM user_account

To SELECT from individual columns using a Core approach,
:class:`_schema.Column` objects are accessed from the :attr:`_schema.Table.c`
accessor and can be sent directly; the FROM clause will be inferred as the set
of all :class:`_schema.Table` and other :class:`_sql.FromClause` objects that
are represented by those columns::

    >>> print(select(user_table.c.name, user_table.c.fullname))
    SELECT user_account.name, user_account.fullname
    FROM user_account

.. _tutorial_selecting_orm_entities:

Selecting ORM Entities and Columns
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

ORM entities, such our ``User`` class as well as the column-mapped
attributes upon it such as ``User.name``, also participate in the SQL Expression
Language system representing tables and columns.    Below illustrates an
example of SELECTing from the ``User`` entity, which ultimately renders
in the same way as if we had used ``user_table`` directly::

    >>> print(select(User))
    SELECT user_account.id, user_account.name, user_account.fullname
    FROM user_account

To select from individual columns using ORM entities, the class-bound
attributes can be passed directly which are resolved into the
:class:`_schema.Column` or other SQL expression represented by each attribute::

    >>> print(select(User.name, User.fullname))
    SELECT user_account.name, user_account.fullname
    FROM user_account

.. tip::

    When ORM-related objects are used within the :class:`_sql.Select`
    construct, they are resolved into the underlying :class:`_schema.Table` and
    :class:`_schema.Column` and similar Core constructs they represent; at the
    same time, they apply a **plugin** to the core :class:`_sql.Select`
    construct such that a new set of ORM-specific behaviors make take
    effect when the construct is being compiled.

.. seealso::

    :ref:`orm_queryguide_select_columns` - in the :doc:`orm/queryguide`

Selecting from Labeled SQL Expressions
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

The :meth:`_sql.ColumnElement.label` method as well as the same-named method
available on ORM attributes provides a SQL label of a column or expression,
allowing it to have a specific name in a result set.  This can be helpful
when referring to arbitrary SQL expressions in a result row by name:

.. sourcecode:: pycon+sql

    >>> from sqlalchemy import func, cast
    >>> stmt = (
    ...     select(
    ...         ("Username: " + user_table.c.name).label("username"),
    ...     ).order_by(user_table.c.name)
    ... )
    >>> with engine.connect() as conn:
    ...     for row in conn.execute(stmt):
    ...         print(f"{row.username}")
    {opensql}BEGIN (implicit)
    SELECT ? || user_account.name AS username
    FROM user_account ORDER BY user_account.name
    [...] ('Username: ',){stop}
    Username: patrick
    Username: sandy
    Username: spongebob
    {opensql}ROLLBACK{stop}

.. _tutorial_select_where_clause:

The WHERE clause
^^^^^^^^^^^^^^^^

SQLAlchemy allows us to compose SQL expressions, such as ``name = 'squidward'``
or ``user_id > 10``, by making use of standard Python operators in
conjunction with
:class:`_schema.Column` and similar objects.   For boolean expressions, most
Python operators such as ``==``, ``!=``, ``<``, ``>=`` etc. generate new
SQL Expression objects, rather than plain boolean True/False values::

    >>> print(user_table.c.name == 'squidward')
    user_account.name = :name_1

    >>> print(address_table.c.user_id > 10)
    address.user_id > :user_id_1


We can use expressions like these to generate the WHERE clause by passing
the resulting objects to the :meth:`_sql.Select.where` method::

    >>> print(select(user_table).where(user_table.c.name == 'squidward'))
    SELECT user_account.id, user_account.name, user_account.fullname
    FROM user_account
    WHERE user_account.name = :name_1


To produce multiple expressions joined by AND, the :meth:`_sql.Select.where`
method may be invoked any number of times::

    >>> print(
    ...     select(address_table.c.email_address).
    ...     where(user_table.c.name == 'squidward').
    ...     where(address_table.c.user_id == user_table.c.id)
    ... )
    SELECT address.email_address
    FROM address, user_account
    WHERE user_account.name = :name_1 AND address.user_id = user_account.id

A single call to :meth:`_sql.Select.where` also accepts multiple expressions
with the same effect::

    >>> print(
    ...     select(address_table.c.email_address).
    ...     where(
    ...          user_table.c.name == 'squidward',
    ...          address_table.c.user_id == user_table.c.id
    ...     )
    ... )
    SELECT address.email_address
    FROM address, user_account
    WHERE user_account.name = :name_1 AND address.user_id = user_account.id

"AND" and "OR" conjunctions are both available directly using the
:func:`_sql.and_` and :func:`_sql.or_` functions, illustrated below in terms
of ORM entities::

    >>> from sqlalchemy import and_, or_
    >>> print(
    ...     select(Address.email_address).
    ...     where(
    ...         and_(
    ...             or_(User.name == 'squidward', User.name == 'sandy'),
    ...             Address.user_id == User.id
    ...         )
    ...     )
    ... )
    SELECT address.email_address
    FROM address, user_account
    WHERE (user_account.name = :name_1 OR user_account.name = :name_2)
    AND address.user_id = user_account.id

For simple "equality" comparisons against a single entity, there's also a
popular method known as :meth:`_sql.Select.filter_by` which accepts keyword
arguments that match to column keys or ORM attribute names.  It will filter
against the leftmost FROM clause or the last entity joined::

    >>> print(
    ...     select(User).filter_by(name='spongebob', fullname='Spongebob Squarepants')
    ... )
    SELECT user_account.id, user_account.name, user_account.fullname
    FROM user_account
    WHERE user_account.name = :name_1 AND user_account.fullname = :fullname_1


.. seealso::


    :doc:`core/operators` - descriptions of most SQL operator functions in SQLAlhcemy



Explicit FROM clauses and JOINs
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

As mentioned previously, the FROM clause is usually **inferred**
based on the expressions that we are setting in the columns
clause as well as other elements of the :class:`_sql.Select`.

If we set a single column from a particular :class:`_schema.Table`
in the COLUMNS clause, it puts that :class:`_schema.Table` in the FROM
clause as well::

    >>> print(select(user_table.c.name))
    SELECT user_account.name
    FROM user_account

If we were to put columns from two tables, then we get a comma-separated FROM
clause::

    >>> print(select(user_table.c.name, address_table.c.email_address))
    SELECT user_account.name, address.email_address
    FROM user_account, address

In order to JOIN these two tables together, two methods that are
most straightforward are :meth:`_sql.Select.join_from`, which
allows us to indicate the left and right side of the JOIN explicitly::

    >>> print(
    ...     select(user_table.c.name, address_table.c.email_address).
    ...     join_from(user_table, address_table)
    ... )
    SELECT user_account.name, address.email_address
    FROM user_account JOIN address ON user_account.id = address.user_id


the other is the :meth:`_sql.Select.join` method, which indicates only the
right side of the JOIN, the left hand-side is inferred::

    >>> print(
    ...     select(user_table.c.name, address_table.c.email_address).
    ...     join(address_table)
    ... )
    SELECT user_account.name, address.email_address
    FROM user_account JOIN address ON user_account.id = address.user_id

    ..

.. sidebar::  The ON Clause is inferred

    When using :meth:`_sql.Select.join_from` or :meth:`_sql.Select.join`, we may
    observe that the ON clause of the join is also inferred for us in simple cases.
    More on that in the next section.

We also have the option add elements to the FROM clause explicitly, if it is not
inferred the way we want from the columns clause.  We use the
:meth:`_sql.Select.select_from` method to achieve this, as below
where we establish ``user_table`` as the first element in the FROM
clause and :meth:`_sql.Select.join` to establish ``address_table`` as
the second::

    >>> print(
    ...     select(address_table.c.email_address).
    ...     select_from(user_table).join(address_table)
    ... )
    SELECT address.email_address
    FROM user_account JOIN address ON user_account.id = address.user_id

Another example where we might want to use :meth:`_sql.Select.select_from`
is if our columns clause doesn't have enough information to provide for a
FROM clause.  For example, to SELECT from the common SQL expression
``count(*)``, we use a SQLAlchemy element known as :attr:`_sql.func` to
produce the SQL ``count()`` function::

    >>> from sqlalchemy import func
    >>> print (
    ...     select(func.count('*')).select_from(user_table)
    ... )
    SELECT count(:count_2) AS count_1
    FROM user_account

Setting the ON Clause
^^^^^^^^^^^^^^^^^^^^^

The previous examples on JOIN illustrated that the :class:`_sql.Select` construct
can join between two tables and produce the ON clause automatically.  This
occurs in those examples because the ``user_table`` and ``address_table``
:class:`_sql.Table` objects include a single :class:`_schema.ForeignKeyConstraint`
definition which is used to form this ON clause.

If the left and right targets of the join do not have such a constraint, or
there are multiple constraints in place, we need to specify the ON clause
directly.   Both :meth:`_sql.Select.join` and :meth:`_sql.Select.join_from`
accept an additional argument for the ON clause, which is stated using the
same SQL Expression mechanics as we saw about in :ref:`tutorial_select_where_clause`::

    >>> print(
    ...     select(address_table.c.email_address).
    ...     select_from(user_table).
    ...     join(address_table, user_table.c.id == address_table.c.user_id)
    ... )
    SELECT address.email_address
    FROM user_account JOIN address ON user_account.id = address.user_id

When using ORM entities, an additional mechanism is available to help us set up
the ON clause of a join, which is to make use of the :func:`_orm.relationship`
objects that we set up in our user mapping, as was demonstrated at
:ref:`tutorial_declaring_mapped_classes`. The class-bound attribute
corresponding to the :func:`_orm.relationship` may be passed as the **single
argument** to :meth:`_sql.Select.join`, where it serves to indicate both the
right side of the join as well as the ON clause at once::

    >>> print(
    ...     select(Address.email_address).
    ...     select_from(User).
    ...     join(User.addresses)
    ... )
    SELECT address.email_address
    FROM user_account JOIN address ON user_account.id = address.user_id

The presence of an ORM :func:`_orm.relationship` on a mapping is not used
by :meth:`_sql.Select.join` or :meth:`_sql.Select.join_from` if we don't
specify it; it is **not used for ON clause
inference**.  This means, if we join from ``User`` to ``Address`` without an
ON clause, it works because of the :class:`_schema.ForeignKeyConstraint`
between the two mapped :class:`_schema.Table` objects, not because of the
:func:`_orm.relationship` objects on the ``User`` and ``Address`` classes::

    >>> print(
    ...    select(Address.email_address).
    ...    join_from(User, Address)
    ... )
    SELECT address.email_address
    FROM user_account JOIN address ON user_account.id = address.user_id

ORDER BY
^^^^^^^^^

The ORDER BY clause is constructed in terms
of SQL Expression constructs typically based on :class:`_schema.Column` or
similar objects.  The :meth:`_sql.Select.order_by` method accepts one or
more of these expressions positionally::

    >>> print(select(user_table).order_by(user_table.c.name))
    SELECT user_account.id, user_account.name, user_account.fullname
    FROM user_account ORDER BY user_account.name

Ascending / descending is available from the :meth:`_sql.ColumnElement.asc`
and :meth:`_sql.ColumnElement.desc` modifiers, which are present
from ORM-bound attributes as well::


    >>> print(select(User).order_by(User.name.asc(), User.fullname.desc()))
    SELECT user_account.id, user_account.name, user_account.fullname
    FROM user_account ORDER BY user_account.name ASC, user_account.fullname DESC

.. _tutorial_group_by_w_aggregates:

Aggregate functions with GROUP BY / HAVING
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

In SQL, aggregate functions allow column expressions across multiple rows
to be aggregated together to produce a single result.  Examples include
counting, computing averages, as well as locating the maximum or minimum
value in a set of values.

SQLAlchemy provides for SQL functions in an open-ended way using a namespace
known as :data:`_sql.func`.  This is a special constructor object which
will create new instances of :class:`_functions.Function` when given the name
of a particular SQL function, which can be any name, as well as zero or
more arguments to pass to the function, which are like in all other cases
SQL Expression constructs.   For example, to
render the SQL COUNT() function against the ``user_account.id`` column,
we call upon the name ``count()`` name::

    >>> from sqlalchemy import func
    >>> count_fn = func.count(user_table.c.id)
    >>> print(count_fn)
    count(user_account.id)

When using aggregate functions in SQL, the GROUP BY clause is essential in that
it allows rows to be partitioned into groups where aggregate functions will
be applied to each group individually.  When requesting non-aggregated columns
in the COLUMNS clause of a SELECT statement, SQL requires that these columns
all be subject to a GROUP BY clause, either directly or indirectly based on
a primary key association.    The HAVING clause is then used in a similar
manner as the WHERE clause, except that it filters out rows based on aggregated
values rather than direct row contents.

SQLAlchemy provides for these two clauses using the :meth:`_sql.Select.group_by`
and :meth:`_sql.Select.having` methods.   Below we illustrate selecting
user name fields as well as count of addresses, for those users that have more
than one address:

.. sourcecode:: python+sql

    >>> with engine.connect() as conn:
    ...     result = conn.execute(
    ...         select(User.name, func.count(Address.id).label("count")).
    ...         join(Address).
    ...         group_by(User.name).
    ...         having(func.count(Address.id) > 1)
    ...     )
    ...     print(result.all())
    {opensql}BEGIN (implicit)
    SELECT user_account.name, count(address.id) AS count
    FROM user_account JOIN address ON user_account.id = address.user_id GROUP BY user_account.name
    HAVING count(address.id) > ?
    [...] (1,){stop}
    [('sandy', 2)]
    {opensql}ROLLBACK{stop}

Ordering or Grouping by a Label
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

An important technique in particular on some database backends is the ability
to ORDER BY or GROUP BY an expression that is already stated in the columns
clause, without re-stating the expression in the ORDER BY or GROUP BY clause
and instead using the column name or labeled name from the COLUMNS clause.
This form is available by passing the string text of the name to the
:meth:`_sql.Select.order_by` or :meth:`_sql.Select.group_by` method.  The text
passed is **not rendered directly**; instead, the name given to an expression
in the columns clause and rendered as that expression name in context, raising an
error if no match is found.   The unary modifiers
:func:`.asc` and :func:`.desc` may also be used in this form:

.. sourcecode:: pycon+sql

    >>> from sqlalchemy import func, desc
    >>> stmt = select(
    ...         Address.user_id,
    ...         func.count(Address.id).label('num_addresses')).\
    ...         group_by("user_id").order_by("user_id", desc("num_addresses"))
    >>> print(stmt)
    SELECT address.user_id, count(address.id) AS num_addresses
    FROM address GROUP BY address.user_id ORDER BY address.user_id, num_addresses DESC

.. _tutorial_using_aliases:

Using Aliases
^^^^^^^^^^^^^

Now that we are selecting from multiple tables and using joins, we quickly
run into the case where we need to refer to the same table mutiple times
in the FROM clause of a statement.  We accomplish this using SQL **aliases**,
which are a syntax that supplies an alternative name to a table or subquery
from which it can be referred towards in the statement.

In the SQLAlchemy Expression Language, these "names" are instead represented by
:class:`_sql.FromClause` objects known as the :class:`_sql.Alias` construct,
which is constructed in Core using the :meth:`_sql.FromClause.alias`
method. An :class:`_sql.Alias` construct is just like a :class:`_sql.Table`
construct in that it also has a namespace of :class:`_schema.Column`
objects within the :attr:`_sql.Alias.c` collection.  The SELECT statement
below for example returns all unique pairs of user names::

    >>> user_alias_1 = user_table.alias()
    >>> user_alias_2 = user_table.alias()
    >>> print(
    ...     select(user_alias_1.c.name, user_alias_2.c.name).
    ...     join_from(user_alias_1, user_alias_2, user_alias_1.c.id > user_alias_2.c.id)
    ... )
    SELECT user_account_1.name, user_account_2.name
    FROM user_account AS user_account_1
    JOIN user_account AS user_account_2 ON user_account_1.id > user_account_2.id

.. _tutorial_orm_entity_aliases:

ORM Entity Aliases
~~~~~~~~~~~~~~~~~~

The ORM equivalent of the :meth:`_sql.FromClause.alias` method is the
ORM :func:`_orm.aliased` function, which may be applied to an entity
such as ``User`` and ``Address``.  This produces a :class:`_sql.Alias` object
internally that's against the original mapped :class:`_schema.Table` object,
while maintaining ORM functionality.  The SELECT below selects from the
``User`` entity all objects that include two particular email addresses::

    >>> from sqlalchemy.orm import aliased
    >>> address_alias_1 = aliased(Address)
    >>> address_alias_2 = aliased(Address)
    >>> print(
    ...     select(User).
    ...     join_from(User, address_alias_1).
    ...     where(address_alias_1.email_address == 'patrick@aol.com').
    ...     join_from(User, address_alias_2).
    ...     where(address_alias_2.email_address == 'patrick@gmail.com')
    ... )
    SELECT user_account.id, user_account.name, user_account.fullname
    FROM user_account
    JOIN address AS address_1 ON user_account.id = address_1.user_id
    JOIN address AS address_2 ON user_account.id = address_2.user_id
    WHERE address_1.email_address = :email_address_1
    AND address_2.email_address = :email_address_2

To make use of a :func:`_orm.relationship` to construct a join **to** an
aliased entity, use the :meth:`_orm.PropComparator.of_type` modifier::

    >>> print(
    ...        select(User).
    ...        join(User.addresses.of_type(address_alias_1)).
    ...        where(address_alias_1.email_address == 'patrick@aol.com').
    ...        join(User.addresses.of_type(address_alias_2)).
    ...        where(address_alias_2.email_address == 'patrick@gmail.com')
    ...    )
    SELECT user_account.id, user_account.name, user_account.fullname
    FROM user_account
    JOIN address AS address_1 ON user_account.id = address_1.user_id
    JOIN address AS address_2 ON user_account.id = address_2.user_id
    WHERE address_1.email_address = :email_address_1
    AND address_2.email_address = :email_address_2

To make use of a :func:`_orm.relationship` to construct a join **from** an
aliased entity, the attribute is available from the :func:`_orm.aliased`
construct directly::

    >>> user_alias_1 = aliased(User)
    >>> print(
    ...     select(user_alias_1.name).
    ...     join(user_alias_1.addresses)
    ... )
    SELECT user_account_1.name
    FROM user_account AS user_account_1
    JOIN address ON user_account_1.id = address.user_id

.. _tutorial_subqueries_ctes:

Subqueries and CTEs
^^^^^^^^^^^^^^^^^^^^

A subquery in SQL is a SELECT statement that is rendered within parenthesis and
placed within the context of an enclosing statement, typically a SELECT
statement but not necessarily.    This section will cover a so-called
"non-scalar" subquery, which is a subquery typically placed as a member of the
FROM clause of an enclosing SELECT, providing a source of rows in the same way
as if a table were placed there.   In modern SQL, there is also a concept known
as a Common Table Expression (CTE) which is used in a similar way as the
non-scalar subquery, though it has a fairly different syntax and the ability
to be used in a "recursive" style for querying against hierarchical data.  Some
databases such as PostgreSQL support CTEs against :term:`DML` expressions as well.

Within SQLAlchemy, the non-scalar subquery and the CTE are represented
using the :class:`_sql.Subquery` and the
:class:`_sql.CTE` classes, which starting from a :class:`_sql.Select`
construct are composed using the :meth:`_sql.Select.subquery` and
:meth:`_sql.Select.cte` methods.  These objects are intended to be used in
the FROM clause of a SELECT statement or a similar context within other
kinds of statement.

Below illustrates a :class:`_sql.Subquery` used as the target of another call
to :func:`_sql.select`, which SELECTs multiple columns from the
``user_account`` table while at the same time selecting an aggregate count of
rows from the ``address`` table (aggregate functions and GROUP BY were
introduced previously at :ref:`tutorial_group_by_w_aggregates`)::

    >>> subq = select(
    ...     func.count(address_table.c.id).label("count"),
    ...     address_table.c.user_id
    ... ).group_by(address_table.c.user_id).subquery()
    >>> stmt = select(
    ...    user_table.c.name,
    ...    user_table.c.fullname,
    ...    subq.c.count
    ... ).join_from(user_table, subq)
    >>> print(stmt)
    SELECT user_account.name, user_account.fullname, anon_1.count
    FROM user_account JOIN (SELECT count(address.id) AS count, address.user_id AS user_id
    FROM address GROUP BY address.user_id) AS anon_1 ON user_account.id = anon_1.user_id

Above, note that the :meth:`_sql.Select.join_from` method was able infer the ON
clause of the join from ``user_table`` to our custom subquery, as the subquery
is **derived** from the ``address_table.c.user_id`` column that has a foreign
key back to ``user_table``.

Usage of the :class:`_sql.CTE` construct in SQLAlchemy is virtually
the same as how the :class:`_sql.Subquery` construct is used.  By changing
the invocation of the :meth:`_sql.Select.subquery` method to use
:meth:`_sql.Select.cte` instead, we can use the resulting object as a FROM
element in the same way, but the SQL rendered is the very different common
table expression syntax::

    >>> subq = select(
    ...     func.count(address_table.c.id).label("count"),
    ...     address_table.c.user_id
    ... ).group_by(address_table.c.user_id).cte()
    >>> stmt = select(
    ...    user_table.c.name,
    ...    user_table.c.fullname,
    ...    subq.c.count
    ... ).join_from(user_table, subq)
    >>> print(stmt)
    WITH anon_1 AS
    (SELECT count(address.id) AS count, address.user_id AS user_id
    FROM address GROUP BY address.user_id)
     SELECT user_account.name, user_account.fullname, anon_1.count
    FROM user_account JOIN anon_1 ON user_account.id = anon_1.user_id

The :class:`_sql.CTE` construct also features the ability to be used
in a "recursive" style, and may in more elaborate cases be composed from the
RETURNING clause of an INSERT, UPDATE or DELETE statement.  The docstring
for :class:`_sql.CTE` includes details on these additional patterns.

.. seealso::

    :meth:`_sql.Select.subquery` - further detail on subqueries

    :meth:`_sql.Select.cte` - examples for CTE including how to use
    RECURSIVE as well as DML-oriented CTEs

ORM Entity Subqueries/CTEs
~~~~~~~~~~~~~~~~~~~~~~~~~~

In the ORM, the :func:`_orm.aliased` construct may be used to associate an ORM
entity, such as our ``User`` or ``Address`` class, with any :class:`_sql.FromClause`
concept that represents a source of rows.  The preceding section
:ref:`tutorial_orm_entity_aliases` illustrates using :func:`_orm.aliased`
to associate the mapped class with an :class:`_sql.Alias` of its
mapped :class:`_schema.Table`.   Here we illustrate :func:`_orm.aliased` doing the same
thing against both a :class:`_sql.Subquery` as well as a :class:`_sql.CTE`
generated against a :class:`_sql.Select` construct, that ultimately derives
from that same mapped :class:`_schema.Table`.

Below is an example of applying :func:`_orm.aliased` to the :class:`_sql.Subquery`
construct, so that ORM entities can be extracted from its rows.  The result
shows a series of ``User`` and ``Address`` objects, where the data for
each ``Address`` object ultimately came from a subquery against the
``address`` table rather than that table directly:

.. sourcecode:: python+sql

    >>> subq = select(Address).where(~Address.email_address.like('%@aol.com')).subquery()
    >>> address_subq = aliased(Address, subq)
    >>> stmt = select(User, address_subq).join_from(User, address_subq).order_by(User.id, address_subq.id)
    >>> with Session(engine) as session:
    ...     for user, address in session.execute(stmt):
    ...         print(f"{user} {address}")
    {opensql}BEGIN (implicit)
    SELECT user_account.id, user_account.name, user_account.fullname,
    anon_1.id AS id_1, anon_1.email_address, anon_1.user_id
    FROM user_account JOIN
    (SELECT address.id AS id, address.email_address AS email_address, address.user_id AS user_id
    FROM address
    WHERE address.email_address NOT LIKE ?) AS anon_1 ON user_account.id = anon_1.user_id
    ORDER BY user_account.id, anon_1.id
    [...] ('%@aol.com',){stop}
    User(id=1, name='spongebob', fullname='Spongebob Squarepants') Address(id=1, email_address='spongebob@sqlalchemy.org')
    User(id=2, name='sandy', fullname='Sandy Cheeks') Address(id=2, email_address='sandy@sqlalchemy.org')
    User(id=2, name='sandy', fullname='Sandy Cheeks') Address(id=3, email_address='sandy@squirrelpower.org')
    {opensql}ROLLBACK{stop}

Another example follows, which is exactly the same except it makes use of the
:class:`_sql.CTE` construct instead:

.. sourcecode:: python+sql

    >>> cte = select(Address).where(~Address.email_address.like('%@aol.com')).cte()
    >>> address_cte = aliased(Address, cte)
    >>> stmt = select(User, address_cte).join_from(User, address_cte).order_by(User.id, address_cte.id)
    >>> with Session(engine) as session:
    ...     for user, address in session.execute(stmt):
    ...         print(f"{user} {address}")
    {opensql}BEGIN (implicit)
    WITH anon_1 AS
    (SELECT address.id AS id, address.email_address AS email_address, address.user_id AS user_id
    FROM address
    WHERE address.email_address NOT LIKE ?)
    SELECT user_account.id, user_account.name, user_account.fullname,
    anon_1.id AS id_1, anon_1.email_address, anon_1.user_id
    FROM user_account
    JOIN anon_1 ON user_account.id = anon_1.user_id
    ORDER BY user_account.id, anon_1.id
    [...] ('%@aol.com',){stop}
    User(id=1, name='spongebob', fullname='Spongebob Squarepants') Address(id=1, email_address='spongebob@sqlalchemy.org')
    User(id=2, name='sandy', fullname='Sandy Cheeks') Address(id=2, email_address='sandy@sqlalchemy.org')
    User(id=2, name='sandy', fullname='Sandy Cheeks') Address(id=3, email_address='sandy@squirrelpower.org')
    {opensql}ROLLBACK{stop}

In both cases, the subquery and CTE were named at the SQL level using an
"anonymous" name.  In the Python code, we don't need to provide these names
at all.  The object identity of the :class:`_sql.Subquery` or :class:`_sql.CTE`
instances serves as the syntactical identity of the object when rendered.

.. _tutorial_scalar_subquery:

Scalar and Correlated Subqueries
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

A scalar subquery is a subquery that returns exactly zero or one row and
exactly one column.  The subquery is then used in the COLUMNS or WHERE clause
of an enclosing SELECT statement and is different than a regular subquery in
that it is not used in the FROM clause.   A :term:`correlated subquery` is a
scalar subquery that refers to a table in the enclosing SELECT statement.

SQLAlchemy represents the scalar subquery using the
:class:`_sql.ScalarSelect` construct, which is part of the
:class:`_sql.ColumnElement` expression hierarchy, in contrast to the regular
subquery which is represented by the :class:`_sql.Subquery` construct, which is
in the :class:`_sql.FromClause` hierarchy.

Scalar subqueries are often, but not necessarily, used with aggregate functions,
introduced previously at :ref:`tutorial_group_by_w_aggregates`.   A scalar
subquery is indicated explicitly by making use of the :meth:`_sql.Select.scalar_subquery`
method as below.  It's default string form when stringified by itself
renders as an ordinary SELECT statement that is selecting from two tables::

    >>> subq = select(func.count(address_table.c.id)).\
    ...             where(user_table.c.id == address_table.c.user_id).\
    ...             scalar_subquery()
    >>> print(subq)
    (SELECT count(address.id) AS count_1
    FROM address, user_account
    WHERE user_account.id = address.user_id)

The above ``subq`` object now falls within the :class:`_sql.ColumnElement`
SQL expression hierarchy, in that it may be used like any other column
expression::

    >>> print(subq == 5)
    (SELECT count(address.id) AS count_1
    FROM address, user_account
    WHERE user_account.id = address.user_id) = :param_1


Although the scalar subquery by itself renders both ``user_account`` and
``address`` in its FROM clause when stringified by itself, when embedding it
into an enclosing :func:`_sql.select` construct that deals with the
``user_account`` table, the ``user_account`` table is automatically
**correlated**, meaning it does not render in the FROM clause of the subquery::

    >>> stmt = select(user_table.c.name, subq.label("address_count"))
    >>> print(stmt)
    SELECT user_account.name, (SELECT count(address.id) AS count_1
    FROM address
    WHERE user_account.id = address.user_id) AS address_count
    FROM user_account

Simple correlated subqueries will usually do the right thing that's desired.
However, in the case where the correlation is ambiguous, SQLAlchemy will let
us know that more clarity is needed::

    >>> stmt = select(
    ...     user_table.c.name,
    ...     address_table.c.email_address,
    ...     subq.label("address_count")
    ... ).\
    ... join_from(user_table, address_table).\
    ... order_by(user_table.c.id, address_table.c.id)
    >>> print(stmt)
    Traceback (most recent call last):
    ...
    InvalidRequestError: Select statement '<... Select object at ...>' returned
    no FROM clauses due to auto-correlation; specify correlate(<tables>) to
    control correlation manually.

To specify that the ``user_table`` is the one we seek to correlate we specify
this using the :meth:`_sql.ScalarSelect.correlate` or
:meth:`_sql.ScalarSelect.correlate_except` methods::

    >>> subq = select(func.count(address_table.c.id)).\
    ...             where(user_table.c.id == address_table.c.user_id).\
    ...             scalar_subquery().correlate(user_table)

The statement then can return the data for this column like any other:

.. sourcecode:: pycon+sql

    >>> with engine.connect() as conn:
    ...     result = conn.execute(
    ...         select(
    ...             user_table.c.name,
    ...             address_table.c.email_address,
    ...             subq.label("address_count")
    ...         ).
    ...         join_from(user_table, address_table).
    ...         order_by(user_table.c.id, address_table.c.id)
    ...     )
    ...     print(result.all())
    {opensql}BEGIN (implicit)
    SELECT user_account.name, address.email_address, (SELECT count(address.id) AS count_1
    FROM address
    WHERE user_account.id = address.user_id) AS address_count
    FROM user_account JOIN address ON user_account.id = address.user_id ORDER BY user_account.id, address.id
    [...] (){stop}
    [('spongebob', 'spongebob@sqlalchemy.org', 1), ('sandy', 'sandy@sqlalchemy.org', 2),
     ('sandy', 'sandy@squirrelpower.org', 2), ('patrick', 'pat999@aol.com', 1)]
    {opensql}ROLLBACK{stop}


.. rst-class:: core-header, orm-addin

Core UPDATE and DELETE
----------------------

So far we've covered :class:`_sql.Insert`, so that we can get some data into
our database, and then spent a lot of time on :class:`_sql.Select` which
handles the broad range of usage patterns used for retrieving data from the
database.   In this section we will cover the :class:`_sql.Update` and
:class:`_sql.Delete` constructs, which are used to modify existing rows
as well as delete existing rows.    This section will cover these constructs
from a Core-centric perspective.


.. container:: orm-header

    **ORM Readers** - As was the case mentioned at :ref:`tutorial_core_insert`,
    the :class:`_sql.Update` and :class:`_sql.Delete` operations when used with
    the ORM are usually invoked internally from the :class:`_orm.Session`
    object as part of the :term:`unit of work` process.

    However, unlike :class:`_sql.Insert`, the ORM also supports direct use of
    the :class:`_sql.Update` and :class:`_sql.Delete` constructs with
    :meth:`_orm.Session.execute` known as "bulk update" and "bulk delete". For
    this reason, Core UPDATE and DELETE are more relevant for ORM users, and
    background on their ORM-specific usage pattern can be seen at
    :ref:`bulk_update_delete`.

The update() SQL Expression Construct
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

include seealso to ORM bulk update()

The delete() SQL Expression Construct
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

include seealso to ORM bulk delete()


.. rst-class:: orm-header

.. _tutorial_orm_data_manipulation:

Data Manipulation with the ORM
==============================

The previous section :ref:`tutorial_working_with_data` remained focused on
the SQL Expression Language from a Core perspective, in order to provide
continuity across the major SQL statement constructs.  This section will
then build out the lifecycle of the :class:`_orm.Session` and how it interacts
with these constructs.

**Prerequisite Sections** - the ORM focused part of the tutorial builds upon
two previous ORM-centric sections in this document:

* :ref:`tutorial_executing_orm_session` - introduces how to make an ORM :class:`_orm.Session` object

* :ref:`tutorial_orm_table_metadata` - where we set up our ORM mappings of the ``User`` and ``Address`` entities

* :ref:`tutorial_selecting_orm_entities` - a few examples on how to run SELECT statements for entities like ``User``

.. _tutorial_inserting_orm:

Inserting Rows with the ORM
---------------------------

When using the ORM, the :class:`_orm.Session` object is responsible for
constructing :class:`_sql.Insert` constructs and emitting them for us in a
transaction. The way we instruct the :class:`_orm.Session` to do so is by
**adding** object entries to it; the :class:`_orm.Session` then makes sure
these new entries will be emitted to the database when they are needed, using
a process known as a **flush**.

Instances of Classes represent Rows
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Whereas in the previous example we emitted an INSERT using Python dictionaries
to indicate the data we wanted to add, with the ORM we make direct use of the
custom Python classes we defined, back at
:ref:`tutorial_orm_table_metadata`.    At the class level, the ``User`` and
``Address`` classes served as a place to define what the corresponding
database tables should look like.   These classes also serve as extensible
data objects that we use to create and manipulate rows within a transaction
as well.  Below we will create two ``User`` objects each representing a new
database row::

    >>> squidward = User(name="squidward", fullname="Squidward Tentacles")
    >>> krabs = User(name="ehkrabs", fullname="Eugene H. Krabs")

The ``User`` class includes an automatically generated ``__init__()``
constructor that was provided by the ORM mapping so that we could create
each object using column names as keys in the constructor.   In a similar
manner as in our Core examples of :class:`_sql.Insert`, we did not include a
primary key (i.e. an entry for the ``id`` column), since we would like to make
use of SQLite's auto-incrementing primary key feature which the ORM also
integrates with.

Adding objects to a Session
^^^^^^^^^^^^^^^^^^^^^^^^^^^^

To illustrate the addition process step by step, we will create a
:class:`_orm.Session` without using a context manager (and hence we must
make sure we close it later!)::

    >>> session = Session(engine)

The objects are then added to the :class:`_orm.Session` using the
:meth:`_orm.Session.add` method.   When this is called, the objects are in a
state known as :term:`pending` and have not been inserted yet::

    >>> session.add(squidward)
    >>> session.add(krabs)

When we have pending objects, we can see this state by looking at a
collection on the :class:`_orm.Session` called :attr:`_orm.Session.new`::

    >>> session.new
    IdentitySet([User(id=None, name='squidward', fullname='Squidward Tentacles'), User(id=None, name='ehkrabs', fullname='Eugene H. Krabs')])

The above view is using a collection called :class:`.IdentitySet` that is
essentially a Python set that hashes on object identity in all cases (i.e.,
using Python built-in ``id()`` function, rather than the Python ``hash()`` function).

Flushing
^^^^^^^^

The :class:`_orm.Session` makes use of a pattern known as :term:`unit of work`.
This generally means it accumulates changes one at a time, but does not actually
communicate them to the database until needed.   This allows it to make
better decisions about how SQL DML should be emitted in the transaction based
on a given set of pending changes.   When it does emit SQL to the database
to push out the current set of changes, the process is known as a **flush**.

We can illustrate the flush process manually by calling the :meth:`_orm.Session.flush`
method:

.. sourcecode:: pycon+sql

    >>> session.flush()
    {opensql}BEGIN (implicit)
    INSERT INTO user_account (name, fullname) VALUES (?, ?)
    [...] ('squidward', 'Squidward Tentacles')
    INSERT INTO user_account (name, fullname) VALUES (?, ?)
    [...] ('ehkrabs', 'Eugene H. Krabs')

Above we observe the :class:`_orm.Session` was first called upon to emit
SQL, so it created a new transaction and emitted the appropriate INSERT
statements for the two objects.   The transaction now **remains open**
until we call the :meth:`_orm.Session.commit` method.

While :meth:`_orm.Session.flush` may be used to manually push out pending
changes to the current transaction, it is usually unnecessary as the
:class:`_orm.Session` features a behavior known as **autoflush**, which
we will illustrate later.   It also flushes out changes whenever
:meth:`_orm.Session.commit` is called.


Autogenerated primary key attributes
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Once the rows are inserted, the two Python objects we've created are in a
state known as :term:`persistent`, where they are associated with the
:class:`_orm.Session` object in which they were added or loaded, and feature lots of
other behaviors that will be covered later.

Another effect of the INSERT that occurred was that the ORM has retrieved the
new primary key identifiers for each new object; internally it normally uses
the same :attr:`_engine.CursorResult.inserted_primary_key` accessor we
introduced previously.   The ``squidward`` and ``krabs`` objects now have these new
primary key identifiers associated with them and we can view them by acesssing
the ``id`` attribute::

    >>> squidward.id
    4
    >>> krabs.id
    5

.. tip::  Why did the ORM emit two separate INSERT statements when it could have
   used :ref:`executemany <tutorial_multiple_parameters>`?  As we'll see in the
   next section, the
   :class:`_orm.Session` when flushing objects always needs to know the
   primary key of newly inserted objects.  If a feature such as SQLite's autoincrement is used
   (other examples include PostgreSQL IDENTITY or SERIAL, using sequences,
   etc.), the :attr:`_engine.CursorResult.inserted_primary_key` feature
   usually requires that each INSERT is emitted one row at a time.  If we had provided values for the primary keys ahead of
   time, the ORM would have been able to optimize the operation better.  Some
   database backends such as :ref:`psycopg2 <postgresql_psycopg2>` can also
   INSERT many rows at once while still being able to retrieve the primary key
   values.

Identity Map
^^^^^^^^^^^^

The primary key identity of the objects are significant to the :class:`_orm.Session`,
as the objects are now linked to this identity in memory using a feature
known as the :term:`identity map`.   The identity map is an in-memory store
that links all objects currently loaded in memory to their primary key
identity.   We can observe this by retrieving one of the above objects
using the :meth:`_orm.Session.get` method, which will return an entry
from the identity map if locally present, otherwise emitting a SELECT::

    >>> some_squidward = session.get(User, 4)
    >>> some_squidward
    User(id=4, name='squidward', fullname='Squidward Tentacles')

The important thing to note about the identity map is that it maintains a
**unique instance** of a particular Python object per a particular database
identity, within the scope of a particular :class:`_orm.Session` object.  We
may observe that the ``some_squidward`` refers to the **same object** as that
of ``squidward`` previously::

    >>> some_squidward is squidward
    True

The identity map is a critical feature that allows complex sets of objects
to be manipulated within a transaction without things getting out of sync.


Committing
^^^^^^^^^^^

There's much more to say about how the :class:`_orm.Session` works which will
be discussed further.   For now we will commit the transaction so that
we can build up knowledge on how to SELECT rows before examining more ORM
behaviors and features:

.. sourcecode:: pycon+sql

    >>> session.commit()
    COMMIT

Closing
^^^^^^^^

Additionally, in the absence of using a context manager for this :class:`_orm.Session`,
we may explicitly close it using the :meth:`_orm.Session.close` method.  This method,
which is also called automatically when we use the context manager form of
:class:`_orm.Session`, :term:`releases` any remaining connection resources and **expunges** all objects from the :class:`_orm.Session`::

    >>> session.close()

After the above call to :meth:`_orm.Session.close` (as well as when a context
manager ends) our ``squidward`` and ``krabs`` objects in a state known as
:term:`detached`.   While detached objects still retain their normal Python
object functionality, they no longer proxy any database row, nor do they have
any reference to any ongoing database state.  They can be re-attached to a
:class:`_orm.Session` at a later point or simply discarded (in fact they are
safe to discard while in the persistent state as well).

Updating ORM Objects
--------------------

There are two ways of directly emitting an UPDATE for an ORM object; one
is that UPDATE statements are emitted on a per-primary key basis, and the
other is a single UPDATE statement that affects many rows.  The usual UPDATE
performed is the former style, as it occurs automatically when we modify
an object.

Supposing we loaded the ``User`` object for the username ``sandy`` into
a transaction (also showing off the :meth:`_sql.Select.filter_by` method
as well as the :meth:`_engine.Result.scalar_one` method):

.. sourcecode:: pycon+sql

    {sql}>>> sandy = session.execute(select(User).filter_by(name="sandy")).scalar_one()
    BEGIN (implicit)
    SELECT user_account.id, user_account.name, user_account.fullname
    FROM user_account
    WHERE user_account.name = ?
    [...] ('sandy',)

The Python object ``sandy`` as mentioned before acts as a **proxy** for the
row in the database, more specifically the database row **in terms of the
current transaction**, that has the primary key identity of ``2``::

    >>> sandy
    User(id=2, name='sandy', fullname='Sandy Cheeks')

If we alter the attributes of this object, the :class:`_orm.Session` tracks
this change::

    >>> sandy.fullname = "Sandy Squirrel"

The object appears in a collection called :attr:`_orm.Session.dirty`, indicating
the object is "dirty"::

    >>> sandy in session.dirty
    True

When the :class:`_orm.Session` next emits a flush, an UPDATE will be emitted
that updates this value in the database.  As mentioned previously, a flush
occurs automatically before we emit any SELECT, using a behavior known as
**autoflush**.  We can query directly for the ``User.fullname`` column
from this row and we will get our updated value back:

.. sourcecode:: pycon+sql

    >>> sandy_fullname = session.execute(
    ...     select(User.fullname).where(User.id == 2)
    ... ).scalar_one()
    {opensql}UPDATE user_account SET fullname=? WHERE user_account.id = ?
    [...] ('Sandy Squirrel', 2)
    SELECT user_account.fullname
    FROM user_account
    WHERE user_account.id = ?
    [...] (2,){stop}
    >>> print(sandy_fullname)
    Sandy Squirrel

We can see above that we requested that the :class:`_orm.Session` execute
a single :func:`_sql.select` statement.  However the SQL emitted shows
that an UPDATE were emitted as well, which was the flush process pushing
out pending changes.  The ``sandy`` Python object is now no longer considered
dirty::

    >>> sandy in session.dirty
    False

However note we are **still in a transaction** and our changes have not
been pushed to the database's permanent storage.   Since Sandy's last name
is in fact "Cheeks" not "Squirrel", we will repair this mistake in the
next section.


.. seealso::

    :ref:`session_flushing`- details the flush process as well as information
    about the :paramref:`_orm.Session.autoflush` setting.


Rolling Back
-------------

The :class:`_orm.Session` has a :meth:`_orm.Session.rollback` method that as
expected emits a ROLLBACK on the SQL connection in progress.  However, it also
has an effect on the objects that are currently associated with the
:class:`_orm.Session`, in our previous example the Python object ``sandy``.
While we changed the ``.fullname`` of the ``sandy`` object to read ``"Sandy
Squirrel"``, we want to roll back this change.   Calling
:meth:`_orm.Session.rollback` will not only roll back the transaction but also
**expire** all objects currently associated with this :class:`_orm.Session`,
which will have the effect that they will refresh themselves when next accessed
using a process known as :term:`lazy loading`:

.. sourcecode:: pycon+sql

    >>> session.rollback()
    ROLLBACK

To view the "expiration" process more closely, we may observe that the
Python object ``sandy`` has no state left within its Python ``__dict__``,
with the exception of a special SQLAlchemy internal state object::

    >>> sandy.__dict__
    {'_sa_instance_state': <sqlalchemy.orm.state.InstanceState object at 0x...>}

This is the "expired" state; accessing the attribute again will autobegin
a new transaction and refresh ``sandy`` with the current database row:

.. sourcecode:: pycon+sql

    >>> sandy.fullname
    {opensql}BEGIN (implicit)
    SELECT user_account.id AS user_account_id, user_account.name AS user_account_name,
    user_account.fullname AS user_account_fullname
    FROM user_account
    WHERE user_account.id = ?
    [...] (2,){stop}
    'Sandy Cheeks'

We may now observe that the full database row was also populated into the
``__dict__`` of the ``sandy`` object::

    >>> sandy.__dict__  # doctest: +SKIP
    {'_sa_instance_state': <sqlalchemy.orm.state.InstanceState object at 0x...>,
     'id': 2, 'name': 'sandy', 'fullname': 'Sandy Cheeks'}


Deleting ORM Objects
---------------------

ORM Selection Patterns
-----------------------

Working with Related Objects
----------------------------

Configuring delete/delete-orphan cascade
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Querying with Joins
--------------------

Using Aliases
^^^^^^^^^^^^^^

Selecting Entities from Subqueries
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Using EXISTS
^^^^^^^^^^^^

Common Relationship Operators
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Loader Strategies
-----------------

Selectin Load
^^^^^^^^^^^^^^

Joined Load
^^^^^^^^^^^

Explicit Join + Eager load
^^^^^^^^^^^^^^^^^^^^^^^^^^^


ORM-Specific Options
--------------------



ORM UPDATE and DELETE patterns
------------------------------










additional content not integrated yet
=====================================






Update
------

* :func:`_sql.update` - this function generates a new instance of
  :class:`_sql.Update` which represents an UPDATE statement in SQL, that
  will update existing data in a table.

  Like the :func:`_sql.insert` construct, there is a "traditional" form
  of :func:`_sql.update`, which emits UPDATE against a single table at a time
  and does not return any rows.   However some backends support an UPDATE
  statement that may modify multiple tables at once, and the UPDATE
  statement also supports RETURNING such that rows would be returned.

  The :func:`_sql.update` function is commonly used with both Core and ORM
  oriented programming.   Within the ORM, like the :func:`_sql.insert`
  construct, most UPDATE statements are managed internally by the
  :term:`unit of work` process and is not directly exposed to the user.
  However, the ORM also has the ability to accommodate the :func:`_sql.update`
  construct directly in the same way it would be used in Core to support
  a so-called :ref:`bulk update <bulk_update_delete>` query.

Delete
------

* :func:`_sql.delete` - this function generates a new instance of
  :class:`_sql.Delete` which represents an DELETE statement in SQL, that
  will delete rows from a table.

  The :func:`_sql.delete` statement from an API perpsective is very similar
  to that of the :func:`_sql.update` construct, traditionally returning
  no rows but allowing for a RETURNING variant, and additionally is
  more commonly used with Core rather than the ORM, but is also available
  in explicit form with the ORM for :ref:`bulk delete <bulk_update_delete>`
  queries.