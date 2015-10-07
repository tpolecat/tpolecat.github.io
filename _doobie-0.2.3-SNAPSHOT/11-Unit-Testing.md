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
import doobie.imports._
import scalaz._, Scalaz._
import scalaz.concurrent.Task

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
[info] $line14.$read$$iw$$iw$$iw$$iw$$iw$$iw$$iw$$iw$AnalysisTestSpec$
[info] 
[info] Query0[(Int, String)] defined at <console>:22
  
  select 42, 'foo'::varchar
[info]   + SQL Compiles and Typechecks
[info]   + C01 ?column? INTEGER (int4)    NULL?  →  Int
[info]   + C02 varchar  VARCHAR (varchar) NULL?  →  String
[info] Query0[Country] defined at <console>:24
  
  select code, name, population, gnp, indepyear
  from country
  where population > ?
[info]   + SQL Compiles and Typechecks
[error]   x P01 Short  →  INTEGER (int4)
[error]    x Short is not coercible to INTEGER (int4) according to the JDBC specification.
     Fix this by changing the schema type to SMALLINT, or the Scala type to Int or
     JdbcType. (<console>:24)
[info] 
[error]   x C01 code       CHAR     (bpchar)  NOT NULL  →  Int
[error]    x CHAR (bpchar) is ostensibly coercible to Int according to the JDBC specification
     but is not a recommended target type. Fix this by changing the schema type to
     INTEGER; or the Scala type to PersonId or String. (<console>:24)
[info] 
[info]   + C02 name       VARCHAR  (varchar) NOT NULL  →  String
[info]   + C03 population INTEGER  (int4)    NOT NULL  →  Int
[error]   x C04 gnp        NUMERIC  (numeric) NULL      →  Double
[error]    x NUMERIC (numeric) is ostensibly coercible to Double according to the JDBC
     specification but is not a recommended target type. Fix this by changing the
     schema type to FLOAT or DOUBLE; or the Scala type to BigDecimal or BigDecimal.
   x Reading a NULL value into Double will result in a runtime failure. Fix this by
     making the schema type NOT NULL or by changing the Scala type to Option[Double] (<console>:24)
[info] 
[error]   x C05 indepyear  SMALLINT (int2)    NULL      →  
[error]    x Column is unused. Remove it from the SELECT statement. (<console>:24)
[info] 
[info] Update0 defined at <console>:22
  
  update country set name = ? where name = ?
[info]   + SQL Compiles and Typechecks
[info]   + P01 String  →  VARCHAR (varchar)
[info]   + P02 String  →  VARCHAR (text)
[info] 
[info] Total for specification $line14.$read$$iw$$iw$$iw$$iw$$iw$$iw$$iw$$iw$AnalysisTestSpec$
[info] Finished in 29 ms
13 examples, 4 failures, 0 error
[info] 
```



