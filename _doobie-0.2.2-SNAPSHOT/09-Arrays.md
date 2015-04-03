---
layout: book
number: 9
title: SQL Arrays
---

This chapter shows how we can map Scala sequence types to SQL `ARRAY` types, for vendors that support it. Note that although SQL array mappings are part of the JDBC specification,  their behavior is vendor-specific and requires an add-on library; the code in this chapter requires `doobie-contrib-postgresql`.

### Setting Up

Again we set up a transactor and pull in YOLO mode. We also need an import to get PostgreSQL-specific type mappings.

```scala
import doobie.imports._, scalaz._, Scalaz._, scalaz.concurrent.Task

val xa = DriverManagerTransactor[Task](
  "org.postgresql.Driver", "jdbc:postgresql:world", "postgres", ""
)

import xa.yolo._, doobie.contrib.postgresql.pgtypes._
```

### Reading and Writing Arrays

Let's create a new table with a SQL array column. Note that this is likely to work only for PostgreSQL; the syntax for arrays differs significantly from vendor to vendor.

```scala
val drop = sql"DROP TABLE IF EXISTS person".update.quick

val create = 
  sql"""
    CREATE TABLE person (
      id   SERIAL,
      name VARCHAR   NOT NULL UNIQUE,
      pets VARCHAR[] NOT NULL
    )
  """.update.quick
```

```scala
scala> (drop *> create).run
  0 row(s) updated
  0 row(s) updated
```

**doobie** maps SQL array columns to `Array`, `List`, and `Vector` by default. No special handling is required, other than importing the vendor-specific array support above.

```scala
case class Person(id: Long, name: String, pets: List[String])

def insert(name: String, pets: List[String]): ConnectionIO[Person] = {
  sql"insert into person (name, pets) values ($name, $pets)"
    .update.withUniqueGeneratedKeys("id", "name", "pets")
}
```

Insert works fine, as does reading the result. No surprises.

```scala
scala> insert("Bob", List("Nixon", "Slappy")).quick.run
  Person(1,Bob,List(Nixon, Slappy))

scala> insert("Alice", Nil).quick.run
  Person(2,Alice,List())
```

### Lamentations of `NULL`

**doobie** maps nullable columns via `Option`, so `null` is never observed in programs that use the high-level API, and the typechecking feature discussed earlier will find mismatches. So this means if you have a nullable SQL `varchar[]` then you will be chided grimly if you don't map it as `Option[List[String]]` (or some other supported sequence type).

However there is another axis of variation here: the *array cells* themselves may contain null values. And the query checker can't save you here because this is not reflected in the metadata provided by PostgreSQL or H2, and probably not by anyone.

So there are actually four ways to map an array, and you should carefully consider which is appropriate for your schema. In the first two cases reading a `NULL` cell would result in a `NullableCellRead` exception.

```scala
scala> sql"select array['foo','bar','baz']".query[List[String]].quick.run
  List(foo, bar, baz)

scala> sql"select array['foo','bar','baz']".query[Option[List[String]]].quick.run
  Some(List(foo, bar, baz))

scala> sql"select array['foo',NULL,'baz']".query[List[Option[String]]].quick.run
  List(Some(foo), None, Some(baz))

scala> sql"select array['foo',NULL,'baz']".query[Option[List[Option[String]]]].quick.run
  Some(List(Some(foo), None, Some(baz)))
```

### Diving Deep

We can easily add support for other sequence types like `scalaz.IList` by invariant mapping. The `nxmap` method is a variant of `xmap` that ensures null values read from the database are never observed. The `TypeTag` is required to provide better feedback when type mismatches are detected.

```scala
import scala.reflect.runtime.universe.TypeTag

implicit def IListMeta[A: TypeTag](implicit ev: Meta[List[A]]): Meta[IList[A]] =
  ev.nxmap[IList[A]](IList.fromList, _.toList)
```

Once this mapping is in scope we can map columns directly to `IList`.

```scala
scala> sql"select pets from person where name = 'Bob'".query[IList[String]].quick.run
  [Nixon,Slappy]
```



