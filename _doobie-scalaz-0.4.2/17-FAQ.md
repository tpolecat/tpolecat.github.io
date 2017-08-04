---
layout: book
number: 17
title: Frequently-Asked Questions
---

In this chapter we address some frequently-asked questions, in no particular order. First a bit of set-up.

```scala
import doobie.imports._
import java.awt.geom.Point2D
import java.util.UUID
import scalaz._, Scalaz._
import shapeless._

val xa = DriverManagerTransactor[IOLite](
  "org.postgresql.Driver", "jdbc:postgresql:world", "postgres", ""
)

val y = xa.yolo; import y._
```

### My IDE is freaking out! What can I do?

Not much, sadly. Neither Scala's presentation compiler (used by Scala-IDE and ENSIME) nor Intellij's custom typechecker can deal with shapeless `ProductArgs` which is used by the `sql` and `fr/fr0` interpolators.

There are open issues for IntelliJ that you can vote up if you like.

- [SCL-10091](https://youtrack.jetbrains.com/issue/SCL-10091)
- [SCL-10928](https://youtrack.jetbrains.com/issue/SCL-10928)

It has been [reported](https://github.com/tpolecat/doobie/issues/508) that `-Ymacro-expand:none` improves the behavior of Scala-IDE, so you might investigate that (and please comment on the issue with your experience).

### How do I do an `IN` clause?

This used to be very irritating, but as of 0.4.0 there is a good solution. See the section on `IN` clauses in [Chapter 5](05-Parameterized.html) and [Chapter 8](08-Fragments.html) on statement fragments.

### How do I ascribe an SQL type to an interpolated parameter?

Interpolated parameters are replaced with `?` placeholders, so if you need to ascribe an SQL type you can use vendor-specific syntax in conjunction with the interpolated value. For example, in PostgreSQL you use `:: type`:

```scala
scala> val s = "foo"
s: String = foo

scala> sql"select $s".query[String].check.unsafePerformIO

  select ?

  ✕ SQL Compiles and Typechecks
    - ERROR: could not determine data type of parameter $1

scala> sql"select $s :: char".query[String].check.unsafePerformIO

  select ? :: char

  ✓ SQL Compiles and Typechecks
  ✓ P01 String  →  CHAR (bpchar)
  ✓ C01 bpchar CHAR (bpchar) NULL?  →  String
```

### How do I do several things in the same transaction?

You can use a `for` comprehension to compose any number of `ConnectionIO` programs, and then call `.transact(xa)` on the result. All of the composed programs will run in the same transaction. For this reason it's useful for your APIs to expose values in `ConnectionIO`, so higher-level code can place transaction boundaries as needed.

### How do I run something outside of a transaction?

`Transactor.transact` takes a `ConnectionIO` and constructs a `Task` or similar that will run it in a single transaction, but it is also possible to include transaction boundaries *within* a `ConnectionIO`, and to disable transaction handling altogether. Some kinds of DDL statements may require this for some databases. You can define a combinator to do this for you.

```scala
/**
 * Take a program `p` and return an equivalent one that first commits any ongoing transaction, runs
 * `p` without transaction handling, then starts a new transaction.
 */
def withoutTransaction[A](p: ConnectionIO[A]): ConnectionIO[A] =
  FC.setAutoCommit(true) *> p <* FC.setAutoCommit(false)
```

Note that you need both of these operations if you are using a `Transactor` because it will always start a transaction and will try to commit on completion.


### How do I turn an arbitrary SQL string into a `Query0/Update0`?

As of **doobie** 0.4.0 this is done via [statement fragments](08-Fragments.html). Here we choose the sort order dynamically.

```scala
case class Code(country: String)
case class City(code: Code, name: String, population: Int)

def cities(code: Code, asc: Boolean): Query0[City] = {
  val ord = if (asc) fr"ASC" else fr"DESC"
  val sql = fr"""
    SELECT countrycode, name, population
    FROM   city
    WHERE  countrycode = $code
    ORDER BY name""" ++ ord
  sql.query[City]
}
```

We can check the resulting `Query0` as expected.

```
scala> cities(Code("USA"), true).check.unsafePerformIO

      SELECT countrycode, name, population
      FROM   city
      WHERE  countrycode = ?
      ORDER BY name ASC 

  ✓ SQL Compiles and Typechecks
  ✓ P01 Code  →  CHAR (bpchar)
  ✓ C01 countrycode CHAR    (bpchar)  NOT NULL  →  Code
  ✓ C02 name        VARCHAR (varchar) NOT NULL  →  String
  ✓ C03 population  INTEGER (int4)    NOT NULL  →  Int
```

And it works!

```scala
scala> cities(Code("USA"), true).process.take(5).quick.unsafePerformIO
  City(Code(USA),Abilene,115930)
  City(Code(USA),Akron,217074)
  City(Code(USA),Albany,93994)
  City(Code(USA),Albuquerque,448607)
  City(Code(USA),Alexandria,128283)

scala> cities(Code("USA"), false).process.take(5).quick.unsafePerformIO
  City(Code(USA),Yonkers,196086)
  City(Code(USA),Worcester,172648)
  City(Code(USA),Winston-Salem,185776)
  City(Code(USA),Wichita Falls,104197)
  City(Code(USA),Wichita,344284)
```

### How do I handle outer joins?

With an outer join you end up with set of nullable columns, which you typically want to map to a single `Option` of some composite type. The most straightforward way to do this is to select the `Option` columns directly, then use the `map` method on `Query0` to transform the result type using applicative composition on the optional values:

```scala
case class Country(name: String, code: String)
case class City(name: String, district: String)

val join: Query0[(Country, Option[City])] =
  sql"""
    select c.name, c.code,
           k.name, k.district
    from country c
    left outer join city k
    on c.capital = k.id
  """.query[(Country, Option[String], Option[String])].map {
    case (c, n, d) => (c, (n |@| d)(City))
  }
```

Some examples, filtered for size.

```scala
scala> join.process.filter(_._1.name.startsWith("United")).quick.unsafePerformIO
  (Country(United Arab Emirates,ARE),Some(City(Abu Dhabi,Abu Dhabi)))
  (Country(United Kingdom,GBR),Some(City(London,England)))
  (Country(United States,USA),Some(City(Washington,District of Columbia)))
  (Country(United States Minor Outlying Islands,UMI),None)
```

### How do I resolve `error: Could not find or construct Param[...]`?

When we use the `sql` interpolator we require a `Param` instance for an `HList` composed of the types of the interpolated query parameters. For instance, in the following code (which has parameters of type `String` and `UUID`, in that order) we need a `Param[String :: UUID :: HNil]` and none is available.

```
scala> def query(s: String, u: UUID) = sql"… $s … $u …".query[Int]
<console>:31: error: Could not find or construct Param[shapeless.::[String,shapeless.::[java.util.UUID,shapeless.HNil]]].
Ensure that this type is an atomic type with an Atom instance in scope, or is an HList whose members
have Atom instances in scope. You can usually diagnose this problem by trying to summon the Atom
instance for each element in the REPL. See the FAQ in the Book of Doobie for more hints.
       def query(s: String, u: UUID) = sql"… $s … $u …".query[Int]
                                       ^
```

Ok, so the message suggests that we need an `Meta[A]` for each `A` or `Option[A]` in the `HList`, so let's see which one is missing by trying to summon them in the REPL.

```
scala> Meta[String]
res11: doobie.util.meta.Meta[String] = doobie.util.meta$Meta$$anon$3@5e903e98

scala> Meta[UUID]
<console>:31: error: Could not find an instance of Meta[java.util.UUID]; you can construct one based on a primitive instance via `xmap`.
       Meta[UUID]
           ^
```

So what this means is that we have not defined a mapping for the `UUID` type to an underlying JDBC type, and **doobie** doesn't know how to set an argument of that type on the underlying `PreparedStatement`. So we have a few choices. We can `xmap` from an existing `Meta` instance, as described in [Chapter 10](10-Custom-Mappings.html); or we can import a provided mapping from a vendor-specific `contrib` package. Since we're using PostgreSQL here, let's do that.

```scala
scala> import doobie.postgres.imports.UuidType
import doobie.postgres.imports.UuidType
```

Having done this, the `Meta` and `Param` instances are now present and our code compiles.

```scala
scala> Meta[UUID]
res13: doobie.util.meta.Meta[java.util.UUID] = doobie.util.meta$Meta$$anon$4@2efed74f

scala> Param[String :: UUID :: HNil]
res14: doobie.util.param.Param[shapeless.::[String,shapeless.::[java.util.UUID,shapeless.HNil]]] = Param(doobie.util.composite$LowerPriorityComposite$$anon$9@47ca1a2f)

scala> def query(s: String, u: UUID) = sql"select ... where foo = $s and url = $u".query[Int]
query: (s: String, u: java.util.UUID)doobie.util.query.Query0[Int]
```

### How do I resolve `error: Could not find or construct Composite[...]`?

When we use the `sql` interpolator and use the `.query[A]` method we require a `Composite` instance for the output type `A`, which we can define directly (as described in [Chapter 10](10-Custom-Mappings.html)) or derive automatically if `A` has a `Meta` instance, or is an option thereof, or a product type whose elements have `Composite` instances.

```scala
case class Point(lat: Double, lon: Double)
case class City(name: String, loc: Point)
case class State(name: String, capitol: City)
```

In this case if we were to say `.query[State]` the derivation would be automatic, because all elements of the "flattened" structure have `Meta` instances for free.

```scala
State(String, City(String, Point(Double, Double))) // our structure
     (String,     (String,      (Double, Double))) // is isomorphic to this
      String,      String,       Double, Double    // so we expect a column vector of this shape
```

But what if we wanted to use AWT's `Point2D.Double` instead of our own `Point` class?

```scala
case class City(name: String, loc: Point2D.Double)
case class State(name: String, capitol: City)
```

The derivation now fails.

```
scala> sql"…".query[State]
<console>:34: error: Could not find or construct Composite[State].

  If State is a simple type (or option thereof) that maps to a single column, you're
  probably missing a Meta instance. If State is a product type (typically a case class,
  tuple, or HList) then probably one of its component types is missing a Composite instance. You can
  usually diagnose this by evaluating Composite[Foo] for each element Foo of the product type in
  question. See the FAQ in the Book of Doobie for more hints.

       sql"…".query[State]
                   ^
```

And if we look at the flat structure it's clear that the culprit has to be `Point2D.Double` since we know `String` has a defined column mapping.

```scala
State(String, City(String, Point2D.Double)) // our structure
     (String,     (String, Point2D.Double)) // is isomorphic to this
      String,      String, Point2D.Double   // so we expect a column vector of this shape
```

And indeed this type has no column vector mapping.

```
scala> Composite[Point2D.Double]
<console>:32: error: Could not find or construct Composite[java.awt.geom.Point2D.Double].

  If java.awt.geom.Point2D.Double is a simple type (or option thereof) that maps to a single column, you're
  probably missing a Meta instance. If java.awt.geom.Point2D.Double is a product type (typically a case class,
  tuple, or HList) then probably one of its component types is missing a Composite instance. You can
  usually diagnose this by evaluating Composite[Foo] for each element Foo of the product type in
  question. See the FAQ in the Book of Doobie for more hints.

       Composite[Point2D.Double]
                ^
```

If this were an atomic type it would be a matter of importing or defining a `Meta` instance, but here we need to define a `Composite` directly because we're mapping a type with several members. As this type is isomorphic to `(Double, Double)` we can simply base our mapping off of the existing `Composite`.

```scala
implicit val Point2DComposite: Composite[Point2D.Double] =
  Composite[(Double, Double)].xmap(
    (t: (Double, Double)) => new Point2D.Double(t._1, t._2),
    (p: Point2D.Double) => (p.x, p.y)
  )
```

Our derivation now works and the code compiles.

```scala
scala> sql"…".query[State]
res17: doobie.util.query.Query0[State] = doobie.util.query$Query$$anon$4@787c8c05
```

### How do I time query execution?

### How do I log the SQL produced for my query after interpolation?

As of **doobie** 0.4 there is a reasonable solution to the logging/instrumentation question. See [Chapter 10](10-Logging.html) for more details.

### Why is there no `Meta[SQLXML]`?

There are a lot of ways to handle `SQLXML` so there is no pre-defined strategy, but here is one that maps `scala.xml.Elem` to `SQLXML` via streaming.

```scala
import doobie.enum.jdbctype.Other
import java.sql.SQLXML
import scala.xml.{ XML, Elem }

implicit val XmlMeta: Meta[Elem] =
  Meta.advanced[Elem](
    NonEmptyList(Other),
    NonEmptyList("xml"),
    (rs, n) => XML.load(rs.getObject(n).asInstanceOf[SQLXML].getBinaryStream),
    (ps, n,  e) => {
      val sqlXml = ps.getConnection.createSQLXML
      val osw = new java.io.OutputStreamWriter(sqlXml.setBinaryStream)
      XML.write(osw, e, "UTF-8", false, null)
      osw.close
      ps.setObject(n, sqlXml)
    },
    (_, _,  _) => sys.error("update not supported, sorry")
  )
```

## How do I set the chunk size for streaming results?

By default streams constructed with the `sql` interpolator are fetched `Query.DefaultChunkSize` rows at a time (currently 512). If you wish to change this chunk size you can use `processWithChunkSize` for queries, and `withGeneratedKeysWithChunkSize` for updates that return results.
