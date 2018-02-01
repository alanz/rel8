.. highlight:: haskell

Getting Started
===============

In this article, we'll take a tour through the basics of Rel8. We'll learn how
to define base tables, write simple queries, and execute these queries against a
real database.


Required language extensions and imports
----------------------------------------

::

  {-# LANGUAGE Arrows, DataKinds, DeriveGeneric, FlexibleInstances,
               MultiParamTypeClasses, OverloadedStrings #-}

  import Control.Applicative
  import Control.Arrow
  import Rel8

To use Rel8, you will need a few language extensions:

* ``Arrows`` is necessary to use ``proc`` notation - like ``do`` notation
  for monads. As with Opaleye, Rel8 uses arrows to guarantee queries are valid.

* ``DataKinds`` is used to promote values to the type level when defining
  table/column metadata.

* ``DeriveGeneric`` is used to derive functions from schema
  information.

The other extensions can be considered as "necessary evil" to provide the type
system extensions needed by Rel8.


Defining base tables
--------------------

To query a database of existing tables, we need to let Rel8 know
about these tables, and the schema for each table. This is done by defining a
Haskell *record* for each table in the database. These records should have a
type of the form ``C f name hasDefault t``. Let's see how that looks with some
example tables::

  data Part f = Part
    { partId     :: C f "PID" 'HasDefault Integer
    , partName   :: C f "PName" 'NoDefault String
    , partColor  :: C f "Color" 'NoDefault Integer
    , partWeight :: C f "Weight" 'NoDefault Double
    , partCity   :: C f "City" 'NoDefault String
    } deriving (Generic)

  instance BaseTable Part where tableName = "part"
  instance Table (Part Expr) (Part QueryResult)

The ``Part`` table has 5 columns, each defined with the ``C f ..`` pattern. For
each column, we are specifying:

1. The column name.
2. Whether this column has a default value when inserting new rows. In
   this case ``partId`` does, as this is an auto-incremented primary key managed
   by the database.
3. The type of the column.

After defining the table, we finally need to make instances of ``BaseTable`` and
``Table`` so Rel8 can query this table. By using ``deriving (Generic)``, we
need to write ``instance BaseTable Part where tableName = "part"``. The
``Table`` instance demonstrates that a ``Part Expr`` value can be selected from
the database as ``Part QueryResult``.


Querying tables
---------------

With tables defined, we are now ready to write some queries. All ``BaseTable`` s
give rise to a query - the query of all rows in that table::

  allParts :: O.Query (Part Expr)
  allParts = queryTable

Notice the type of ``allParts`` specifies that we're working with ``Part Expr``.
This means that the contents of the ``Part`` record will contain expressions -
one for each column in the table. As ``O.Query`` is a ``Functor``, we can derive
a new query for all part cities in the database::

  allPartCities :: O.Query (Expr String)
  allPartCities = partCity <$> allParts

Now we have a query containing one column - expressions of type ``String``.

``WHERE`` clauses
-----------------

Usually when we are querying database, we are querying for subsets of
information. In SQL, we apply predicates using ``WHERE`` - and Rel8 supports
this too, in two forms.

We use ``filterQuery`` as we would use ``filter``::

  londonParts :: O.Query (Part Expr)
  londonParts = filterQuery (\p -> partCity p ==. "London") allParts

``filterQuery`` takes a function from rows in a query to a predicate. In this
case we can use ``==.`` to compare to expressions for equality. On the left,
``partCity p :: Expr String``, and on the right ``"London" :: Expr String``
(the literal string ``London``).
Alternatively, we can use ``where_`` with arrow notation, which is like
using ``guard`` with ``MonadPlus``::

  heavyParts :: O.Query (Part Expr)
  heavyParts = proc _ -> do
    part <- queryTable -< ()
    where_ -< partWeight part >. 5
    returnA -< part

Joining Queries
---------------

Rel8 supports joining multiple queries into one, in a few different ways.

Products and Inner Joins
^^^^^^^^^^^^^^^^^^^^^^^^

We can take the product of two queries - each row of the first query paired with
each row of the second query - by sequencing queries inside a ``O.Query``. Let's
introduce another table::

  data Supplier f = Supplier
    { supplierId :: C f "SID" 'HasDefault Int
    , supplierName :: C f "SName" 'NoDefault String
    , supplierStatus :: C f "Status" 'NoDefault Int
    , supplierCity :: C f "City" 'NoDefault String
    } deriving (Generic)

  instance BaseTable Supplier where tableName = "supplier"
  instance Table (Supplier Expr) (Supplier QueryResult)

We can take the product of all parts paired against all suppliers by simplying
selecting from both tables and returning a tuple::

  allPartsAndSuppliers :: O.Query (Part Expr, Supplier Expr)
  allPartsAndSuppliers = proc _ -> do
    part <- queryTable -< ()
    supplier <- queryTable -< ()
    returnA -< (part, supplier)

We could write this a little more succinctly using using the ``Applicative``
instance for ``O.Query``, as ``<*>`` corresponds to products::

  allPartsAndSuppliers2 :: O.Query (Part Expr, Supplier Expr)
  allPartsAndSuppliers2 = liftA2 (,) queryTable queryTable

In both queries, we've used ``queryTable`` to select the necessary rows.
``queryTable`` is overloaded, but by knowing the type of rows to select, it will
change which table it queries from.

We can combine products with the techniques we've seen to produce
the inner join of two tables. For example, here is a query to pair up each part
with all suppliers in the same city::

  partsAndSuppliers :: Query (Part Expr, Supplier Expr)
  partsAndSuppliers =
    filterQuery
      (\(part, supplier) -> partCity part ==. supplierCity supplier)
      allPartsAndSuppliers

Left Joins
^^^^^^^^^^

The previous query gave us parts with *at least one* supplier in the same city.
If a part has no suppliers in the same city, it will be omitted from the
results. But what if we needed this information? In SQL we can capture this with
a ``LEFT JOIN``, and Rel8 supports this.

Left joins can be introduced with the ``leftJoin``, which takes two queries, or
using arrow notation with ``leftJoinA``. Let's look at the latter, as it is
often more concise::

  partsAndSuppliersLJ :: Query (Part Expr, MaybeTable (Supplier Expr))
  partsAndSuppliersLJ = proc _ -> do
    part <- queryTable -< ()
    maybeSupplier
      <- leftJoinA queryTable
      -< \supplier -> partCity part ==. supplierCity supplier
    returnA -< (part, maybeSupplier)

This is a little different to anything we've seen so far, so let's break it
down. ``leftJoinA`` takes as its first argument the query to join in. In this
case we use ``queryTable`` to select all supplier rows. ``LEFT JOIN`` s also
require a predicate, and we supply this as *input* to ``leftJoinA``. The input
is itself a function, a function from rows in the to-be-joined table to
booleans. Notice that in this predicate, we are free to refer to tables and
columns already in the query (as ``partCity part`` is not part of the supplier
table).

Left joins themselves can be filtered, as they are just another query. However,
the results of a left join are wrapped in ``MaybeTable``, which indicates that
*all* of the columns in this table might be ``null``, if the join failed to
match any rows. We can use this information with our ``partsAndSuppliersLJ``
query to find parts where there are no suppliers in the same city::

  partsWithoutSuppliersInCity :: Query (Part Expr)
  partsWithoutSuppliersInCity = proc _ -> do
    (part, maybeSupplier) <- partsAndSuppliersLJ -< ()
    where_ -< isNull (maybeSupplier $? supplierId)
    returnA -< part

.. note::

  This type of query is what is known as an *antijoin*. A more efficient way to
  write the above is by using the ``notExists`` function. For more information,
  see :ref:`antijoins` in :doc:`concepts`.

We are filtering our query for suppliers where the id is null. Ordinarily this
would be a type error - we declared that ``supplierId`` contains ``Int``, rather
than ``Maybe Int``. However, because suppliers come from a left join, when we
project out from ``MaybeTable`` *all* columns become nullable. It may help to
think of ``($?)`` as having the type:::

  ($?) :: (a -> Expr b) -> MaybeTable a -> Expr (Maybe b)

though in Rel8 we're a little bit more general.


Aggregation
-----------

To aggregate a series of rows, use the ``aggregate`` query transform.
``aggregate`` takes a ``Query`` that returns any ``AggregateTable`` as a result.
``AggregateTable`` s are like ``Tables``, except that all expressions describe a
way to aggregate data. While tuples are instances of ``AggregateTable``, it's
recommended to introduce new data types to represent aggregations for clarity.

As an example of aggregation, let's start with a table modelling all users in
our application::

  data User f = User
    { userId :: Col f "id" 'HasDefault Int64
    , userLastLoggedIn :: Col f "last_logged_in_at" 'NoDefault UTCTime
    , userType :: Col f "user_type" 'NoDefault Text
    } deriving (Generic)

  instance Table (User Expr) (User QueryResult)
  instance BaseTable User where tableName = "users"

We would like to aggregate over this table, grouped by user type, learning how
many users we have and the latest login time in that group. First, let's
introduce a record to be able to refer to this data::

  data UserInfo f = UserInfo
    { userCount :: Anon f Int64
    , latestLogin :: Anon f UTCTime
    , uType :: Anon f Text
    } deriving (Generic)

  instance AggregateTable (UserInfo Aggregate) (UserInfo Expr)
  instange Table (UserInfo Expr) (UserInfo QueryResult)

This record is defined in a similar pattern to tables we've seen before,
but this time we're using the ``Anon`` decorator, rather than ``C``. ``Anon``
can be used for tables that aren't base tables, and means we don't have to
provide metadata about the column name and whether it has a default
value. In this case, ``UserInfo`` doesn't model a base table, it models a
derived table.

Also, notice that we derived a new type class instance that we haven't seen yet.
``UserInfo`` will be used with ``Aggregate`` expressions, and the
``AggregateTable`` instance states we can aggregate the ``UserInfo`` data type.

With this, aggregation can be written as a concise query::

  userInfo :: Query (UserInfo Expr)
  userInfo = aggregate $ proc _ -> do
    user <- queryTable -< ()
    returnA -< UserInfo { userCount = count (userId user)
                        , latestLogin = max (userLastLoggedIn user)
                        , uType = groupBy (userType user)
                        }

Running Queries
---------------

So far we've written various queries, but we haven't actually seen how to
perform any IO with them. Rel8 gives you entry points into the main ways of
interacting with a relational database - ``DELETE``, ``INSERT``, ``SELECT`` and
``UPDATE``. ``SELECT`` is arguably the most common type of query, so we'll begin
with that.

You can run any query that returns results using the ``select`` function from
``Rel8.IO``. ``select`` needs to be given a ``QueryRunner``, which is a type of
function for actually performing the IO. There are two default query runners,
``stream`` and ``streamCursor``. It's beyond the scope of this tutorial to
discuss the difference, curious users are encouraged to check the API
documentation. ``stream`` is often enough, so let's look at a program that
queries the ``part`` table from earlier

Select
^^^^^^

::

  import Database.PostgreSQL.Simple
  import Control.Monad.Trans.Resource (runResourceT)
  import qualified Streaming.Prelude as Stream

  selectAllParts :: IO [Part QueryResult]
  selectAllParts = do
    databaseConnection <- connect defaultConnectInfo
    runResourceT . Stream.toList_ $
      select (stream databaseConnection) allParts

We use ``select`` with a ``stream`` ``QueryRunner`` built from our
``databaseConnection``. This returns a ``Stream`` of results - in this case we
immediately flatten that stream into a concrete list with ``toList_``. Finally,
we need to deal with resource handling on that query, which can be done with
``runResourceT``.


Data Modification
^^^^^^^^^^^^^^^^^

Data modification queries are queries that use ``DELETE``, ``INSERT`` or
``UPDATE``, and Rel8 gives two interfaces to these queries - one that
runs the query, and another than runs the query and returns a ``Stream`` of
results (the ``Returning`` family of functions).

For ``update``, we specify a database connection, a predicate to select rows to
update, and a function that transforms each row. The following will change the
colour of part 5 to red::

  update databaseConnection
         (\part -> partId part ==. lit 5)
         (\part -> part { partColor = lit "red" })

For ``insert``, we have some extra syntax for fields that can contain default
values. Note that we marked ``partId`` as having a default value::

  partId :: C f "PID" 'HasDefault Int

This means the database can provide a default value for this column when we
insert rows (usually automatically incrementing a sequence)::

  insert databaseConnection
         [Part { partId     = InsertDefault
               , partName   = lit "New part"
               , partColor  = lit "Gold"
               , partWeight = lit 3.14
               , partCity   = lit "London"
               }]

Using ``insertReturning`` you can immediately witness what these default values
are.

Finally, there is ``delete`` which requires only a predicate to choose which
rows should be deleted::

  delete databaseConnection (\p -> partId p >=. 10)
