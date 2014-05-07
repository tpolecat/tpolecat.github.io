---
layout: post
title: Highlights from LambdaConf 2014
tags: scala
---

I just returned from [LambdaConf](http://www.degoesconsulting.com/lambdaconf/) in lovely Boulder, Colorado and my mind is full of interesting things I need to write down somewhere. So here goes.

#### Note to readers:

- If I have neglected links to associated resources (slides, etc.) it's because I couldn't find them. Please let me know and I'll update.
- If for some reason I mention your name and you're prefer not to appear in my dumb blog post, please contact me and I'll be happy to redact you.

## TL;DR

It was awesome. Be sure to go next year.

## Still here?

Ok. So, congratulations are in order for **John DeGoes** ([@jdegoes](https://twitter.com/jdegoes)) and his many helpers for putting on such a well-organized conference, especially in its inaugural year. The venues were the right size and easy to walk to from downtown, everything ran more or less on time, food was good, plenty of coffee, etc. There was ample time for socializing, which I normally hate but at this kind of thing I feel that I am among my people and the aggregate introversion cancels out somehow.

Also, Boulder is almost overwhelmingly pleasant, if that makes any sense at all. Highly recommended.

### Highlights: Friday Night Lightning Talks

There were a few lightning talks on Friday night (hopefully there will be many more next year) and among them were a couple gems:

- **Phil Freeman** ([@paf31](https://twitter.com/paf31)) talked about liar's logic and walked through code in Haskell to solve [Smullyan](http://en.wikipedia.org/wiki/Raymond_Smullyan)'s liar puzzles. I think this kind of talk (silly on the surface but actually quite interesting) does very well for short sessions. Makes me think I should do one on my [execrable Scala DSL](https://github.com/tpolecat/basic-dsl) sometime.
- **Jon Pretty** ([@propensive](https://twitter.com/propensive)) talked about algebraic data types along lines similar to the talk and blog series by [Chris Taylor](http://chris-taylor.github.io/blog/2013/02/10/the-algebra-of-algebraic-data-types/), and did so in fine style. He's very entertaining. The garden path leading to type differentiation surprised lots of people.

I gave a kind of meh talk about hiding dangerous APIs in monads (slides will be up at some point; I need to integrate my notes) and somehow I ended up winning a [Sphero](http://www.gosphero.com/). It is creepy and the cat is never going to come out from under the couch.

<center><img src="/assets/sphero.png"/><br><small>Do not taunt happy fun ball.</small></center>

### Highlights: Saturday Sessions

There were three tracks for the morning and two in the afternoon so I can only report on the sessions I attended:

- For the morning keynote **Paul Phillips** ([@extempore2](https://twitter.com/extempore2)) gave another raging volcano talk, this time making the argument that compilers should operate on AST deltas (and languages, to the extent necessary, should be designed to fit this model). Sounds good to me, if only because it sounds like a fun problem to work on. 
- **Phil Freeman** ([@paf31](https://twitter.com/paf31)) gave a [sesson on PureScript](https://github.com/paf31/lambdaconf), which is a Haskell-like language that compiles to Javascript. I spent most of the class trying to get it compiled on my machine so I didn't get to play with the examples, but the demonstration was very impressive. It's much farther along than I had anticipated.
- **Daniel Capo Sobral** ([@dcsobral](https://twitter.com/dcsobral)) talked about Scala puzzlers, but not simply about weird behavior; he discovered by sifting through SO questions that there are far more "what does this mean?" questions for Scala than "how do I do X?", which reinforces my belief that DSLs are a net negative. I'm a bit more sympathetic to complaints about symbolic operator now, but I think these are largely subsumed by the DSL problem.
- The main event as far as I was concerned was the [Idris session](https://github.com/puffnfresh/idris-workshop) by **Brian McKenna** ([@puffnfresh](https://twitter.com/puffnfresh)). I have been defeated several times by the tutorial and was hoping to push past, and I think I can say mission accomplished. The first code example was a [typesafe `printf`](https://www.youtube.com/watch?v=fVBck2Zngjo) which I got working, but not without some baffling unification errors along the way. (Observation: it sucks that the typechecking error messages are so useless in a language where typechecking is the central activity.) We then turned to proving simple theorems, which is cool but kind of opaque to me and will require some study. But all in all I'm really excited about Idris and plan to spend some quality time with it.
- Next up was **Kris Nuttycombe** ([@nuttycom](https://twitter.com/nuttycom)) on complexity and reasoning in FP. The first part of the talk revisited some of the things Jon talked about on Friday night, with an eye on using our knowledge of the cardinality of types to limit the number of states we can represent (IP address as `Byte^4` rather than `String` for example). This is an enormously important idea that is rarely taught. The second part of the talk was about turning programs into values via `Free` which we implemented from scratch. Challenging for Scala and non-Scala folks alike. I had to duck out early so I missed the tail end of this one.
- Last talk was a marathon demo of [Rapture](http://rapture.io/) by **Jon Pretty** ([@propensive](https://twitter.com/propensive)), whom I pestered with questions throughout. Rapture is an IO library that leans heavily on [typeclasses](http://tpolecat.github.io/2013/10/12/typeclass.html) (as well as some eyebrow-raising language features) to provide a very friendly and expressive API. We chatted about it that evening and it sounds like adapting it to existing [typelevel.org](http://typelevel.org/) abstractions should be trivial (i.e., providing brief typeclass instances).


### Missed Opportunities

Sadly I didn't get to do everything I wanted to do:

- **Matthew Flat** gave a talk on modules and languages in [Racket](http://racket-lang.org/) that I wish I could have seen. The Racket ecosystem has been evolving for a really long time and they have quietly accumulated a boatload of impressive technical achievements. Also it started at Rice so it has to be good, right?
- **Colt Frederickson** ([@coltfred](https://twitter.com/coltfred)) and some of his co-workers were eager to discuss [atto](https://github.com/tpolecat/atto) and offered some good suggestion (including a few things I was considering already, which is good validation) but I was called away mid-conversation. So I need to follow up with them.
- I was kind of hoping to get into an argument with **Daniel Spiewak** ([@djspiewak](https://twitter.com/djspiewak)) but our paths never crossed so I didn't have a chance to introduce myself. He led a hike on Sunday that sounded like great fun but I had to get back early. Next year though.
- **Leif Warner** ([@pdxleif](https://twitter.com/pdxleif)) says he and Jon went on a mountain drive which I can only assume was crazy and Mr. Toad-like. I think I would have enjoyed that very much.





