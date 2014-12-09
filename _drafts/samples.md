---
layout: post
title: Doobie Examples
tags: scala
---

## Setting Up

If you want to play along at home you can import the `world.sql` database into a local database. It works fine in H2 and PostgreSQL (which is what is used in the example output).

In your REPL you will want to do some imports.

```scala
import doobie.imports._
import scalaz._, Scalaz._
import scalaz.concurrent.Task
```

Set up a `Transactor` in the REPL so we can connect to our World database, and import its `Yolo` module, which adds some syntax for doing things quickly in the REPL.

```scala
val xa: Transactor[Task] = DriverManagerTransactor[Task](
    "org.postgresql.Driver", 
    "jdbc:postgresql:world", 
    "rnorris", 
    ""
  )
 
import xa.yolo._
```


## Select Some Data

#### Basic SELECT, with the `sql` Interpolator

Use the `sql` interpolator to construct a parameterized query. The result is a `Query0` parameterized by the returned row type, which is a tuple for now.

```scala
scala> def pop(n: Int) = sql"""
           select name, population 
           from country 
           where population > $n
         """.query[(String, Int)]
pop: (n: Int)doobie.util.query.Query0[(String, Int)]
```

It's just a value; nothing "happens" when we construct it.

```scala
scala> val bigCountries = pop(100000000)
bigCountries: doobie.util.query.Query0[(String, Int)] = ...
```

We can use `quick` in the REPL to stream results to the console.

```scala
scala> bigCountries.quick.run
  (Bangladesh,129155000)
  (Brazil,170115000)
  (Indonesia,212107000)
  (India,1013662000)
  (Japan,126714000)
  (China,1277558000)
  (Nigeria,111506000)
  (Pakistan,156483000)
  (Russian Federation,146934000)
  (United States,278357000)

scala>
```

Or get the `process` and add some operations.

```scala
scala> bigCountries.process.drop(1).take(5).quick.run
  (Brazil,170115000)
  (Indonesia,212107000)
  (India,1013662000)
  (Japan,126714000)
  (China,1277558000)

scala>
```

Collect the results as a `List`. Since we're computing a result (not just printing it out) we don't need the `Yolo` syntax here.

```scala
scala> bigCountries.process.take(3).list.transact(xa).run
res38: List[(String, Int)] = List((Bangladesh,129155000), (Brazil,170115000), (Indonesia,212107000))
```

Let's break that down.

```scala

// Start with a Query0[(String, Int)]
bigCountries    

// `process` gets us a scalaz stream                                   
bigCountries.process          // Process[ConnectionIO,(String, Int)] 
bigCountries.process.take(3)  // Process[ConnectionIO,(String, Int)]

// `list` is equivalent to `.runLog.run.map(_.toList)`
bigCountries.process.take(3).list // ConnectionIO[List[(String, Int)]]

// `transact` transforms `ConnectionIO` ~> `Task` 
bigCountries.process.take(3).list.transact(xa) // Task[List[(String, Int)]]

// `run` performs the side-effecting computation
bigCountries.process.take(3).list.transact(xa).run // List[(String, Int)]
```


#### Check our Query

`Yolo` syntax adds a `check` method that constructs and prints out an analysis of our query.

```
scala> bigCountries.check.run

  select name, population from country where population > ?

  ✓ SQL Compiles and Typechecks
  ✓ P01 Int  →  INTEGER (int4)
  ✓ C01 name       VARCHAR (text) NOT NULL  →  String
  ✓ C02 population INTEGER (int4) NOT NULL  →  Int

scala>
```

This tells us that the SQL is consistent with the schema, and that our parameter and output types are sensible. If we change the `population` column to `gnp` we see that the situation changes.

```scala
scala> def pop(n: Int) = sql"""
           select name, gnp 
           from country 
           where population > $n
         """.query[(String, Int)]
pop: (n: Int)doobie.util.query.Query0[(String, Int)]
```
```
scala> pop(0).check.run // argument doesn't matter since we don't actually run the query

  select name, gnp from country where population > ?

  ✓ SQL Compiles and Typechecks
  ✓ P01 Int  →  INTEGER (int4)
  ✓ C01 name VARCHAR (text)    NOT NULL  →  String
  ✕ C02 gnp  NUMERIC (numeric) NULL      →  Int
    - NUMERIC (numeric) is ostensibly coercible to Int according to the JDBC specification but is not a
      recommended target type. Fix this by changing the schema type to INTEGER; or the Scala type to
      BigDecimal.
    - Reading a NULL value into Int will result in a runtime failure. Fix this by making the schema type
      NOT NULL or by changing the Scala type to Option[Int]

scala>
```

Here we see that there is a misalignment of both type and nullability. These checks are useful when experimenting in the REPL, and can also be done as part of a unit test with the `doobie-contrib-specs2` package.


#### SELECT with `doobie.hi`

nu


## Inserting


## Updating


