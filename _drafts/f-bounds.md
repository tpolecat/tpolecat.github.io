---
layout: post
title: Returning the "Current" Type in Scala
tags: scala
---

A common question on the `#scala` IRC channel is

> I have a type hierarchy ... how do I declare a supertype method that returns the "current" type?

This question comes up a lot because Scala encourages immutability, so methods that return a modified copy of `this` are quite common. Making the return type of such methods sufficiently precise is tricky, and is the subject of this article.

The closest thing to a "standard" approach (and the one used by stdlib collections, for example) is to use an **F-bounded type**, which *mostly* works, but can't fully enforce the constraint we're after (it still takes some discipline and leaves room for error). I must pause to offer a hat-tip to [@nuttycom](https://twitter.com/nuttycom) for his [prior work](http://logji.blogspot.com/2012/11/f-bounded-type-polymorphism-give-up-now.html) exploring the pitfalls of this approach.

A better approach is to use a **typeclass**, which solves the problem neatly and leaves little room for worry. In fact it's worth considering abandoning subtype polymorphism altogether in these situations.

We will examine the problem and both solutions, and finish up by exploring **heterogeneous collections** of these beasties, which involves some pleasantly fancy types. But first let's talk about...

### The Problem

Say we have an open trait for pets, with an unknown number of implementations. Let's say every type of `Pet` has a name, as well as a method that returns an otherwise identical copy with a new name. 

> **Our problem is this:** for any expression `x` with some type `A <: Pet`, ensure that `x.renamed(...)` also has type `A`. To be clear: this is a *static* guarantee that we want, not a runtime property.

Right. So here is our first attempt, and one implementation.

```scala
trait Pet {
  def name: String
  def renamed(newName: String): Pet
}

case class Fish(name: String, age: Int) extends Pet {
  def renamed(newName: String): Fish = copy(name = newName)
}
```

In our `Fish` implementation `name` is implemented via a case class field, and the `renamed` method simply delegates to the generated `copy` method ... but note that it returns a `Fish` rather than a `Pet`. This is allowed because return types are in covariant position; it's always ok to return something **more specific** than what is promised.

Just as a sanity check, we can create a `Fish` and rename it and all is well; the static type of the returned value is what we want.

```scala
scala> val a = Fish("Jimmy", 2)
a: Fish = Fish(Jimmy,2)

scala> val b = a.renamed("Bob")
b: Fish = Fish(Bob,2)
```

However a limitation of this approach is that our trait doesn't actually constrain the implementation very much; you are simply required to return a `Pet`, not necessarily the *same type* of pet. So here is a `Kitty` that turns into a `Fish` when you rename it.

```scala
case class Kitty(name: String, color: Color) {
  def renamed(newName: String): Fish = new Fish(newName, 42) // oops
}
```

We also run into problems trying to abstract over renaming. For example, this attempt at a general renaming method fails to compile because the return type of `renamed` for an arbitrary `A <: Pet` is not specific enough; the best we can do is return `Pet`.

```scala
def esquire[A <: Pet](a: A): A = a.renamed(a.name + ", Esq.")
```
```
<console>:28: error: type mismatch;
 found   : Pet
 required: A
       def esquire[A <: Pet](a: A): A = a.renamed(a.name + ", Esq.")
                                                 ^
```

So this approach doesn't meet our stated goal of requiring that `renamed` return the same type as its receiver, and we can't abstract over our renaming operation. So let's see what we can do if we make our types a bit fancier.



### F-Bounded Types

An F-bounded type is **parameterized over its own subtypes**, which allows us to "pass" the implementing type as an argument to the superclass. The self-referential nature of `Pet[A <: Pet[A]]` is puzzling when you first see it; if it doesn't click for you just keep on reading and it should start making more sense.

```scala
trait Pet[A <: Pet[A]] {
  def name: String
  def renamed(newName: String): A // note this return type
}
```

Ok, so any subtype of `Pet` needs to pass "itself" as a type argument.

```scala
case class Fish(name: String, age: Int) extends Pet[Fish] { // note the type argument
  def renamed(newName: String) = copy(name = newName)
}

```

And all is well.

```scala
scala> val a = Fish("Jimmy", 2)
a: Fish = Fish(Jimmy,2)

scala> val b = a.renamed("Bob")
b: Fish = Fish(Bob,2)
```

This time we **can** write our generic renaming method because we now have a more specific return type for `renamed`: any `Pet[A]` will return an `A`.

```scala
scala> def esquire[A <: Pet[A]](a: A): A = a.renamed(a.name + ", Esq.")
esquire: [A <: Pet[A]](a: A)A

scala> esquire(a)
res8: Fish = Fish(Jimmy, Esq.,2)
```

So this is a big win. We now have a way to talk about the "current" type because it appears as a parameter.

However we still have a problem with lying about what the "current" type is; there is nothing forcing us to pass the correct type argument. So here again is our `Kitty` that turns into a `Fish`.

```scala
case class Kitty(name: String, color: Color) extends Pet[Fish] { // oops
  def renamed(newName: String): Fish = new Fish(newName, 42)
}
```

Rats. What we need is a way to restrict the implementing class *claiming* to be an `A` to *actually* be an `A`. And it turns out that Scala does give us a way to do that: a **self-type** annotation.

```scala
trait Pet[A <: Pet[A]] { this: A => // self-type
  def name: String
  def renamed(newName: String): A 
}
```

Now when we try to define our fishy `Kitty` the compiler says nope.

```scala
case class Kitty(name: String, color: Color) extends Pet[Fish] {
  def renamed(newName: String): Fish = new Fish(newName, 42)
}
```

```
<console>:19: error: illegal inheritance;
 self-type Kitty does not conform to Pet[Fish]'s selftype Fish
       case class Kitty(name: String, color: Color) extends Pet[Fish] {
                                                            ^
```

This boxes us in considerably, and we may think we have won. But alas it turns out that we can still lie about the "current" type by extending *another* type that correctly meets the constraint. Subtyping has provided an unwanted loophole.

```scala
class Mammal(val name: String) extends Pet[Mammal] {
  def renamed(newName: String) = new Mammal(newName)
}

class Monkey(name: String) extends Mammal(name) // hmm, Monkey is a Pet[Mammal]
```

And on it goes. I am not aware of any way to further constrain the F-bounded type. So if we use this technique we can do fairly well, but we still can't totally guarantee that `renamed` meets the specificaton. Also note that the clutter introduced by the type parameter on `Pet` doesn't add any information; it's purely a mechanism to restrict implementations.

So let's try another approach.

### How about a Typeclass?

As is often the case, we can avoid our subtyping-related problems by using a typeclass. Let's redefine `Pet` without our `renamed` method, and instead define an orthogonal [typeclass](/2013/10/12/typeclass.html) to deal with this operation. 

```scala
trait Pet {
  def name: String
}

trait Rename[A <: Pet] {
  def renamed(a: A, newName: String): A
}
```

We can now define `Fish` and an *instance* of `Rename[Fish]`. We make the instance implicit and place it on the companion object so it will be available during implicit search.

```scala
case class Fish(name: String, age: Int) extends Pet

object Fish {
  implicit val FishRename = new Rename[Fish] {
    def renamed(a: Fish, newName: String) = a.copy(name = newName)
  }
}
```

And we can use an implicit class to make this operation act like a method as before. With this extra help any `Pet` with a `Rename` intance will automatically gain a `renamed` method by implicit conversion.

```scala
implicit class PetOps[A <: Pet](a: A) {
  def renamed(newName: String)(implicit ev: Rename[A]) = ev.renamed(a, newName)
}
```

And our simple test still works, although the mechanism is quite different.

```scala
scala> val a = Fish("Jimmy", 2)
a: Fish = Fish(Jimmy,2)

scala> val b = a.renamed("Bob")
b: Fish = Fish(Bob,2)
```

With the typeclass-based design there is no simply way to define a `Rename[Kitty]` instance that returns anything other than another `Kitty`; the types make this quite clear. And our `esquire` method is a snap; the type bounds are different, but the implementation it is identical to the one in the F-bounded case above.

```scala
scala> def esquire[A <: Pet : Rename](a: A): A = a.renamed(a.name + ", Esq.")
esquire: [A <: Pet](a: A)(implicit evidence$1: Rename[A])A

scala> esquire(a)
res10: Fish = Fish(Jimmy, Esq.,2)e
```

blah blah awkward splitting of concerns

### How about *only* a Typeclass?

```scala
trait Pet[A] {
  def name(a: A): String
  def renamed(a: A, newName: String): A
}

implicit class PetOps[A](a: A)(implicit ev: Pet[A]) {
  def name = ev.name(a)
  def renamed(newName: String): A = ev.renamed(a, newName)
}
```


```scala
case class Fish(name: String, age: Int)

object Fish {
  implicit val FishPet = new Pet[Fish] {
    def name(a: Fish) = a.name
    def renamed(a: Fish, newName: String) = a.copy(name = newName)
  }
}
```

```scala
scala> Fish("Bob", 42).renamed("Steve")
res0: Fish = Fish(Steve,42)
```  


### Bonus Round: How do we deal with collections?

An interesting exercise is to consider the case where we have a heterogeneous collection of pets. Specifically, how might we map `esquire` over a `List[Pet]`?


```scala
trait Pet[A <: Pet[A]] {
  def name: String
  def renamed(newName: String): A 
}

case class Fish(name: String, age: Int) extends Pet[Fish] { 
  def renamed(newName: String) = copy(name = newName)
}

case class Kitty(name: String, color: Color) extends Pet[Kitty] {
  def renamed(newName: String) = copy(name = newName)
}

def esquire[A <: Pet[A]](a: A): A = a.renamed(a.name + ", Esq.")

val bob  = Fish("Bob", 12)
val thor = Kitty("Thor", Color.ORANGE)
```

```scala
type AnyPet = A forSome { type A <: Pet[A] }
```

```
scala> List[AnyPet](bob, thor).map { case a => esquire(a) }
res0: List[A forSome { type A <: Pet[A] }] = List(Fish(Bob, Esq.,12), Kitty(Thor, Esq.,java.awt.Color[r=255,g=200,b=0]))
```

> **Pro Tip:** use pattern matching to preserve existentials.

```scala
trait Pet[A] {
  def name(a: A): String
  def renamed(a: A, newName: String): A
}

implicit class PetOps[A](a: A)(implicit ev: Pet[A]) {
  def name = ev.name(a)
  def renamed(newName: String): A = ev.renamed(a, newName)
}

case class Fish(name: String, age: Int)

object Fish {
  implicit object FishPet extends Pet[Fish] {
    def name(a: Fish) = a.name
    def renamed(a: Fish, newName: String) = a.copy(name = newName)
  }
}

case class Kitty(name: String, color: Color)

object Kitty {
  implicit object KittyPet extends Pet[Kitty] {
    def name(a: Kitty) = a.name
    def renamed(a: Kitty, newName: String) = a.copy(name = newName)
  }
}

def esquire[A: Pet](a: A): A = a.renamed(a.name + ", Esq.")

val bob  = Fish("Bob", 12)
val thor = Kitty("Thor", Color.ORANGE)
```


```scala
def sig[A: Pet](a: A): (A, Pet[A]) = (a, implicitly)
```

```
scala> List[(A, Pet[A]) forSome { type A }](sig(bob), sig(thor)).map { case (p, r) => (esquire(p)(r), r) }
warning: there was one feature warning; re-run with -feature for details
res6: List[(A, Pet[A]) forSome { type A }] = List((Fish(Bob, Esq.,12),Fish$$anon$1@1b65f66), (Kitty(Thor, Esq.,java.awt.Color[r=255,g=200,b=0]),Kitty$$anon$2@4b140630))
```

### Wrapping Up




