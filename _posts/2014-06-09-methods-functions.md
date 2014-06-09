---
layout: post
title: Methods are not Functions
tags: scala
---

In Scala anyway.

Ok this was going to be a short post that I could link to when people get confused on `#scala`, but it turns out that there are a lot of cases to consider. It would be nice if this could just be the last word on methods vs. functions in Scala, so if I missed anything please let me know.

> **TL;DR** ... Methods in Scala are not values, but functions are. You can construct a function that delegates to a method via η-expansion (triggered by the trailing underscore thingy).

The definition I will be using here is that a **method** is something defined with `def` and a **value** is something you can assign to a `val`.

## The Basic Idea

When we define a method we see that we cannot assign it to a `val`. 


```scala
scala> def add1(n: Int): Int = n + 1
add1: (n: Int)Int

scala> val f = add1
<console>:8: error: missing arguments for method add1;
follow this method with `_' if you want to treat it as a partially applied function
       val f = add1
               ^
```

Note also the **type** of `add1`, which doesn't look normal; you can't declare a variable of type `(n: Int)Int`. Methods are not values.

However, by adding the η-expansion postfix operator (η is pronounced "eta"), we can turn the method into a function value. Note the type of `f`.

```scala
scala> val f = add1 _
f: Int => Int = <function1>

scala> f(3)
res0: Int = 4
```

The effect of `_` is to perform the equivalent of the following: we construct a `Function1` instance that delegates to our method.

```scala
scala> val g = new Function1[Int, Int] { def apply(n: Int): Int = add1(n) }
g: Int => Int = <function1>

scala> g(3)
res18: Int = 4
```

So that's really all there is to it. The rest of this post is just details.

## Automatic Expansion

In contexts where the compiler expects a function type, the desired expansion is inferred and the underscore is not needed:

```scala
scala> List(1,2,3).map(add1 _)
res5: List[Int] = List(2, 3, 4)

scala> List(1,2,3).map(add1)
res6: List[Int] = List(2, 3, 4)
```

This applies in any position where a function type is expected, as is the case with a declared or ascribed type:

```scala
scala> val z = add1
<console>:8: error: missing arguments for method add1;
follow this method with `_' if you want to treat it as a partially applied function
       val z = add1
               ^

scala> val z: Int => Int = add1
z: Int => Int = <function1>

scala> val z = add1 : Int => Int
z: Int => Int = <function1>
```

## Effect of Overloading

In the presence of overloading you must provide enough type information to disambiguate:

```scala
scala> "foo".substring _
<console>:8: error: ambiguous reference to overloaded definition,
both method substring in class String of type (x$1: Int, x$2: Int)String
and  method substring in class String of type (x$1: Int)String
match expected type ?
              "foo".substring _
                    ^

scala> "foo".substring _ : (Int => String)
res14: Int => String = <function1>
```


## Fistful of Parameters

Scala actually has a lot of ways to specify parameters, but they all work with η-expansion. Let's look at each case.

#### Parameterless Methods

Methods with no parameter list follow the same pattern, but in this case the compiler can't tell us about the missing `_` because the invocation is legal on its own.

```scala
scala> def x = println("hi")
x: Unit

scala> val z = x // ok
hi
z: Unit = ()

scala> val z = x _
z: () => Unit = <function0>

scala> z()
hi
```

Note that unlike the method (which has *no* parameter list) the function value has an *empty* parameter list.

#### Multiple Parameters

Methods with multiple parameters expand to equivalent multi-parameter functions:

```scala
scala> def plus(a: Int, b: Int): Int = a + b
plus: (a: Int, b: Int)Int

scala> plus _
res8: (Int, Int) => Int = <function2>
```

Methods with multiple parameter *lists* become curried functions:

```scala
scala> def plus(a: Int)(b: Int): Int = a + b
plus: (a: Int)(b: Int)Int

scala> plus _
res11: Int => (Int => Int) = <function1>
```

Perhaps surprisingly, such methods also need explicit η-expansion when partially applied:

```scala
scala> plus(1)
<console>:9: error: missing arguments for method plus;
follow this method with `_' if you want to treat it as a partially applied function
              plus(1)
                  ^

scala> plus(1) _
res13: Int => Int = <function1>
```

However curried functions do not.

```scala
scala> val x = plus _
x: Int => (Int => Int) = <function1>

scala> x(1) // no underscore needed
res0: Int => Int = <function1>
```

#### Type Parameters

Values in scala cannot have type parameters; when η-expanding a parameterized method all type arguments must be specified (or they will be inferred as non-useful types):

```scala
scala> def id[A](a:A):A = a
id: [A](a: A)A

scala> val x = id _
x: Nothing => Nothing = <function1>

scala> val y = id[Int] _
y: Int => Int = <function1>

scala> y(10)
res2: Int = 10
```

#### Implicit Parameters

Implicit parameters are passed at the point of expansion and do not appear in the type of the constructed function value:

```scala
scala> def foo[N:Numeric](n:N):N = n
foo: [N](n: N)(implicit evidence$1: Numeric[N])N

scala> foo[String] _
<console>:9: error: could not find implicit value for evidence parameter of type Numeric[String]
              foo[String] _
                 ^

scala> foo[Int] _
res3: Int => Int = <function1>

scala> def bar[N](n:N)(implicit ev: Numeric[N]):N = n
bar: [N](n: N)(implicit ev: Numeric[N])N

scala> bar[Int] _
res4: Int => Int = <function1>
```

#### By-Name Parameters

The "by-nameness" of by-name parameters is preserved on expansion: 

```scala
scala> def foo(a: => Unit): Int = 42
foo: (a: => Unit)Int

scala> foo(println("hi"))
res15: Int = 42

scala> val x = foo _
x: (=> Unit) => Int = <function1>

scala> x(println("hi"))
res16: Int = 42
```

Also note that η-expansion can capture a by-name argument and delay its evaluation:

```scala
scala> def foo(a: => Unit): () => Unit = a _
foo: (a: => Unit)() => Unit

scala> val z = foo(println("hi"))
z: () => Unit = <function0>

scala> z()
hi
```

#### Sequence Parameters

Sequence ("vararg") parameters become `Seq` parameters on expansion:

```scala
scala> def foo(as: Int*): Int = as.sum
foo: (as: Int*)Int

scala> def x = foo _
x: Seq[Int] => Int

scala> x(1,2,3)
<console>:10: error: too many arguments for method apply: (v1: Seq[Int])Int in trait Function1
              x(1,2,3)
               ^

scala> x(Seq(1,2,3))
res2: Int = 6

```

#### Default Arguments

Default arguments are ignored for the purposes of η-expansion; it is not possible to use named arguments to simulate partial application.

```scala
scala> def foo(n: Int = 3, s: String) = s * n
foo: (n: Int, s: String)String

scala> foo _
res19: (Int, String) => String = <function2>

scala> foo(42) _
<console>:9: error: not enough arguments for method foo: (n: Int, s: String)String.
Unspecified value parameter s.
              foo(42) _
                 ^
```

## Is that all?

I think so, but let me know if I missed one. There are a lot of cases but no real surprises.

Can't think of anything else to say about this, sorry.
