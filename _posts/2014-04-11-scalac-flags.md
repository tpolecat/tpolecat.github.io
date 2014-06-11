---
layout: post
title: Useful Scalac Flags
tags: scala
---

So, there have been some discussions and angry tweets recently about irritating Scala "features" (like value discarding and auto-tupling) that can actually be turned off by selecting the right compiler flag in conjunction with `-Xfatal-warnings`. I *highly* recommend a set of options something like those below.

```scala
scalacOptions ++= Seq(
  "-deprecation",           
  "-encoding", "UTF-8",       // yes, this is 2 args
  "-feature",                
  "-language:existentials",
  "-language:higherKinds",
  "-language:implicitConversions",
  "-unchecked",
  "-Xfatal-warnings",       
  "-Xlint",
  "-Yno-adapted-args",       
  "-Ywarn-dead-code",        // N.B. doesn't work well with the ??? hole
  "-Ywarn-numeric-widen",   
  "-Ywarn-value-discard",
  "-Xfuture"     
)
```

If you're like me you may and you find that the standard library gets in the way (especially `Predef` implicits) you might consider one of these (you don't need both):

```scala
  "-Yno-predef"   // no automatic import of Predef (removes irritating implicits)
  "-Yno-imports"  // no automatic imports at all; all symbols must be imported explicitly
```

... which inspired this contribution from @timperrett

<center><img src="/assets/yuno.jpg"></center>

I also recommend keeping an eye on [warteremover](https://github.com/puffnfresh/wartremover) which casts a wider net and disables many other unsafe features such as `null` and `return`. If you try it out and find it's hitting false positives (generally with synthetic code) please file a bug report. We're working on it!

#### Updates

Sukant Hajra has more energy than I do and finally got to the bottom of the `-Xlint` and `-Ywarn-all` flags and found that although they are not the same from version to version, they are identical, which is probably a copy/paste bug. Here's the relevant code from [2.9.3](https://github.com/scala/scala/blob/v2.9.3/src/compiler/scala/tools/nsc/settings/Warnings.scala#L22-L46) and [2.10.3](https://github.com/scala/scala/blob/v2.10.3/src/compiler/scala/tools/nsc/settings/Warnings.scala#L18-L44) if you want to see for yourself. 

Brian McKenna points out that "only good things happen with `-Xfuture`" so it's now on the list above. Thanks for the tip.

