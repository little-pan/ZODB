.. % Introduction
.. % What is ZODB?
.. % What is ZEO?
.. % OODBs vs. Relational DBs
.. % Other OODBs


Introduction
============

This guide explains how to write Python programs that use the Z Object Database
(ZODB) and Zope Enterprise Objects (ZEO).  The latest version of the guide is
always available at `<http://www.zope.org/Wikis/ZODB/guide/index.html>`_.


What is the ZODB?
-----------------

The ZODB is a persistence system for Python objects.  Persistent programming
languages provide facilities that automatically write objects to disk and read
them in again when they're required by a running program.  By installing the
ZODB, you add such facilities to Python.

It's certainly possible to build your own system for making Python objects
persistent.  The usual starting points are the :mod:`pickle` module, for
converting objects into a string representation, and various database modules,
such as the :mod:`gdbm` or :mod:`bsddb` modules, that provide ways to write
strings to disk and read them back.  It's straightforward to combine the
:mod:`pickle` module and a database module to store and retrieve objects, and in
fact the :mod:`shelve` module, included in Python's standard library, does this.

The downside is that the programmer has to explicitly manage objects, reading an
object when it's needed and writing it out to disk when the object is no longer
required.  The ZODB manages objects for you, keeping them in a cache, writing
them out to disk when they are modified, and dropping them from the cache if
they haven't been used in a while.


OODBs vs. Relational DBs
------------------------

Another way to look at it is that the ZODB is a Python-specific object-oriented
database (OODB).  Commercial object databases for C++ or Java often require that
you jump through some hoops, such as using a special preprocessor or avoiding
certain data types.  As we'll see, the ZODB has some hoops of its own to jump
through, but in comparison the naturalness of the ZODB is astonishing.

Relational databases (RDBs) are far more common than OODBs. Relational databases
store information in tables; a table consists of any number of rows, each row
containing several columns of information.  (Rows are more formally called
relations, which is where the term "relational database" originates.)

Let's look at a concrete example.  The example comes from my day job working for
the MEMS Exchange, in a greatly simplified version.  The job is to track process
runs, which are lists of manufacturing steps to be performed in a semiconductor
fab.  A run is owned by a particular user, and has a name and assigned ID
number.  Runs consist of a number of operations; an operation is a single step
to be performed, such as depositing something on a wafer or etching something
off it.

Operations may have parameters, which are additional information required to
perform an operation.  For example, if you're depositing something on a wafer,
you need to know two things: 1) what you're depositing, and 2) how much should
be deposited.  You might deposit 100 microns of silicon oxide, or 1 micron of
copper.

Mapping these structures to a relational database is straightforward::

   CREATE TABLE runs (
     int      run_id,
     varchar  owner,
     varchar  title,
     int      acct_num,
     primary key(run_id)
   );

   CREATE TABLE operations (
     int      run_id,
     int      step_num, 
     varchar  process_id,
     PRIMARY KEY(run_id, step_num),
     FOREIGN KEY(run_id) REFERENCES runs(run_id),
   );

   CREATE TABLE parameters (
     int      run_id,
     int      step_num, 
     varchar  param_name, 
     varchar  param_value,
     PRIMARY KEY(run_id, step_num, param_name)
     FOREIGN KEY(run_id, step_num) 
        REFERENCES operations(run_id, step_num),
   );  

In Python, you would write three classes named :class:`Run`, :class:`Operation`,
and :class:`Parameter`.  I won't present code for defining these classes, since
that code is uninteresting at this point. Each class would contain a single
method to begin with, an :meth:`__init__` method that assigns default values,
such as 0 or ``None``, to each attribute of the class.

It's not difficult to write Python code that will create a :class:`Run` instance
and populate it with the data from the relational tables; with a little more
effort, you can build a straightforward tool, usually called an object-
relational mapper, to do this automatically. (See
`<http://www.amk.ca/python/unmaintained/ordb.html>`_ for a quick hack at a
Python object-relational mapper, and
`<http://www.python.org/workshops/1997-10/proceedings/shprentz.html>`_ for Joel
Shprentz's more successful implementation of the same idea; Unlike mine,
Shprentz's system has been used for actual work.)

However, it is difficult to make an object-relational mapper reasonably quick; a
simple-minded implementation like mine is quite slow because it has to do
several queries to access all of an object's data.  Higher performance object-
relational mappers cache objects to improve performance, only performing SQL
queries when they actually need to.

That helps if you want to access run number 123 all of a sudden.  But what if
you want to find all runs where a step has a parameter named 'thickness' with a
value of 2.0?  In the relational version, you have two unappealing choices:

#. Write a specialized SQL query for this case: ``SELECT run_id FROM operations
   WHERE param_name = 'thickness' AND param_value = 2.0``

   If such queries are common, you can end up with lots of specialized queries.
   When the database tables get rearranged, all these queries will need to be
   modified.

#. An object-relational mapper doesn't help much.  Scanning through the runs
   means that the the mapper will perform the required SQL queries to read run #1,
   and then a simple Python loop can check whether any of its steps have the
   parameter you're looking for. Repeat for run #2, 3, and so forth.  This does a
   vast number of SQL queries, and therefore is incredibly slow.

An object database such as ZODB simply stores internal pointers from object to
object, so reading in a single object is much faster than doing a bunch of SQL
queries and assembling the results. Scanning all runs, therefore, is still
inefficient, but not grossly inefficient.


What is ZEO?
------------

The ZODB comes with a few different classes that implement the :class:`Storage`
interface.  Such classes handle the job of writing out Python objects to a
physical storage medium, which can be a disk file (the :class:`FileStorage`
class), a BerkeleyDB file (:class:`BDBFullStorage`), a relational database
(:class:`DCOracleStorage`), or some other medium.  ZEO adds
:class:`ClientStorage`, a new :class:`Storage` that doesn't write to physical
media but just forwards all requests across a network to a server.  The server,
which is running an instance of the :class:`StorageServer` class, simply acts as
a front-end for some physical :class:`Storage` class.  It's a fairly simple
idea, but as we'll see later on in this document, it opens up many
possibilities.


About this guide
----------------

The primary author of this guide works on a project which uses the ZODB and ZEO
as its primary storage technology.  We use the ZODB to store process runs and
operations, a catalog of available processes, user information, accounting
information, and other data.  Part of the goal of writing this document is to
make our experience more widely available.  A few times we've spent hours or
even days trying to figure out a problem, and this guide is an attempt to gather
up the knowledge we've gained so that others don't have to make the same
mistakes we did while learning.

The author's ZODB project is described in a paper available here,
`<http://www.amk.ca/python/writing/mx-architecture/>`_

This document will always be a work in progress.  If you wish to suggest
clarifications or additional topics, please send your comments to the
`ZODB-dev mailing list <https://groups.google.com/forum/#!forum/zodb>`_.


Acknowledgements
----------------

Andrew Kuchling wrote the original version of this guide, which provided some of
the first ZODB documentation for Python programmers. His initial version has
been updated over time by Jeremy Hylton and Tim Peters.

I'd like to thank the people who've pointed out inaccuracies and bugs, offered
suggestions on the text, or proposed new topics that should be covered: Jeff
Bauer, Willem Broekema, Thomas Guettler, Chris McDonough, George Runyan.

