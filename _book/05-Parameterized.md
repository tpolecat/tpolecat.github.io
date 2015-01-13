---
layout: book
number: 5
title: Parameterized Queries
---

In this chapter we learn how to construct parameterized queries, and introduce the `Composite` typeclass.

### Setting Up

Same as last chapter, so if you're still set up you can skip this section. Otherwise let's set up an in-memory database.

```scala
import doobie.imports._, scalaz._, Scalaz._, scalaz.concurrent.Task

val xa = DriverManagerTransactor[Task](
  "org.h2.Driver",                      // driver class
  "jdbc:h2:mem:ch5;DB_CLOSE_DELAY=-1",  // connect URL
  "sa", ""                              // user and pass
)
