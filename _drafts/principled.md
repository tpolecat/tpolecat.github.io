---
layout: post
title: Introduction to Principled Programming in Scala
tags: scala
---

## Goal

Learn the landmarks of typed functional programming.

> I heard this word before, I understood it at the time, and it was not scary.



## Preliminaries:

- What are types?
- What are kinds?

## Polymorphism:

- Subtyping (and variance)
- Parametric (and constraints)
- Ad-Hoc (TBD)

## Core Principles:

- Referential Transparency
- Parametricity

-----------------------------------------

## Mezzanine: Pure Functional IO

Useful to discuss this, since it's probably not at all clear how this could possibly work.


-----------------------------------------


## Typeclasses:

- Ad-hoc polymorphism (explain the name)
- Benefits over subtype polymorphism
- Laws

## Typeclassopedia I: Kind *

- Semigroup
  - work example slowly
  - explain what "is-a" means in this context
- Monoid
- Higher structures (not in scalaz but see Spire)

## Typeclassopedia II: Functors (Kind * -> *)

- Functor
- Contravariant
- Applicative
- Monad

## Monad Bestiary

- Reader
- State
- Writer

## Typeclassopedia IV: Kind * -> * -> *

- Profunctor
- Arrow
- Bifunctor






