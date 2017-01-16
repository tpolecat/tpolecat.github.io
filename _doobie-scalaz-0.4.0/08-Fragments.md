---
layout: book
number: 8
title: Statement Fragments
---

In this chapter we discuss how to construct SQL statements at runtime.

### Setting Up

Same as last chapter, so if you're still set up you can skip this section. Otherwise let's set up a `Transactor` and YOLO mode.

```scala
import doobie.imports._
import scalaz._, Scalaz._
val xa = DriverManagerTransactor[IOLite](
  "org.postgresql.Driver", "jdbc:postgresql:world", "postgres", ""
)
import xa.yolo._
```

We're still playing with the `country` table, shown here for reference.

```sql
CREATE TABLE country (
  code       character(3)  NOT NULL,
  name       text          NOT NULL,
  population integer       NOT NULL,
  gnp        numeric(10,2)
  -- more columns, but we won't use them here
)
```

### Composing SQL literals

SQL literals constructed with the `fr` interpolator behave just like the `sql` interpolator and can be composed by concatenation.

```scala
scala> val a = fr"select name from country"
a: doobie.util.fragment.Fragment = Fragment("select name from country ")

scala> val b = fr"where code = 'USA'"
b: doobie.util.fragment.Fragment = Fragment("where code = 'USA' ")

scala> val c = a ++ b // concatenation by ++
c: doobie.util.fragment.Fragment = Fragment("select name from country where code = 'USA' ")

scala> c.query[String].unique.quick.unsafePerformIO
  United States
```

Fragments can capture arguments of any type with a `Param` instance, just as the `sql` interpolator does.

```scala
scala> def whereCode(s: String) = fr"where code = $s"
whereCode: (s: String)doobie.util.fragment.Fragment

scala> val fra = whereCode("FRA")
fra: doobie.util.fragment.Fragment = Fragment("where code = ? ")

scala> (fr"select name from country" ++ fra).query[String].quick.unsafePerformIO
  France
```

You can lift an arbitrary string value via `Fragment.const`, which allows you to parameterize on things that aren't valid SQL parameters.

```scala
scala> def count(table: String) = (fr"select count(*) from" ++ Fragment.const(table)).query[Int].unique
count: (table: String)doobie.free.connection.ConnectionIO[Int]

scala> count("city").quick.unsafePerformIO
  4079
```


### Whitespace handling

The rendered SQL string for a `fr` fragment will have a single space character appended, which is usually what you want. Normally you don't need to worry about whitespace when composing fragments.

If you do *not* want a fragment to have trailing space you can use the `fr0` interpolator. This is used here and there in the `Fragments` module to yield prettier SQL strings.

```scala
scala> fr"IN (" ++ List(1, 2, 3).map(n => fr"$n").intercalate(fr",") ++ fr")"
res3: doobie.util.fragment.Fragment = Fragment("IN ( ? , ? , ? ) ")

scala> fr0"IN (" ++ List(1, 2, 3).map(n => fr0"$n").intercalate(fr",") ++ fr")"
res4: doobie.util.fragment.Fragment = Fragment("IN (?, ?, ?) ")
```
Note that the `sql` interpolator is simply an alias for `fr0`.

### The `Fragments` Module

The `Fragments` module provides some combinators for common patterns when working with fragments. The following example illustrates a few of them. See the Scaladoc or source for more information.

Here we define a query with a three optional filter conditions.

```scala
// Import some convenience combinators.
import Fragments.{ in, whereAndOpt }

// Country Info
case class Info(name: String, code: String, population: Int)

// Construct a Query0 with some optional filter conditions and a configurable LIMIT.
def select(name: Option[String], pop: Option[Int], codes: List[String], limit: Long) = {

  // Three Option[Fragment] filter conditions.
  val f1 = name.map(s => fr"name LIKE $s")
  val f2 = pop.map(n => fr"population > $n")
  val f3 = codes.toNel.map(cs => in(fr"code", cs))

  // Our final query
  val q: Fragment =
    fr"SELECT name, code, population FROM country" ++
    whereAndOpt(f1, f2, f3)                         ++
    fr"LIMIT $limit"

  // Consruct a Query0
  q.query[Info]

}
```

We first construct three optional filters, the third of which uses the `in` combinator to construct an SQL `IN` clause. The final statement uses the `whereAndOpt` combinator that constructs a `WHERE` clause with the passed sequence of `Option[Fragment]` joined with `AND` if any are defined, otherwise it evaluates to the empty fragment. The end result is that the `WHERE` clause appears only if at least one filter is defined.

Let's look at a few possibilities.

```scala
scala> select(None, None, Nil, 10).check.unsafePerformIO // no filters

  SELECT name, code, population FROM country LIMIT ? 

  ✓ SQL Compiles and Typechecks
  ✓ P01 Long  →  BIGINT (int8)
  ✓ C01 name       VARCHAR (varchar) NOT NULL  →  String
  ✓ C02 code       CHAR    (bpchar)  NOT NULL  →  String
  ✓ C03 population INTEGER (int4)    NOT NULL  →  Int

scala> select(Some("U%"), None, Nil, 10).check.unsafePerformIO // one filter

  SELECT name, code, population FROM country WHERE name LIKE ? LIMIT ? 

  ✓ SQL Compiles and Typechecks
  ✓ P01 String  →  VARCHAR (text)
  ✓ P02 Long    →  BIGINT  (int8)
  ✓ C01 name       VARCHAR (varchar) NOT NULL  →  String
  ✓ C02 code       CHAR    (bpchar)  NOT NULL  →  String
  ✓ C03 population INTEGER (int4)    NOT NULL  →  Int

scala> select(Some("U%"), Some(12345), List("FRA", "GBR"), 10).check.unsafePerformIO // three filters

  SELECT name, code, population FROM country WHERE name LIKE ? AND population > ? AND code IN (?, ?) LIMIT ? 

  ✓ SQL Compiles and Typechecks
  ✓ P01 String  →  VARCHAR (text)
  ✓ P02 Int     →  INTEGER (int4)
  ✓ P03 String  →  CHAR    (bpchar)
  ✓ P04 String  →  CHAR    (bpchar)
  ✓ P05 Long    →  BIGINT  (int8)
  ✓ C01 name       VARCHAR (varchar) NOT NULL  →  String
  ✓ C02 code       CHAR    (bpchar)  NOT NULL  →  String
  ✓ C03 population INTEGER (int4)    NOT NULL  →  Int
```
