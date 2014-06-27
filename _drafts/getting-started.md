---
layout: post
title: Getting Started with Scala
tags: scala
---

Getting set up to play around with Scala is way harder than it should be, due in no small part to the [instructions](http://scala-lang.org/download/) on [scala-lang.org](http://scala-lang.org) which suggest the following possible courses of action, all non-optimal:

- **Downloading Scala** is unnecessary and actively confusing. An important aspect of Scala development is that the language itself is a dependency of your project, and it is very common to be working on projects that use different versions.
- **Typesafe Activator** is not useful. I'm not even going to link to it. It makes an already confusing process even more confusing by adding a 240MB UI that obscures the development process you are trying to learn, and leaves you with "starter" projects that are all crapped up with stuff you do not need.
- **Downloading an IDE** is also a path to confusion and suffering. You *must* treat your commandline build as the ultimate reality, and an IDE-first approach puts the cart before the horse. Or however that metaphor is supposed to work.

## Prerequisites

First things first.

- I'm just going to assume you're using a Mac or Linux. If you're on Windows, install [Cygwin](http://www.cygwin.com/) and pretend you're not on Windows anymore. It may help to put a bucket over your head.
- You need a **Java JRE** 1.6 or newer, which you can download [here](http://www.java.com) if you don't already have it. Be sure not to install the crapware that Oracle still insists on bundling. Once it's installed be sure `JAVA_HOME` is set correctly, so you can open a new shell and do this:

```
$ $JAVA_HOME/bin/java -version
java version "1.6.0_65"
Java(TM) SE Runtime Environment (build 1.6.0_65-b14-462-11M4609)
Java HotSpot(TM) 64-Bit Server VM (build 20.65-b04-462, mixed mode)
```

- You need a **decent text editor**. Since you're probably a programmer already I'm sure you have a favorite. I recommend just installing syntax highlighting for Scala to begin with, and adding fancier stuff later as you gain an understanding of what's going on. I use [Sublime Text](http://www.sublimetext.com/) with nothing but syntax highlighting and it's just fine.

## Install sbt

For better or worse **sbt** is the accepted way to build Scala programs. You may find it comforting to know that sbt is confusing as all hell, and essentially nobody understands it, including me. But for simple projects it's pretty easy to get up and running, and since you have no real choice we'll just move on.

*I should note at this point that there is some disagreement about the next step. Many advise to never install sbt, but instead  include it in each project. This eliminates a potential compatibility issue, but you end up with N+1 copies of various versions of the sbt launcher everywhere. We're going to go ahead and install it, but keep in mind that you will probably run into projects that take the other approach.*

Right. So, on the Mac you can just `brew install sbt`, assuming you have [Homebrew](http://brew.sh/) installed, which you probably do. If this doesn't match your situation, check out the instructions [here](http://www.scala-sbt.org/release/docs/Getting-Started/Setup.html) and get back to me when you can do this:

```
$ cd /tmp
$ mkdir foo
$ cd foo
$ sbt
[info] Set current project to foo (in build file:/private/tmp/foo/)
> quit
```

If you do this anywhere other than a new directory you may find that you now have `target` and `project` subdirectories. You can just delete these.

## A Wee Project

Ok so let's set up a project. The idea here is that we have a `build.sbt` that describes the project, and a conventional directory structure so we don't have to describe the project in much detail. All of this is infinitely customizable of course but most projects do fine with the simple setup.

- Make a **new directory** called `banjo` or something. It doesn't actually matter but it's common for this to match the project name.
- Create `banjo/build.sbt` with something like the following content. The order doesn't matter but **yes you need blank lines** between settings. This is a pretty minimal `build.sbt` file that provides enough information to identify the project and name the build artifact.

```scala
name := "banjo"

organization := "org.chickenpants"

version := "0.1-SNAPSHOT"

scalaVersion := "2.10.3" // Scala is a dependency (!)
```

- Create `banjo/project/build.properties` with the following content. This is weird, sorry, but this is how to pick an sbt version (the sbt you installed is really just a bootloader that downloads whichever version of sbt you specify here).

```
sbt.version=0.13.2
```

- Create the directory `banjo/src/main/scala`. This is one of the places sbt will look for Scala source files, and by far the most commonly used.
- `cd banjo` and then `sbt`. With some luck it will download the internet and then dump you at a prompt. You are now read to

## Write Some Effin Scala

Ok let's write us a Scala. Down in `banjo/src/main/scala/` which you created momemnts ago, add the file `Cookout.scala` with the following content:

```scala
object Cookout extends App {
  println("snausages")  
}
```

And at the sbt prompt type `run`. You will find to your squealing delight that the program runs. Some other things you might want to do at the sbt prompt are:

- `compile` which compiles everything it can find.
- `clean` to remove all the build products.
- `~compile` to compile every time a source file changes. You can put `~` in front of any command, btw.





