---
layout: book
number: 10
title: Custom Mappings
---

### Setting Up

```scala
import doobie.imports._, scalaz._, Scalaz._, scalaz.concurrent.Task, java.awt.geom.Point2D

val xa = DriverManagerTransactor[Task](
  "org.h2.Driver",                      
  "jdbc:h2:mem:ch7;DB_CLOSE_DELAY=-1",  
  "sa", ""                              
)

import xa.yolo._
```

### Meta, Atom, and Composite

### Meta by Invariant Map

Let's say we have a class that we want to map to a column.

```scala
case class PersonId(toInt: Int)

val pid = PersonId(42)
```

If we try to use this type for a column value, it doesn't compile. However it gives a useful error message.

```scala
scala> sql"select * from person where id = $pid"
<console>:27: error: Could not find or construct Atom[PersonId]; ensure that PersonId has a Meta instance.
              sql"select * from person where id = $pid"
              ^
```

So how do we create a `Meta[PersonId]` instance? The simplest way is by basing it on an existing instance, using the invariant functor method `xmap`.

```scala
implicit val PersonIdMeta: Meta[PersonId] = 
  Meta[Int].xmap(PersonId, _.toInt)
```

Now it works!

```scala
scala> sql"select * from person where id = $pid"
res1: doobie.syntax.string.SqlInterpolator#Builder = doobie.syntax.string$SqlInterpolator$Source@70b18b04
```


### Composite by Invariant Map

We get `Composite[A]` for free given `Atom[A]`, or for tuples and case classes whose fields have `Composite` instances. This covers a lot of cases, but we still need a way to map other types (non-`case` classes for instance). 


```scala
scala> sql"select x, y from points".query[Point2D.Double]
<console>:27: error: Could not find or construct Composite[java.awt.geom.Point2D.Double].
              sql"select x, y from points".query[Point2D.Double]
                                                ^
```

```scala
implicit val Point2DComposite: Composite[Point2D.Double] = 
  Composite[(Double, Double)].xmap(
    t => new Point2D.Double(t._1, t._2),
    p => (p.x, p.y)
  )
```

```scala
scala> sql"select 12, 42".query[Point2D.Double].unique.quick.run
  Point2D.Double[12.0, 42.0]
```

