
# Get your doobie on

Inredients:
- Algebras of all operations for all JDBC types.

```scala
java.sql.Connection       -> ConnectionOp          58 ops
java.sql.PrepardStatement -> PreparedStatementOp  114 ops
java.sql.ResultSet        -> ResultSetOp          199 ops

// (16 algebras, 1000 operations)
```
---
# Get your doobie on

Inredients:
- Free monads thereof, with lifted constructors.

```scala
type ConnectionIO[A]        = Free[ConnectionOp,        A]
type PreparedStatementIO[A] = Free[PreparedStatementOp, A]
type ResultSetIO[A]         = Free[ResultSetOp,         A]

// (16 of these)
```

---
# Get your doobie on

Inredients:
- Resource-safe constructors for embedded sub-programs.

```scala
// Low-level API - 1:1 with JDBC, no resource management
prepareStatement(sql: String)
  : ConnectionIO[PreparedStatent]

// High-level API - resource-safe, unleakable
prepareStatement[A](sql: String)(k: PreparedStatementIO[A])
  : ConnectionIO[A]
```

---
# Get your doobie on

Inredients:
- High-level constructors for column/resultset mapping.

```scala
// Low-level API - 1:1 with JDBC
getString(index: Int)
  : ResutSetIO[String]

// High-level API - column vector read into product types
get[A: Composite](index: Int = 1)
  : ResultSetIO[A]

// or aggrerate structure
list[A: Composite]
  : ResultSetIO[List[A]]
```

---
# Get your doobie on

Inredients:
- `ConnectionIO` constructor via SQL literal.
- Statement and type-mapping checked at compile-time (soon).

```scala
tsql"""
  DELETE FROM woozle WHERE blork = 12
""" // ConnectionIO[Int]

tsql"""
  SELECT blork FROM woozle
""".as[Vector[Int]] // ConnectionIO[Vector[Int]]

// and so on, we will see more of this
```

???

- note that the high and low-level stuff isn't really a stratification; the types are all the same and the only difference is in which constructors you use. So you can compose high and low-level stuff together directly.

---
# Get your doobie on

Inredients:
- Default interpreters into Kleisli arrows for any IO-like monad.

```scala
// Interpreter for Op type
def interpK[M[_]: Monad: Catchable: Capture]
  : CallableStatementOp ~> Kleisli[M, CallableStatement, ?]

// Which via Free.foldMap gives us
def transK[M[_]: Monad: Catchable: Capture]
  : CallableStatementIO ~> Kleisli[M, CallableStatement, ?]

// Or if we have the JDBC type at hand
def trans[M[_]: Monad: Catchable: Capture](c: CallableStatement)
  : CallableStatementIO ~> M

// (for all 16 algebras)
```

???

- So the last one lets you use a doobie program to manipulate a JDBC resource that you already have. So this lets you integrate with existing code, which isn't always possible with other database libraries.

---
# Get your doobie on

Inredients:
- Conveniences for `ConnectionIO`, which is the top-level effect type.

```scala
// Abastract over connection pools, data sources, and so on.
// Provides a pair of Natural transformations that add logic
// to perform operations on logically fresh connections, with
// transaction logic and resource cleanup.
trait Transactor[M[_]] {
  def trans: ConnectionIO ~> M
  def transP: Process[ConnectionIO, ?] ~> Process[M, ?]
}

// Syntax for discharging ConnectionIO
val xa: Transactor[IO] = ...
val cw: ConnectionIO[Woozle] = ...

cw.transact(xa) // IO[Woozle] - doobie types are gone
```

- And there's some convenience machinery to abstract over connection pools and data sources, and syntax to let you interpret a `ConnectionIO` into something you can run.
