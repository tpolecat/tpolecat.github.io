---
layout: post
title: By-Name Parameters in Scala
tags: scala
---

A common question on `#scala` is, what's the difference between a parameter of type `A` and one of type `=> A`? The short answer is:

> **TL;DR** ... A parameter of type `A` acts like a `val` (its body is evaluated once, when bound) and one of type `=> A` acts like a `def` (its body is evaluated whenever it is used).

Ok. So let's work an example by attempting to define our own `if/else` control structure:

```scala
// Our own if/then/else 
def when[A](test: Boolean, whenTrue: A, whenFalse: A): A = 
  test match {
    case true  => whenTrue
    case false => whenFalse
  }

scala> when(1 == 2, "foo", "bar")
res13: String = bar

scala> when(1 == 1, "foo", "bar")
res14: String = foo

// Ok so far, but...

scala> when(1 == 1, println("foo"), println("bar"))
foo
bar

scala> // hmmm
```

The problem here is that the arguments to `when` are always evaluated, even if they are never used. This means that at best we're doing extra work, and at worst we're triggering side-effects that should not happen. 

Scala has a solution to this problem called **by-name parameters**. By declaring a parameter as `a: => A` (note that the space after the `:` is necessary) we are telling Scala to evalate `a` only when it is used (which may be never). So let's fix our method.

```scala
def when[A](test: Boolean, whenTrue: => A, whenFalse: => A): A = 
  test match {
    case true  => whenTrue
    case false => whenFalse
  }

// Try that again...

scala> when(1 == 1, println("foo"), println("bar"))
foo

scala> when(1 == 2, println("foo"), println("bar"))
bar

scala> // much better
```

So this is actually quite nice. We have defined a new control structure as a library function, which is something you can't really do in most languages. Although `if` and `while` are built-in syntax in Scala, they don't need to be; and unlike most other languages which define short-circuiting boolean operators in syntax, Scala defines `&&` and `||` simply as methods with by-name parameters.


## So is this laziness?

Not exactly. When we talk about laziness we mean that an expression is reduced **by need**, as is the case with `lazy val` in Scala. In case you're not familiar, let's generate a random number in three ways, first as a `val`:

```scala
scala> val a = { println("computing a"); util.Random.nextInt(10) }
computing a
a: Int = 5

scala> List(a, a, a, a, a)
res4: List[Int] = List(5, 5, 5, 5, 5)
```

So `a` was computed immediately, one time, as expected. What if it's a `def`?

```scala
scala> def a = { println("computing a"); util.Random.nextInt(10) }
a: Int

scala> List(a, a, a, a, a)
computing a
computing a
computing a
computing a
computing a
res5: List[Int] = List(1, 6, 8, 4, 3)
```

Again, no surprises; `a` is computed each time it's used. And what about `lazy val`?

```scala
scala> lazy val a = { println("computing a"); util.Random.nextInt(10) }
a: Int = <lazy>

scala> List(a, a, a, a, a)
computing a
res6: List[Int] = List(4, 4, 4, 4, 4)
```

So the `lazy val` is computed once like any other `val`, but not until its value is needed.

So what happens with a by-name parameter? It might not be what you expect.

```scala
scala> def five[A](a: => A): List[A] = List(a, a, a, a, a)
five: [A](a: => A)List[A]

scala> five { println("computing a"); util.Random.nextInt(10) }
computing a
computing a
computing a
computing a
computing a
res7: List[Int] = List(1, 4, 6, 1, 5)
```

The by-name parameter behaves like the `def` above, *not* like `lazy val`. So although we used by-name parameters to implement lazy behavior for our `when` function above, this is dependent on only referencing each parameter once.

> A by-name parameter acts like a `def`.

Note that we *can* get the lazy behavior we might have expected by combining a by-name parameter with a `lazy val` in the function body.

```scala
scala> def five[A](a: => A): List[A] = { lazy val b = a; List(b, b, b, b, b) }
five: [A](a: => A)List[A]

scala> five { println("computing a"); util.Random.nextInt(10) }
computing a
res8: List[Int] = List(9, 9, 9, 9, 9)
```

## Anything else?

Glad you asked. Here are a few more tidbits that might come in handy:

 - Is there a way to reference a by-name parameter without forcing its evaluation? Yes! A by-name parameter acts like a `def` in another way: you can apply **η-expansion** (put an underscore after the identifier) to turn it into a function value. For much much more on this, go [here](/2014/06/09/methods-functions.html).
 - You might hear people say that a by-name parameter is **non-strict** because it's not evaluated immediately, but this is not technically correct. A parameter is **strict** if its argument is always evaluated, regardless of calling convention; in our `five` function above our parameter is both by-name and strict.

## Done.

Hope this helps. Please let me know if you find any errors above or have suggestions for improving the explanation.







