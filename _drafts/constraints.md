---
layout: post
title: Type Constraints
tags: scala
---

This is yet another in the continuing series of posts inspired by common questions on `#scala`, usually of the form "what the hell does `<:<` mean?" or similar. It is distilled from a presentation I gave for PDXScala in May of 2013.

##￼What’s this about?

In languages like Scala that support parametric polymorphism ("generics") we can write functions that operate over any type at all, such as the following:

```scala
def twice[A](a: A) = (a, a)

scala> twice("hi")
res0: (String, String) = (hi,hi)

scala> twice(123)
res1: (Int, Int) = (123,123)
```

However sometimes we want to constrain the type parameter `A` to admit only types with particular characteristics, such as a common superclass or common structural element, or that support a common set of operations. This post describes the "out of the box" mechanisms Scala provides for expressing such constraints.

This sounds boring already, sorry. But it's actually very powerful and people usually end up pretty excited about it once they grasp the potential.

I would characterize this as an "enthusiastic beginner" topic, and I will assume you have some familiarity with parametric polymorphism and implicit values/conversions. But if you don't you can probably keep up anyway. You can always find me on Twitter or `#scala` on FreeNode if you remain confused.

## Type Bounds

In languages with both parametric and subtype polymorphism (such as Java and Scala) it is often possible to constrain a type parameter to some upper or lower bound (or both), and in fact all type parameters in Scala have both an upper and lower bound.

> Unless specified otherwise, all type parameters in Scala have an upper bound of `Any` and lower bound of `Nothing`.


```scala
def flip[A <: Shape](a: A): A = 
  a.reflect(math.Pi)
```

Upper Bound Syntax
flip is defined for any type that is assignable to Shape.




```scala
val ss: List[Shape] = ...

def prepend[A >: Shape](a: A): List[A] =
  a :: ss
```

Lower Bound Syntax
prepend is defined for any type to which Shape can be assigned.


you can also specify both ...

## Evidence Constraints

Carried by implicit values Must be constructed
•
By you
•
•
By the compiler, via the implicit resolution process

### View Bounds

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

### Context Bounds

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


### Equality


```scala
class Foo[A](a:A) {
def toOption:Option[A] = Some(a)
def inc(implicit ev: A =:= Int) = new Foo(a + 1)
￼}
```

Equality Evidence
￼inc can only be called if A is known to be Int at compile time.

### Conformance



```scala
class Foo[A](a:A) { ...
def bytes(a: A)(implicit ev: A <:< Serializable) = byteBlaster.writeObject(a)
￼}
```
Conformance Evidence
Can only be called if A is known to be Serializable at compile time.

## Bonus Trivia

You can write your own constraints!
 =:= and <:< are library code
They are almost identical
Compiler knows nothing about them 
Defined in Predef.scala


## Exercise!

link to github













