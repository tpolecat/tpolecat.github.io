---
layout: post
title: What's the deal with null?
tags: scala
---

Most people coming from Java or other object-oriented languages are comfortable with at least the *idea* of a universal common supertype. For example the Java type `Object` is the supertype of all reference types (but not primitives, so it's not quite universal). In C++ there is no such common supertype at all, but the *idea* isn't outrageous. 

In Scala we have `Any` as a true universal supertype, primitives included. Every type is either `Any` or a subtype. A true universal supertype like this is called the **Top** type and is common enough in the literature that it has its own fancy symbol ⊤.

So the top of the Scala type hierarchy looks like this, with `Any` at the top and an immediate division between the primitive types `AnyVal` and the reference types `AnyRef`. 

```
            +-- Any --+
            |         |
         AnyVal     AnyRef
          /  \       /  \
          :  :       :  :
          :  :       :  :
          :  :       :  :
```

This distinction is rather artificial; it's an implementation leakage that mirrors the treatment of primitive and reference types in the JVM (which was maintained in Scala for easier interoperability with Java). But this distinction has quite a bit to do with `null` as we shall see in a moment.

Right. So the top bits make some sense. But what about

## The Bottom Bits

It turns out that not only do all types `A` and `B` share a common supertype (a direct consequence of `Any`), they also share a **common subtype**. That is, there is a root at the top *and* at the bottom of the type hierarchy, so it is in fact not a hierarchy but a full lattice.

The universal subtype is called `Nothing`, and in the literature it is called **Bottom** or ⊥.

```
            +-- Any --+
            |         |
         AnyVal     AnyRef
          /  \       /  \
          :  :       :  :
          :  :       :  :
          :  :       :  :

              Nothing 
```


There is quite a lot to say about `Nothing`, oddly enough, but we will not say it here; this is left for another blog post. The short version is: `Nothing` is **uninhabited** (it has no values at all) and therefore evaluating an expression of type `Nothing` cannot return normally.

But what does this have to do with `null`? Well let's see what the **type** of `null` is.

```scala
scala> null
res0: Null = null
```

Okay ... so the value `null` has type uppercase-`Null`. So where does this type fit into the lattice? If you're curious to figure out yourself, play around in the REPL and try to ascribe various types to `null`; you will find that `null : String` compiles but `null : Int` doesn't, and so on, until you reach the following conclusion:

> `Null` is the common subtype of all reference types.
  

```
            +-- Any --+
            |         |
         AnyVal     AnyRef
          /  \       /  \
          :  :       :  :
          :  :       :  :
          :  :       :  :
          :  :       
          :  :       Null  <------- RIGHT HERE YO
                       |
              Nothing -+
```

So what is the practical consequence? It means that `null` (which is the **only** inhabitant of type `Null`) is also an inhabitant of every reference type, simply due to the subtyping relationship. By inserting `Null` into the lattice in the correct place, Scala gets Java-style `null` behavior for free.

## So what?

Ah, well this is the question then. Why do we care?

Well, for one, if we want to write a parameterized function that returns `null` how do we do it? If `null` inhabited every type we could write this:

```scala
scala> def foo[A]: A = null
<console>:7: error: type mismatch;
 found   : Null(null)
 required: A
       def foo[A]: A = null
```

Which doesn't work of course because `A` might be `Int`. And if any subtype of `AnyRef` could be `null` then we could write this (which most Scala programmers expect to work, actually):

```scala
scala> def foo[A <: AnyRef]: A = null
<console>:7: error: type mismatch;
 found   : Null(null)
 required: A
       def foo[A <: AnyRef]: A = null
                                 ^
```

This fails because in this case `A` could be `Nothing`. The correct constraint is `A >: Null` which constrains `A` to precisely the nullable types.

```scala
scala> def foo[A >: Null]: A = null
foo: [A >: Null]=> A
```

## Ok but really, so what?

...











