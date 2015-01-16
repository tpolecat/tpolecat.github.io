---
layout: book
number: 6
title: Typechecking Queries
---

### Setting Up

Same as last chapter, so if you're still set up you can skip this section. 

```scala
import doobie.imports._

import scalaz._, Scalaz._, scalaz.concurrent.Task

val xa = DriverManagerTransactor[Task](
  "org.h2.Driver",                      // driver class
  "jdbc:h2:mem:ch6;DB_CLOSE_DELAY=-1",  // connect URL
  "sa", ""                              // user and pass
)
```

Sample data and YOLO mode.

```scala
scala> sql"RUNSCRIPT FROM 'world.sql' CHARSET 'UTF-8'".update.run.transact(xa).run
res0: Int = 5313

scala> import xa.yolo._
import xa.yolo._
```

And again, playing with the `country` table, here again for reference.

```sql
CREATE TABLE country (
  code        character(3)  NOT NULL,
  name        text          NOT NULL,
  population  integer NOT NULL,
  gnp         numeric(10,2),
  indepyear   smallint
  -- more columns, but we won't use them here
)
```

### Checking a Query

Let's redefine our `Country` class, just for fun.

```scala
case class Country(code: Int, name: String, pop: Int, gnp: Double)
```

Ok here's our parameterized query from last chapter, but with the new `Country` definition and `minPop` as a `Short`. Looks the same but means something different because the mapped types have changed.

```scala
def biggerThan(minPop: Short) = sql"""
  select code, name, population, gnp, indepyear
  from country
  where population > $minPop
""".query[Country]
```

So let's try the `check` method provided by YOLO and see what happens.

```
scala> biggerThan(0).check.run

    select code, name, population, gnp, indepyear
    from country
    where population > ?

  ✓ SQL Compiles and Typechecks
  ✕ P01 Short  →  INTEGER (INTEGER)
    - Short is not coercible to INTEGER (INTEGER) according to the JDBC specification.
      Fix this by changing the schema type to SMALLINT, or the Scala type to Int or
      JdbcType.
  ✕ C01 CODE       CHAR     (CHAR)     NOT NULL  →  Int
    - CHAR (CHAR) is ostensibly coercible to Int according to the JDBC specification
      but is not a recommended target type. Fix this by changing the schema type to
      INTEGER; or the Scala type to String.
  ✓ C02 NAME       VARCHAR  (VARCHAR)  NOT NULL  →  String
  ✓ C03 POPULATION INTEGER  (INTEGER)  NOT NULL  →  Int
  ✕ C04 GNP        DECIMAL  (DECIMAL)  NULL      →  Double
    - DECIMAL (DECIMAL) is ostensibly coercible to Double according to the JDBC
      specification but is not a recommended target type. Fix this by changing the
      schema type to FLOAT or DOUBLE; or the Scala type to BigDecimal or BigDecimal.
    - Reading a NULL value into Double will result in a runtime failure. Fix this by
      making the schema type NOT NULL or by changing the Scala type to Option[Double]
  ✕ C05 INDEPYEAR  SMALLINT (SMALLINT) NULL      →  
    - Column is unused. Remove it from the SELECT statement.
```

Yikes, there are quite a few problems, in several categories. In this case **doobie** found

- a parameter coercion that should always work but is not required to be supported by compliant drivers;
- two column coercions that **are** supported by JDBC but are not recommended and can fail in some cases;
- a column nullability mismatch, where a column is read into a non-`Option` type;
- and an unused column.

If we fix all of these problems and try again, we get a clean bill of health.

```scala
case class Country(code: String, name: String, pop: Int, gnp: Option[BigDecimal])

def biggerThan(minPop: Int) = sql"""
  select code, name, population, gnp
  from country
  where population > $minPop
""".query[Country]
```

```
scala> biggerThan(0).check.run

    select code, name, population, gnp
    from country
    where population > ?

  ✓ SQL Compiles and Typechecks
  ✓ P01 Int  →  INTEGER (INTEGER)
  ✓ C01 CODE       CHAR    (CHAR)    NOT NULL  →  String
  ✓ C02 NAME       VARCHAR (VARCHAR) NOT NULL  →  String
  ✓ C03 POPULATION INTEGER (INTEGER) NOT NULL  →  Int
  ✓ C04 GNP        DECIMAL (DECIMAL) NULL      →  Option[BigDecimal]
```

**doobie** supports `check` for queries and updates in three ways: programmatically, via YOLO mode in the REPL, and via the `contrib-specs2` package, which allows checking to become part of your unit test suite. We will investigate this in the chapter on testing.

### Diving Deeper

*You do not need to know this, but if you are curious about the implementation you may find it interesting.*

The `check` logic requires both a database connection and concrete `Meta` instances that define column-level JDBC mappings. This could in principle happen at compile-time, but it's not clear that this is what you always want and it's potentially hairy to implement. So for now checking happens at unit-test time.

The way this works is that a `Query` value has enough type information to describe all parameter and column mappings, as well as the SQL literal itself (with interpolated parameters erased into `?`). From here it is straightforward to prepare the statement, pull the `ResultsetMetaData` and `DatabaseMetaData` and work out whether things are aligned correctly (and if not, determine how misalignments might be fixed).








