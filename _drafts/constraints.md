---
layout: post
title: Type Constraints
tags: scala
---

This is yet another in the continuing series of posts inspired by common questions on `#scala`, usually of the form "what the hell does `<:<` mean?" or similar. It is distilled from a presentation I gave for PDXScala in May of 2013.

## What’s this about?

In languages like Scala that support parametric polymorphism ("generics") we can write methods that operate over any type at all, such as the following:

```scala
def twice[A](a: A) = (a, a)

scala> twice("hi")
res0: (String, String) = (hi,hi)

scala> twice(123)
res1: (Int, Int) = (123,123)
```

But sometimes we want to constrain the type parameter `A` to admit only types with particular characteristics, such as a common superclass or common structural element, or that support a common set of operations. This post describes the "out of the box" mechanisms Scala provides for expressing such constraints.

This sounds boring already, sorry. But it's actually very powerful and people usually end up pretty excited about it once they grasp the potential.

I would characterize this as an "enthusiastic beginner" topic, and I will assume you have some familiarity with parametric polymorphism and implicit values/conversions. But if you don't you can probably keep up anyway. You can always find me on Twitter or `#scala` on FreeNode if you remain confused.

## Subtype Bounds

In languages with both parametric and subtype polymorphism (such as Java and Scala) it is often possible to constrain a type parameter to some upper or lower bound (or both), and in fact all type parameters in Scala have both an upper and lower bound.

> Subtyping constraints are a **built-in** feature of the Scala language. 

For our examples here we will be using the following hierarchy:

```
Animal { def sound: String }
  Mammal
    Cat
    Dog
  Bird
    Owl
	
```

#### Upper Bounds

We specify an upper bound `U` as `A <: U` which states that `A` must be a **subtype** of (or the same as) `U`. If left unspecified, the upper bound is `Any`. Here is an example.

```scala
def withSound[A <: Animal](a:A): (A, String) =
  (a, a.sound)
```

The upper bound is necessary here because `A` appears in the return type.

...

An upper bound can specify a **structural type**, which allows you to abstract over types that have a common structure that is not shared by any common supertype.

```scala
// Abstract over any type that has a close() method
def closing[A <: { def close(): Unit }, B](rsrc: A)(f: A => B): B =
  try f(rsrc) finally rsrc.close()

scala> closing(io.Source.fromFile("/etc/services"))(s => s.getLines.length)
res11: Int = 13921
```

So this method allows us to pass a `Source` even though the constraint specifies no supertype (only a structural element). 

Note that in practice a [typeclass](/2013/10/12/typeclass.html) is usually a better way to solve this kind of problem.

#### Lower Bounds

We specify a lower bound `L` as `A >: L` which states that `A` must be a **supertype** of (or the same as) `L`. If left unspecified, the lower bound is `Nothing`. Here is an example.

...

As with upper bounds it is possible to specify a structural type as the lower bound, but this does not end up being terribly useful as the only supertypes are `AnyRef` and `Any`.

#### Using Both at Once



## Evidence Constraints

Carried by implicit values Must be constructed
•
By you
•
•
By the compiler, via the implicit resolution process

> Evidence constraints are a **derived** feature of the Scala language; they are defined in terms of other language features and are user-extensible. However some have proved to be so useful that they now have [optional] special syntax.

#### View Bounds

```scala
def area(a: Shape): Int = a.width * a.height
def area[A](a: A)(implicit ev: A => Shape): Int = a.width * a.height
```

flip is defined for any type that can be converted [implicitly] to Shape.


```scala
def area[A <% Shape](a: A): Int = a.width * a.height
```

View Bound Syntax
This syntax is exactly equivalent.

#### Context Bounds

```scala
rait Shape[A] {
def reflect(a:A, θ: Double): A = ...
}
def flip[A](a: A)(implicit ev: Shape[A]): A = ev.reflect(a, math.Pi)
```

Defined for any type that has an associated instance of Shape[A].



```scala
trait Shape[A] {
def reflect(a:A, θ: Double): A = ...
}
def flip[A: Shape](a: A): A =
implicitly[Shape[A]].reflect(a, math.Pi)
```

Context Bound Syntax
These are equivalent. This is the typeclass pattern.


#### Equality


```scala
class Foo[A](a:A) {
def toOption:Option[A] = Some(a)
def inc(implicit ev: A =:= Int) = new Foo(a + 1)
}
```

Equality Evidence
inc can only be called if A is known to be Int at compile time.

#### Conformance



```scala
class Foo[A](a:A) { ...
def bytes(a: A)(implicit ev: A <:< Serializable) = byteBlaster.writeObject(a)
}
```
Conformance Evidence
Can only be called if A is known to be Serializable at compile time.


## Example: Working with Arrays

example here with `[A >: Null <: AnyRef: ClassTag]` for constructing an array of `null` values.


## Bonus: Constraints defined by scalaz


#### Leibniz

#### Liskov











