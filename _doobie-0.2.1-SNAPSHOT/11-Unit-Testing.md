---
layout: book
number: 11
title: Unit Testing
---

<div class="alert alert-warning" role="alert">
<b>Warning:</b> The functionality described in this chapter is experimental and is likely to change in future versions.
</div>

The YOLO-mode query checking feature demonstated in an earlier chapter is also available as a trait you can mix into your [Specs2](http://etorreborre.github.io/specs2/) unit tests.

### Setting Up

As with earlier chapters we set up a `Transactor` and YOLO mode. Note that the code in this chapter also requires the `doobie-contrib-specs2` add-on.

```scala
import doobie.imports._, scalaz._, Scalaz._, scalaz.concurrent.Task

val xa = DriverManagerTransactor[Task](
  "org.postgresql.Driver", "jdbc:postgresql:world", "postgres", ""
)
```

And again we are playing with the `country` table, given here for reference.

```sql
CREATE TABLE country (
  code        character(3)  NOT NULL,
  name        text          NOT NULL,
  population  integer       NOT NULL,
  gnp         numeric(10,2),
  indepyear   smallint
  -- more columns, but we won't use them here
)
```

### The Specs Package

The `doobie-contrib-specs2` add-on provides a mix-in trait that we can add to a `Specification` to allow for typechecking of queries, interpreted as a set of specifications.

So here are a few queries we would like to check. Note that we can only check values of type `Query0` and `Update0`; we can't check `Process` or `ConnectionIO` values, so a good practice is to define your queries in a DAO module and apply further operations at a higher level. 

```scala
case class Country(code: Int, name: String, pop: Int, gnp: Double)

val trivial = sql"""
  select 42, 'foo'::varchar
""".query[(Int, String)]

def biggerThan(minPop: Short) = sql"""
  select code, name, population, gnp, indepyear
  from country
  where population > $minPop
""".query[Country]

def update(oldName: String, newName: String) = sql"""
  update country set name = $newName where name = $oldName
""".update
```

Our unit test needs to extend `AnalysisSpec` and must define a `Transactor[Task]`. To construct a testcase for a query, pass it to the `check` method. Note that query arguments are never used, so they can be any values that typecheck.

```scala
import doobie.contrib.specs2.analysisspec.AnalysisSpec

import org.specs2.mutable.Specification

object AnalysisTestSpec extends Specification with AnalysisSpec {
  val transactor = DriverManagerTransactor[Task](
    "org.postgresql.Driver", "jdbc:postgresql:world", "postgres", ""
  )
  check(trivial)
  check(biggerThan(0))
  check(update("", ""))
}
```

When we run the test we get output similar to what we saw in the previous chapter on checking queries, but each item is now a test. Note that doing this in the REPL is a little awkward; in real source you would get the source file and line number associated with each query.

```
scala> { specs2 run AnalysisTestSpec; () } // pretend this is sbt> test
$line12.$read$$iw$$iw$$iw$$iw$$iw$$iw$$iw$$iw$AnalysisTestSpec$

Query0[(Int, String)] defined at <console>:18
  
  select 42, 'foo'::varchar
x SQL Compiles and Typechecks
   x The parameter index is out of range: 1, number of parameters: 0. (specs2.scala:60)


Query0[Country] defined at <console>:20
  
  select code, name, population, gnp, indepyear
  from country
  where population > ?
+ SQL Compiles and Typechecks
x P01 Short  →  INTEGER (int4)
   x Short is not coercible to INTEGER (int4) according to the JDBC specification.
     Fix this by changing the schema type to SMALLINT, or the Scala type to Int or
     JdbcType. (<console>:20)

x C01 code       CHAR     (bpchar)  NOT NULL  →  Int
   x CHAR (bpchar) is ostensibly coercible to Int according to the JDBC specification
     but is not a recommended target type. Fix this by changing the schema type to
     INTEGER; or the Scala type to PersonId or String. (<console>:20)

+ C02 name       VARCHAR  (varchar) NOT NULL  →  String
+ C03 population INTEGER  (int4)    NOT NULL  →  Int
x C04 gnp        NUMERIC  (numeric) NULL      →  Double
   x NUMERIC (numeric) is ostensibly coercible to Double according to the JDBC
     specification but is not a recommended target type. Fix this by changing the
     schema type to FLOAT or DOUBLE; or the Scala type to BigDecimal or BigDecimal.
   x Reading a NULL value into Double will result in a runtime failure. Fix this by
     making the schema type NOT NULL or by changing the Scala type to Option[Double] (<console>:20)

x C05 indepyear  SMALLINT (int2)    NULL      →  
   x Column is unused. Remove it from the SELECT statement. (<console>:20)


Update0 defined at <console>:18
  
  update country set name = ? where name = ?
+ SQL Compiles and Typechecks
+ P01 String  →  VARCHAR (varchar)
+ P02 String  →  VARCHAR (text)

Total for specification $line12.$read$$iw$$iw$$iw$$iw$$iw$$iw$$iw$$iw$AnalysisTestSpec$
Finished in 28 ms
11 examples, 5 failures, 0 error
```



