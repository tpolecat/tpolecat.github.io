---
layout: book
number: 5
title: Parameterized Queries
---

In this short chapter we learn how to construct parameterized queries.

### Setting Up

Same as last chapter, so if you're still set up you can skip this section. Otherwise let's set up an in-memory database.

```scala
import doobie.imports._, scalaz._, Scalaz._, scalaz.concurrent.Task

val xa = DriverManagerTransactor[Task](
  "org.h2.Driver",                      // driver class
  "jdbc:h2:mem:ch5;DB_CLOSE_DELAY=-1",  // connect URL
  "sa", ""                              // user and pass
)
```

And then load up our sample data and pull in YOLO mode to save some typing.

```scala
scala> sql"RUNSCRIPT FROM 'world.sql' CHARSET 'UTF-8'".update.run.transact(xa).run
res0: Int = 5313

scala> import xa.yolo._
import xa.yolo._
```

This time we're still be playing with the `country` table, but just with a few columns, shown here.

```sql
CREATE TABLE country (
  code       character(3)  NOT NULL,
  name       text          NOT NULL,
  population integer       NOT NULL,
  gnp        numeric(10,2)
  -- more columns, but we won't use them here
)
```


### Parameterized Queries

Let's set up our Country class and re-run last chapter's query just to review.

```scala
case class Country(code: String, name: String, pop: Int, gnp: Option[Double])
```

```scala
scala> (sql"select code, name, population, gnp from country"
     |   .query[Country].process.take(5).quick.run)
Country(AFG,Afghanistan,22720000,Some(5976.0))
Country(NLD,Netherlands,15864000,Some(371362.0))
Country(ANT,Netherlands Antilles,217000,Some(1941.0))
Country(ALB,Albania,3401200,Some(3205.0))
Country(DZA,Algeria,31471000,Some(49982.0))
```

Still works. Ok. 

So let's factor our query into a method and add a parameter that selects only the countries with a population larger than some value the user will provide. We insert the `minProp` argument into our SQL statement as `$minPop`, just as if we were doing string interpolation.

```scala
def biggerThan(minPop: Int) = sql"""
  select code, name, population, gnp 
  from country
  where population > $minPop
""".query[Country]
```

And when we run the query ... surprise, it works!

```scala
scala> biggerThan(150000000).quick.run // Let's see them all
Country(BRA,Brazil,170115000,Some(776739.0))
Country(IDN,Indonesia,212107000,Some(84982.0))
Country(IND,India,1013662000,Some(447114.0))
Country(CHN,China,1277558000,Some(982268.0))
Country(PAK,Pakistan,156483000,Some(61289.0))
Country(USA,United States,278357000,Some(8510700.0))
```

So what's going on? It looks like we're just dropping a string literal into our SQL string, but actually wer'e constructing a proper parameterized `PreparedStatement`, and the `minProp` value is ultimately set via a call to `setInteger` (see "Diving Deeper" below.)

**doobie** allows you to interpolate any JVM type that has a target mapping defined by the JDBC spec, plus vendor-specific types and custom column types that you define. We will discuss custom type mappings in a later chapter.

### Diving Deeper

TODO: explain in terms of `hi` combinators.







