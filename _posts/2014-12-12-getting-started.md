---
layout: post
title: Setting up to write some Scala
tags: scala
---

Getting set up to play around with Scala is way harder than it should be, due in no small part to the [instructions](http://scala-lang.org/download/) on [scala-lang.org](http://scala-lang.org) which suggest the following possible courses of action, all non-optimal:

- **Downloading Scala** is unnecessary and actively confusing. An important aspect of Scala development is that the language itself is a dependency of your project, and it is very common to be working on projects that use different versions.
- **Typesafe Activator** is not useful. I'm not even going to link to it. It makes an already confusing process even more confusing by adding a 240MB UI that obscures the development process you are trying to learn, and leaves you with "starter" projects that are all crapped up with stuff you do not need.
- **Downloading an IDE** is also a path to confusion and suffering. You *must* treat your commandline build as the ultimate reality, and an IDE-first approach puts the cart before the horse. Or however that metaphor is supposed to work.

## Prerequisites

First things first. I'm just going to assume you're using a Mac or Linux. If you're on Windows, install [Cygwin](http://www.cygwin.com/) and pretend you're not on Windows anymore. It may help to put a bucket over your head.

You need a **Java JRE** 1.6 or newer, which you can download [here](http://www.java.com) if you don't already have it. Be sure not to install the crapware that Oracle still insists on bundling. Once it's installed be sure `JAVA_HOME` is set correctly, so you can open a new shell and do this:

```
$ $JAVA_HOME/bin/java -version
java version "1.6.0_65"
Java(TM) SE Runtime Environment (build 1.6.0_65-b14-462-11M4609)
Java HotSpot(TM) 64-Bit Server VM (build 20.65-b04-462, mixed mode)
```

You will also need a **decent text editor**. Since you're probably a programmer already I'm sure you have a favorite. I recommend just installing syntax highlighting for Scala to begin with, and adding fancier stuff later as you gain an understanding of what's going on. I use [Sublime Text](http://www.sublimetext.com/) with nothing but syntax highlighting and it's just fine.

## Get sbt

For better or worse [**sbt**](http://www.scala-sbt.org/) is the accepted way to build Scala programs. It's not the easiest build system to understand, but for simple projects it's pretty easy to get up and running. And since you have no real choice we'll just move on.

**Don't follow the installation instructions on the website.** Instead go to [sbt-extras](https://github.com/paulp/sbt-extras) and follow the instructions there to install a nice sbt startup script. At this point you have two options:

1. Install it in some central location on your path like `/usr/local/bin`, or
2. Put it at the root of each project and check it into source control like anything else.

There are smart people who choose (1) and others who choose (2) and some will even argue loudly about which is better. I don't care. Flip a coin.

## A Wee Project

Ok so let's set up a project. The idea here is that we have a `build.sbt` that describes the project, and a conventional directory structure so we don't have to describe the project in much detail. All of this is infinitely customizable of course but most projects do fine with the simple setup.

Make a **new directory** called `banjo` or something. It doesn't actually matter but it's common for this to match the project name. Now `cd` into that directory; everything we do from now on happens here.

Create `build.sbt` with something like the following content. The order doesn't matter but **yes you need blank lines** between settings. This is a pretty minimal `build.sbt` file that provides enough information to identify the project and name the build artifact.

```scala
name := "banjo"

organization := "org.chickenpants"

version := "0.1-SNAPSHOT"

scalaVersion := "2.11.4" // Scala is a dependency (!)
```

Create `project/build.properties` with the following content. This is weird, sorry, but this is how to pick an sbt version (the `sbt` script you installed is really just a bootloader that downloads whichever version you specify here).

```
sbt.version=0.13.5
```

Now create the directory `src/main/scala`. This is one of the places sbt will look for Scala source files. 

To review, our file structure now looks like this:

```
banjo/
  build.sbt
  project/
    build.properties
  src/
    main/
      scala/
```

Ok light a candle and type `sbt`. With some luck it will download the internet and then dump you at a prompt (this downloading only happens once; everything is cached globally in `~/.ivy2/`). 

## Write You a Scala

Ok let's write us a Scala. Down in `src/main/scala/` which you created moments ago, add the file `Cookout.scala` with the following content:

```scala
object Cookout extends App {
  println("snausages")  
}
```

And at the sbt prompt type `run`. You will find to your squealing delight that the program runs. Some other things you might want to do at the sbt prompt are:

- `compile` which compiles everything it can find.
- `clean` to remove all the build products.
- `~compile` to compile every time a source file changes.
- `console` to enter the Scala REPL, with all your code and dependencies available.

## Learn Some More

Congratulations, you now have a **minimal but functional** development environment. There is a ton more to learn of course, but at least you have some working tools. Here are some suggestions:

- Join the `#scala` IRC channel on FreeNode. It's a very friendly group of people who are happy to help anyone who is willing to listen and learn.
- Buy [Programming Scala, 2e](http://shop.oreilly.com/product/0636920033073.do) to learn the Scala language, and [Functional Programming in Scala](http://manning.com/bjarnason/) to learn how to write beautiful programs.
- Take Martin's [Functional Programming Principles in Scala](https://www.coursera.org/course/progfun) course on Coursera.
- Check out [Meetup](http://www.meetup.com/) and see if there's a Scala group in the area. You might be surprised!




