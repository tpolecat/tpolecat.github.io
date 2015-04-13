---
layout: book
number: 4
title: Selecting Data
---

In this chapter we construct some programs that retrieve data from the database and stream it back, mapping to Scala types on the way. We also introduce YOLO mode for experimenting with **doobie** in the REPL.

### Setting Up

First let's get our imports out of the way and set up a `Transactor` as we did before. You can skip this step if you still have your REPL running from last chapter.

```scala
import doobie.imports._, scalaz._, Scalaz._, scalaz.concurrent.Task

val xa = DriverManagerTransactor[Task](
  "org.postgresql.Driver", "jdbc:postgresql:world", "postgres", ""
)
```

We will be playing with the `country` table, shown here for reference.

```sql
CREATE TABLE country (
  code       character(3)  NOT NULL,
  name       text          NOT NULL,
  population integer       NOT NULL,
  gnp        numeric(10,2)
  -- more columns, but we won't use them here
)
```

### Internal Streaming

For our first query let's aim low and select some country names into a `List`, then print out the first few. There are several steps here so we have noted the types along the way.

```scala
scala> (sql"select name from country"
     |   .query[String] // Query0[String]
     |   .list          // ConnectionIO[List[String]]
     |   .transact(xa)  // Task[List[String]]
     |   .run           // List[String]
     |   .take(5).foreach(println))
Afghanistan
Netherlands
Netherlands Antilles
Albania
Algeria
```

Let's break this down a bit.

- `sql"select name from country".query[String]` defines a `Query0[String]`, which is a one-column query that maps each returned row to a `String`. We will get to more interesting row types soon.
- `.list` is a convenience method that streams the results, accumulating them in a `List`, in this case yielding a `ConnectionIO[List[String]]`.
- The rest is familar; `transact(xa)` yields a `Task[List[String]]` which we run, giving us a normal Scala `List[String]` that we print out.

This is ok, but there's not much point reading all the results from the database when we only want the first few rows. So let's try a different approach.

```scala
scala> (sql"select name from country"
     |   .query[String] // Query0[String]
     |   .process       // Process[ConnectionIO, String]
     |   .take(5)       // Process[ConnectionIO, String]
     |   .list          // ConnectionIO[List[String]]
     |   .transact(xa)  // Task[List[String]]
     |   .run           // List[String]
     |   .foreach(println))
Afghanistan
Netherlands
Netherlands Antilles
Albania
Algeria
```

The difference here is that `process` gives us a `scalaz.stream.Process[ConnectionIO, String]` which emits the results as they arrive from the database. By applying `take(5)` we instruct the process to shut everything down (and clean everything up) after five elements have been emitted. This is much more efficient than pulling all 239 rows and then throwing most of them away.

Of course a server-side `LIMIT` would be an even better way to do this (for databases that support it), but in cases where you need client-side filtering or other custom postprocessing, `Process` is a very general and powerful tool. For more information see the [scalaz-stream](https://github.com/scalaz/scalaz-stream) site, which has a good list of learning resources. 


### YOLO Mode

The API we have seen so far is ok, but it's tiresome to keep saying `transact(xa)` and doing `foreach(println)` to see what the results look like. So **just for REPL exploration** there is a module of extra syntax provided on `Transactor` that you can import, and it gives you some shortcuts.

```scala
import xa.yolo._
```

We can now run our previous query in an abbreviated form.

```scala
scala> (sql"select name from country"
     |   .query[String] // Query0[String]
     |   .process       // Process[ConnectionIO, String]
     |   .take(5)       // Process[ConnectionIO, String]
     |   .quick         // Task[Unit]
     |   .run)
  Afghanistan
  Netherlands
  Netherlands Antilles
  Albania
  Algeria
```

This syntax allows you to quickly run a `Query0[A]` or `Process[ConnectionIO, A]` and see the results printed to the console. This isn't a huge deal but it can save you some keystrokes when you're just messing around. 

- The `.quick` method sinks the stream to standard out (adding ANSI coloring for fun) and then calls `.transact`, yielding a `Task[Unit]`. 
- The `.check` method returns a `Task[Unit]` that performs a metadata analysis on the provided query and asserted types and prints out a report. This is covered in detail in the chapter on typechecking queries.

### Multi-Column Queries

We can select multiple columns, of course, and map them to a tuple. The `gnp` column in our table is nullable so we'll select that one into an `Option[Double]`. In a later chapter we'll see how to check the types to be sure they're sensible.

```scala
scala> (sql"select code, name, population, gnp from country"
     |   .query[(String, String, Int, Option[Double])]
     |   .process.take(5).quick.run)
  (AFG,Afghanistan,22720000,Some(5976.0))
  (NLD,Netherlands,15864000,Some(371362.0))
  (ANT,Netherlands Antilles,217000,Some(1941.0))
  (ALB,Albania,3401200,Some(3205.0))
  (DZA,Algeria,31471000,Some(49982.0))
```
**doobie** automatically supports row mappings for atomic column types, as well as options, tuples, `HList`s and case classes thereof. So let's try the same query with an `HList`:

```scala
scala> import shapeless._
import shapeless._

scala> (sql"select code, name, population, gnp from country"
     |   .query[String :: String :: Int :: Option[Double] :: HNil]
     |   .process.take(5).quick.run)
  AFG :: Afghanistan :: 22720000 :: Some(5976.0) :: HNil
  NLD :: Netherlands :: 15864000 :: Some(371362.0) :: HNil
  ANT :: Netherlands Antilles :: 217000 :: Some(1941.0) :: HNil
  ALB :: Albania :: 3401200 :: Some(3205.0) :: HNil
  DZA :: Algeria :: 31471000 :: Some(49982.0) :: HNil
```

And again, mapping rows to a case class.

```scala
case class Country(code: String, name: String, pop: Int, gnp: Option[Double])
```

```scala
scala> (sql"select code, name, population, gnp from country"
     |   .query[Country] // Query0[Country]
     |   .process.take(5).quick.run)
  Country(AFG,Afghanistan,22720000,Some(5976.0))
  Country(NLD,Netherlands,15864000,Some(371362.0))
  Country(ANT,Netherlands Antilles,217000,Some(1941.0))
  Country(ALB,Albania,3401200,Some(3205.0))
  Country(DZA,Algeria,31471000,Some(49982.0))
```

You can also nest case classes, `HList`s, and/or tuples arbitrarily as long as the eventual members are of supported columns types. For instance, here we map the same set of columns to a tuple of two case classes:

```scala
case class Code(code: String)

case class Country(name: String, pop: Int, gnp: Option[Double])
```

```scala
scala> (sql"select code, name, population, gnp from country"
     |   .query[(Code, Country)] // Query0[(Code, Country)]
     |   .process.take(5).quick.run)
  (Code(AFG),Country(Afghanistan,22720000,Some(5976.0)))
  (Code(NLD),Country(Netherlands,15864000,Some(371362.0)))
  (Code(ANT),Country(Netherlands Antilles,217000,Some(1941.0)))
  (Code(ALB),Country(Albania,3401200,Some(3205.0)))
  (Code(DZA),Country(Algeria,31471000,Some(49982.0)))
```

And just for fun, since the `Code` values are constructed from the primary key, let's turn the results into a `Map`. Trivial but useful.

```scala
scala> (sql"select code, name, population, gnp from country"
     |    .query[(Code, Country)] // Query0[(Code, Country)]
     |    .process.take(5)        // Process[ConnectionIO, (Code, Country)]
     |    .list                   // ConnectionIO[List[(Code, Country)]]
     |    .map(_.toMap)           // ConnectionIO[Map[Code, Country]]
     |    .quick.run)
  Map(Code(ANT) -> Country(Netherlands Antilles,217000,Some(1941.0)), Code(DZA) -> Country(Algeria,31471000,Some(49982.0)), Code(ALB) -> Country(Albania,3401200,Some(3205.0)), Code(NLD) -> Country(Netherlands,15864000,Some(371362.0)), Code(AFG) -> Country(Afghanistan,22720000,Some(5976.0)))
```

### Final Streaming

In the examples above we construct a `Process[ConnectionIO, A]` and discharge it via `.list` (which is just shorthand for `.runLog.map(_.toList)`), yielding a `ConnectionIO[List[A]]` which eventually becomes a `Task[List[A]]`. So the construction and execution of the `Process` is entirely internal to the **doobie** program.

However in some cases a stream is what we want as our "top level" type. For example, [http4s](https://github.com/http4s/http4s) can use a `Process[Task, A]` directly as a response type, which could allow us to stream a resultset directly to the network socket. We can achieve this in **doobie** by calling `transact` directly on the `Process[ConnectionIO, A]`.

```scala
scala> val p = {
     |   sql"select name, population, gnp from country"
     |     .query[Country]  // Query0[Country]
     |     .process         // Process[ConnectionIO, Country]
     |     .transact(xa)    // Process[Task, Country]
     |  }
p: scalaz.stream.Process[scalaz.concurrent.Task,Country] = Await(scalaz.concurrent.Task@70a934a7,<function1>)

scala> p.take(5).runLog.run.foreach(println)
Country(Afghanistan,22720000,Some(5976.0))
Country(Netherlands,15864000,Some(371362.0))
Country(Netherlands Antilles,217000,Some(1941.0))
Country(Albania,3401200,Some(3205.0))
Country(Algeria,31471000,Some(49982.0))
```



### Diving Deeper

The `sql` interpolator is sugar for constructors defined in the `doobie.hi.connection` module, aliased as `HC` if you use the standard imports. Using these constructors directly, the above program would look like this:

```scala
val sql = "select code, name, population, gnp from country"

val proc = HC.process[(Code, Country)](sql, ().point[PreparedStatementIO])

(proc.take(5)        // Process[ConnectionIO, (Code, Country)]
     .list           // ConnectionIO[List[(Code, Country)]]
     .map(_.toMap)   // ConnectionIO[Map[Code, Country]]
     .quick.run)
```

The `process` combinator is parameterized on the process element type and consumes a sql statement and a program in `PreparedStatementIO` that sets input parameters and any other pre-execution configuration. In this case the "prepare" program is a no-op.





