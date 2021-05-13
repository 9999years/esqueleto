Esqueleto [![CI](https://github.com/bitemyapp/esqueleto/actions/workflows/haskell.yml/badge.svg?branch=master)](https://github.com/bitemyapp/esqueleto/actions/workflows/haskell.yml)
==========

![Skeleton](./esqueleto.png)
<sup>Image courtesy [Chrissy Long](https://www.flickr.com/photos/chrissylong/313800029/)</sup>

# Esqueleto, a SQL DSL for Haskell

Esqueleto is a bare bones, type-safe EDSL for SQL queries that works with unmodified persistent SQL backends. The name of this library means "skeleton" in Portuguese and contains all three SQL letters in the correct order =). It was inspired by Scala's Squeryl but created from scratch. Its language closely resembles SQL. Currently, SELECTs, UPDATEs, INSERTs and DELETEs are supported.

In particular, esqueleto is the recommended library for type-safe JOINs on persistent SQL backends. (The alternative is using raw SQL, but that's error prone and does not offer any composability.). For more information read [esqueleto](http://hackage.haskell.org/package/esqueleto).

## Setup

If you're already using `persistent`, then you're ready to use `esqueleto`, no further setup is needed.  If you're just starting a new project and would like to use `esqueleto`, take a look at `persistent`'s [book](http://www.yesodweb.com/book/persistent) first to learn how to define your schema.

If you need to use `persistent`'s default support for queries as well, either import it qualified:

```haskell
-- For a module that mostly uses esqueleto.
import Database.Esqueleto
import qualified Database.Persistent as P
```

or import `esqueleto` itself qualified:

```haskell
-- For a module that uses esqueleto just on some queries.
import Database.Persistent
import qualified Database.Esqueleto as E
```

Other than identifier name clashes, `esqueleto` does not conflict with `persistent` in any way.


## Goals

The main goals of `esqueleto` are:

- Be easily translatable to SQL. (You should be able to know exactly how the SQL query will end up.)
- Support the most widely used SQL features.
- Be as type-safe as possible.

It is _not_ a goal to be able to write portable SQL. We do not try to hide the differences between DBMSs from you


## Introduction

For the following examples, we'll use this example schema:

```haskell
share [mkPersist sqlSettings, mkMigrate "migrateAll"] [persist|
  Person
    name String
    age Int Maybe
    deriving Eq Show
  BlogPost
    title String
    authorId PersonId
    deriving Eq Show
  Follow
    follower PersonId
    followed PersonId
    deriving Eq Show
|]
```

## Select

Most of `esqueleto` was created with `SELECT` statements in mind, not only because they're the most common but also because they're the most complex kind of statement.  The most simple kind of `SELECT` would be:

```haskell
putPersons :: SqlPersist m ()
putPersons = do
  people <- select $
              from $ \person -> do
              return person
  liftIO $ mapM_ (putStrLn . personName . entityVal) people
```

which generates this SQL:

```sql
SELECT *
FROM Person
```

`esqueleto` knows that we want an `Entity Person` just because of the `personName` that is printed.

## Where

Filtering by `PersonName`:

```haskell
select $
from $ \p -> do
where_ (p ^. PersonName ==. val "John")
return p
```

which generates this SQL:

```sql
SELECT *
FROM Person
WHERE Person.name = "John"
```

The `(^.)` operator is used to project a field from an entity. The field name is the same one generated by `persistent`s Template Haskell functions.  We use `val` to lift a constant Haskell value into the SQL query.

Another example:

In `esqueleto`, we may write the same query above as:

```haskell
select $
from $ \p -> do
where_ (p ^. PersonAge >=. just (val 18))
return p
```

which generates this SQL:

```sql
SELECT *
FROM Person
WHERE Person.age >= 18
```

Since `age` is an optional `Person` field, we use `just` to lift`val 18 :: SqlExpr (Value Int)` into `just (val 18) ::SqlExpr (Value (Maybe Int))`.

## Experimental/New Joins

There's a new way to write `JOIN`s in esqueleto! It has less potential for
runtime errors and is much more powerful than the old syntax. To opt in to the
new syntax, import:

```haskell
import Database.Esqueleto.Experimental
```

This will conflict with the definition of `from` and `on` in the
`Database.Esqueleto` module, so you'll want to remove that import.

This style will become the new "default" in esqueleto-4.0.0.0, so it's a good
idea to port your code to using it soon.

The module documentation in `Database.Esqueleto.Experimental` has many examples,
and they won't be repeated here. Here's a quick sample:

```haskell
select $ do
  (a :& b) <-
    from $
      Table @BlogPost
      `InnerJoin`
      Table @Person
        `on` do \(bp :& a) ->
          bp ^. BlogPostAuthorId ==. a ^. PersonId
  pure (a, b)
```

Advantages:

- `ON` clause is attached directly to the relevant join, so you never need to
  worry about how they're ordered, nor will you ever run into bugs where the
  `on` clause is on the wrong `JOIN`
- The `ON` clause lambda will all the available tables in it. This forbids
  runtime errors where an `ON` clause refers to a table that isn't in scope yet.
- You can join on a table twice, and the aliases work out fine with the `ON`
  clause.
- You can use `UNION`, `EXCEPT`, `INTERSECTION` etc  with this new syntax!
- You can reuse subqueries more easily.

## Legacy Joins

Implicit joins are represented by tuples.

For example, to get the list of all blog posts and their authors, we could write:

```haskell
select $
from $ \(b, p) -> do
where_ (b ^. BlogPostAuthorId ==. p ^. PersonId)
orderBy [asc (b ^. BlogPostTitle)]
return (b, p)
```

which generates this SQL:

```sql
SELECT BlogPost.*, Person.*
FROM BlogPost, Person
WHERE BlogPost.authorId = Person.id
ORDER BY BlogPost.title ASC
```


However, you may want your results to include people who don't have any blog posts as well using a `LEFT OUTER JOIN`:

```haskell
select $
from $ \(p `LeftOuterJoin` mb) -> do
on (just (p ^. PersonId) ==. mb ?. BlogPostAuthorId)
orderBy [asc (p ^. PersonName), asc (mb ?. BlogPostTitle)]
return (p, mb)
```

which generates this SQL:

```sql
SELECT Person.*, BlogPost.*
FROM Person LEFT OUTER JOIN BlogPost
ON Person.id = BlogPost.authorId
ORDER BY Person.name ASC, BlogPost.title ASC
```

## Left Outer Join

On a `LEFT OUTER JOIN` the entity on the right hand side may not exist (i.e. there may be a `Person` without any `BlogPost`s), so while `p :: SqlExpr (Entity Person)`, we have `mb :: SqlExpr (Maybe (Entity BlogPost))`.  The whole expression above has type `SqlPersist m [(Entity Person, Maybe (Entity BlogPost))]`.  Instead of using `(^.)`, we used `(?.)` to project a field from a `Maybe (Entity a)`.

We are by no means limited to joins of two tables, nor by joins of different tables.  For example, we may want a list of the `Follow` entity:

```haskell
select $
from $ \(p1 `InnerJoin` f `InnerJoin` p2) -> do
on (p2 ^. PersonId ==. f ^. FollowFollowed)
on (p1 ^. PersonId ==. f ^. FollowFollower)
return (p1, f, p2)
```

which generates this SQL:

```sql
SELECT P1.*, Follow.*, P2.*
FROM Person AS P1
INNER JOIN Follow ON P1.id = Follow.follower
INNER JOIN Person AS P2 ON P2.id = Follow.followed
```

## Update and Delete

```haskell
do update $ \p -> do
     set p [ PersonName =. val "João" ]
     where_ (p ^. PersonName ==. val "Joao")
   delete $
     from $ \p -> do
     where_ (p ^. PersonAge <. just (val 14))
```

The results of queries can also be used for insertions. In `SQL`, we might write the following, inserting a new blog post for every user:

```haskell
 insertSelect $ from $ \p->
 return $ BlogPost <# "Group Blog Post" <&> (p ^. PersonId)
```

which generates this SQL:

```sql
INSERT INTO BlogPost
SELECT ('Group Blog Post', id)
FROM Person
```

Individual insertions can be performed through Persistent's `insert` function, reexported for convenience.

### Re-exports

We re-export many symbols from `persistent` for convenience:
- "Store functions" from "Database.Persist".
- Everything from "Database.Persist.Class" except for `PersistQuery` and `delete` (use `deleteKey` instead).
- Everything from "Database.Persist.Types" except for `Update`, `SelectOpt`, `BackendSpecificFilter` and `Filter`.
- Everything from "Database.Persist.Sql" except for `deleteWhereCount` and `updateWhereCount`.

### RDBMS Specific

There are many differences between SQL syntax and functions supported by different RDBMSs.  Since version 2.2.8, `esqueleto` includes modules containing functions that are specific to a given RDBMS.

- PostgreSQL: `Database.Esqueleto.PostgreSQL`
- MySQL: `Database.Esqueleto.MySQL`
- SQLite: `Database.Esqueleto.SQLite`

In order to use these functions, you need to explicitly import their corresponding modules.

### Unsafe functions, operators and values

Esqueleto doesn't support every possible function, and it can't - many functions aren't available on every RDBMS platform, and sometimes the same functionality is hidden behind different names. To overcome this problem, Esqueleto exports a number of unsafe functions to call any function, operator or value. These functions can be found in Database.Esqueleto.Internal.Sql module.

Warning: the functions discussed in this section must always be used with an explicit type signature,and the user must be careful to provide a type signature that corresponds correctly with the underlying code. The functions have extremely general types, and if you allow type inference to figure everything out for you, it may not correspond with the underlying SQL types that you want. This interface is effectively the FFI to SQL database, so take care!

The most common use of these functions is for calling RDBMS specific or custom functions,
for that end we use `unsafeSqlFunction`. For example, if we wish to consult the postgres
`now` function we could so as follow:

```haskell
postgresTime :: (MonadIO m, MonadLogger m) => SqlWriteT m UTCTime
postgresTime =
  result <- select (pure now)
  case result of
    [x] -> pure x
    _ -> error "now() is guaranteed to return a single result"
  where
    now :: SqlExpr (Value UTCTime)
    now = unsafeSqlFunction "now" ()
```

which generates this SQL:

```sql
SELECT now()
```

With the `now` function we could now use the current time of the postgres RDBMS on any query.
Do notice that `now` does not use any arguments, so we use `()` that is an instance of
`UnsafeSqlFunctionArgument` to represent no arguments, an empty list cast to a correct value
will yield the same result as `()`.

We can also use `unsafeSqlFunction` for more complex functions with customs values using
`unsafeSqlValue` which turns any string into a sql value of whatever type we want, disclaimer:
if you use it badly you will cause a runtime error. For example, say we want to try postgres'
`date_part` function and get the day of a timestamp, we could use:

```haskell
postgresTimestampDay :: (MonadIO m, MonadLogger m) => SqlWriteT m Int
postgresTimestampDay =
  result <- select (return $ dayPart date)
  case result of
    [x] -> pure x
    _ -> error "dayPart is guaranteed to return a single result"
  where
    dayPart :: SqlExpr (Value UTCTime) -> SqlExpr (Value Int)
    dayPart s = unsafeSqlFunction "date_part" (unsafeSqlValue "\'day\'" :: SqlExpr (Value String) ,s)
    date :: SqlExpr (Value UTCTime)
    date = unsafeSqlValue "TIMESTAMP \'2001-02-16 20:38:40\'"
```

which generates this SQL:

```sql
SELECT date_part('day', TIMESTAMP '2001-02-16 20:38:40')
```

Using `unsafeSqlValue` we were required to also define the type of the value.

Another useful unsafe function is `unsafeSqlCastAs`, which allows us to cast any type
to another within a query. For example, say we want to use our previews `dayPart` function
on the current system time, we could:

```haskell
postgresTimestampDay :: (MonadIO m, MonadLogger m) => SqlWriteT m Int
postgresTimestampDay = do
  currentTime <- liftIO getCurrentTime
  result <- select (return $ dayPart (toTIMESTAMP $ val currentTime))
  case result of
    [x] -> pure x
    _ -> error "dayPart is guaranteed to return a single result"
  where
    dayPart :: SqlExpr (Value UTCTime) -> SqlExpr (Value Int)
    dayPart s = unsafeSqlFunction "date_part" (unsafeSqlValue "\'day\'" :: SqlExpr (Value String) ,s)
    toTIMESTAMP :: SqlExpr (Value UTCTime) -> SqlExpr (Value UTCTime)
    toTIMESTAMP = unsafeSqlCastAs "TIMESTAMP"
```

which generates this SQL:

```sql
SELECT date_part('day', CAST('2019-10-28 23:19:39.400898344Z' AS TIMESTAMP))
```

### SQL injection

Esqueleto uses parameterization to prevent sql injections on values and arguments
on all queries, for example, if we have:

```haskell
myEvilQuery :: (MonadIO m, MonadLogger m) => SqlWriteT m ()
myEvilQuery =
  select (return $ val ("hi\'; DROP TABLE foo; select \'bye\'" :: String)) >>= liftIO . print
```

which generates this SQL(when using postgres):

```sql
SELECT 'hi''; DROP TABLE foo; select ''bye'''
```

And the printed value is `hi\'; DROP TABLE foo; select \'bye\'` and no table is dropped. This is good
and makes the use of strings values safe. Unfortunately this is not the case when using unsafe functions.
Let's see an example of defining a new evil `now` function:

```haskell
myEvilQuery :: (MonadIO m, MonadLogger m) => SqlWriteT m ()
myEvilQuery =
  select (return nowWithInjection) >>= liftIO . print
  where
    nowWithInjection :: SqlExpr (Value UTCTime)
    nowWithInjection = unsafeSqlFunction "0; DROP TABLE bar; select now" ([] :: [SqlExpr (Value Int)])
```

which generates this SQL:

```sql
SELECT 0; DROP TABLE bar; select now()
```

If we were to run the above code we would see the postgres time printed but the table `bar`
will be erased with no indication whatsoever. Another example of this behavior is seen when using
`unsafeSqlValue`:

```haskell
myEvilQuery :: (MonadIO m, MonadLogger m) => SqlWriteT m ()
myEvilQuery =
  select (return $ dayPart dateWithInjection) >>= liftIO . print
  where
    dayPart :: SqlExpr (Value UTCTime) -> SqlExpr (Value Int)
    dayPart s = unsafeSqlFunction "date_part" (unsafeSqlValue "\'day\'" :: SqlExpr (Value String) ,s)
    dateWithInjection :: SqlExpr (Value UTCTime)
    dateWithInjection = unsafeSqlValue "TIMESTAMP \'2001-02-16 20:38:40\');DROP TABLE bar; select (16"
```

which generates this SQL:

```sql
SELECT date_part('day', TIMESTAMP '2001-02-16 20:38:40');DROP TABLE bar; select (16)
```

This will print 16 and also erase the `bar` table. The main take away of this examples is to
never use any user or third party input inside an unsafe function without first parsing it or
heavily sanitizing the input.

### Tests

To run the tests, do `stack test`. This tests all the backends, so you'll need
to have MySQL and Postgresql installed.

#### Postgres

Using apt-get, you should be able to do:

```
sudo apt-get install postgresql postgresql-contrib
sudo apt-get install libpq-dev
```

Using homebrew on OSx

```
brew install postgresql
brew install libpq
```

Detailed instructions on the Postgres wiki [here](https://wiki.postgresql.org/wiki/Detailed_installation_guides)

The connection details are located near the bottom of the [test/PostgreSQL/Test.hs](test/PostgreSQL/Test.hs) file:

```
withConn =
  R.runResourceT . withPostgresqlConn "host=localhost port=5432 user=esqutest password=esqutest dbname=esqutest"
```

You can change these if you like but to just get them working set up as follows on linux:

```
$ sudo -u postgres createuser esqutest
$ sudo -u postgres createdb esqutest
$ sudo -u postgres psql
postgres=# \password esqutest
```

And on osx

```
$ createuser esqutest
$ createdb esqutest
$ psql postgres
postgres=# \password esqutest
```

#### MySQL

To test MySQL, you'll need to have a MySQL server installation.
Then, you'll need to create a database `esqutest` and a `'travis'@'localhost'`
user which can access it:

```
mysql> CREATE DATABASE esqutest;
mysql> CREATE USER 'travis'@'localhost';
mysql> ALTER USER 'travis'@'localhost' IDENTIFIED BY 'esqutest';
mysql> GRANT ALL ON esqutest.* TO 'travis';
```
