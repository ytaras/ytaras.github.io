---
layout: post
title: "Scala Workshop - day 0"
date: 2013-04-29
comments: true
categories: 
---
I love Scala. It's combines goodies from JVM and Java infrastructure,
Haskell and dynamic languages like Ruby - and it almost doesn't inherit
Java programming language deceases.  

Anyway, I always wanted to create some end-to-end project using Scala
technologies - but never did as  I'm working with completely different
technology set. So, I decided to come up with some fake business need
and implement it. I'm going to use interesting libraries like Akka and
Scalaz in that task, God help me! I'm going to describe things step by
step so people with no experience in Scala can pick it up and proceed
with me.  

Here are things I want to try in this project:

*   Scala parser combinators
*   Explicit error handling with Scalaz
*   MongoDB with Casbah
*   Functional dependency injection via implicit parameters
*   Properties checking with ScalaCheck
*   Concurrency with Akka actors
*   RIA with Play2 or Lift or JS%2BScalatra - yet to decide

I'm also going to focus more on functional side of Scala as I see it.  
<!-- more -->
## Goal

I've come with following fake 'business idea': we'll be building a
service, which provides workflow capabilities. You should be able to
submit workflow definitions in textual format and after that create some
objects, assign them to workflows and traverse them from step to step. I
foresee a lot of users working with that service, so I want to do that
as scalable as I can. I will call that stuff WAAS -
Workflow-as-a-Service, just because W-buzzword isn't taken yet. 

You see what I'm doing here - I just came up with a task which requires
Web, parsing, scalability and persistence. Hopefully it will help me and
you to put different pieces of Scala infrastructure together.  
## What you should know

You should be familiar with Scala syntax - at least not scared with it.
I'm not going to explain, what a 'trait', 'val' or 'def' keyword means.
If you want to read some introductory stuff - I can recommend  [Scala
for Java refugees][1] or [Another tour of Scala][2]. 

## Starting

If you are familiar with Scala infrastructure, you may skip the rest of
this post - I will talk a bit about SBT and Specs2 here. 

So, de-facto standard in building Scala project is [Simple Build
Tool][3]. Go ahead and install it using site instructions. Also I
recommend to install JDK 7 - either OpenJDK or Oracle one. Once you've
done, you should be able to launch sbt:

```
$ sbt
Detected sbt version 0.12.2
Starting sbt: invoke with -help for other options
/home/ytaras/projects/scala/waas_blog doesn't appear to be an sbt
project.
If you want to start sbt anyway, run:
  /home/ytaras/bin/sbt -sbt-create
``` 
    
Nice! It says we don't have a project definition here, so we should go
ahead and create one:

```
$ mkdir project                                                                                                                                 
$ cat > project.sbt
scalaVersion := "2.10.1"

resolvers %2B%2B= Seq("snapshots" at
"http://oss.sonatype.org/content/repositories/snapshots",
                  "releases"  at
"http://oss.sonatype.org/content/repositories/releases")
^C
```

Now you should be able to run `sbt console` and get Scala console
running. It can take some time as it downloads Scala 2.10.1 binaries.   
So far we didn't do anything interesting - we just specified Scala
version to use and added few OSS repositories to be able to access most
Scala libs in future. Now lets try some TDD cycle. 

After trying Scala test frameworks my personal faworite is Specs2
mutable specifications. Even if it is said that default BDD specs should
be preferred over mutable ones, it goes much closer to my JUnit and
RSpec experience and also has nice integration with some other libs. If
you think I'm wrong and I should use something else, please use the
GitHub link I will give in the end of this post and send me a pull
request with specs written with your favorite framework - or at least
write down your comments here. 

So, lets add Specs2 dependency to our project file:

```scala
  libraryDependencies ++= Seq(
      "org.specs2" %% "specs2" % "1.14" % "test"
  )
``` 

If you are familiar with Maven or Apache Ivy, you can get whats going on
here - you're specifying a group, lib and version and put that into
'test' configuration. Notice that double % is not a typo - it's a SBT
hack to overcome Scala's binary incompatibility between versions. Long
story short, every Scala lib tends to have compiled versions for every
recent Scala version and %% operator picks one based on the current
Scala you are running on.

If you have SBT console (that one, which is opened with `sbt` command,
not a Scala console, which is issued by `sbt console`), you can hit
`reload` there to pick up new project definition. If not - you can just
open new SBT shell and will be done for you. Now try typing `test` into
SBT console - it will download binaries if needed, try to compile
non-existing product and test files and will gracefully shut down with
message about no found tests. Nice! This is your first green build. It
is said, that ideal code is no code, so now you have ideal project done
:)  

Now let's start continuous testing - run `~ test` in SBT shell. It will
monitor your source files and relaunch test suite once you saved your
code. Let's add first test file `src/test/scala/CalculatorSpec.scala`

```scala
import org.specs2.mutable._

class CalculatorSpec extends Specification {
}
``` 

Nothing special is going on here - adding necessary imports and
extending specific class. Let's add a first specification. Scala
features made possible fluent specs declarations similar to RSpec:

```scala
  "Native scala operations" should {
    "add" in { 2 + 2 must_== 4 }
    "subtract" in { 5 - 4 must_== 1 }
    "mutliply" in { 3 * 4 must_== 12 }
    "divide" in { 10 / 2 must_== 5 }
  }
```

When you save file you should notice pretty-printed specs with success
message. As a side-note, what we done here is a useful TDD technical
which I call 'wrapping 3rdparty object' (if you know widely adopted
name, please let me know). Idea here is that we write unit tests for
3rd-party components, which we don't write and control, to ensure our
expectation matches it's real behavior. I don't say you should test all
standard lib or operators as I do here, but it may help you if you
unsure what some specific method does in 3rd party library - write down
a test, make it pass and submit it to your SCM. If version upgrade will
break your expectations (and possibly your code) you will be notified
immediately. 

But enough theory. Let's create a calculator - it's a common showcase
for TDD.

```scala
  "Calculator" should {
    "add" in { Calculator.add(3, 3) must_== 6 }
    "subtract" in { Calculator.subtract(10, 3) must_== 7 }
    "multiply" in { Calculator.multiply(5, 5) must_== 25 }
    "divide"   in { Calculator.divide(30, 3) must_== 10 }
  }
```
    

That ugly API will be our calculator. If we have a glance at our SBT
shell we'll notice compilation fails. so let's add calculator definition
into `src/main/scala/Calculator.scala`:

```scala
object Calculator {
  def add(x: Int, y: Int): Int = ???
  def subtract(x: Int, y: Int): Int = ???
  def multiply(x: Int, y: Int): Int = ???
  def divide(x: Int, y: Int): Int = ???
}
```

I really place all that ??? in my code, that's a Scala construct. Long
story short, it's a placeholder for your implementation which has
whatever type you specify but throws exception in runtime  
Scala has very powerful type systems - comparable to Haskell one's - so
usually you make use of it and make sure a lot of errors are caught by
compiler. Some techniques exists for that and I will mention them in
future posts. Anyway, with TDD we want to do one step at a time, so ???
placeholder allows to break Red-Green-Refactor cycle into
Red-Compiles-Green-Refactor. Using that in adding 2 integers doesn't
make too much sense, but if we're playing with more complex types
(especially monadic ones) this technique becomes useful.  
If you have a look at SBT console, you'll notice we moved to Compiles
stage - tests are executed, but exception is thrown. Nice. Let's use
'obvious implementation' and add, well, obvious implementation for all 4
methods:

```scala 
object Calculator {
  def add(x: Int, y: Int) = x + y
  def subtract(x: Int, y: Int) = x - y
  def multiply(x: Int, y: Int) = x * y
  def divide(x: Int, y: Int) = x / y
}
```

I no longer need explicit return types as they are easily inferred out
by a compiler. And... Tadam! We're Green - and looks like there's no
need for refactoring.  

Anyway, currently are testing only against specific values. Is there a
way to improve a test coverage, add more sample values? That's what's
[ScalaCheck][4] framework is for. Let's try to learn it by example.  
First, add it to your dependencies:

```scala
libraryDependencies ++= Seq(
    "org.scalacheck" %% "scalacheck" % "1.10.1" % "test",
    "org.specs2" %% "specs2" % "1.14" % "test"
)
```
    
Specs2 provides nice integration with ScalaCheck, we have to import it
and mixin to our spec, so we are able to use helpers in our specs:

```scala
import org.specs2.ScalaCheck

class CalculatorSpec extends Specification with ScalaCheck {
```

Good. Let's write our first property. What about following -
Calculator.sum should return sum of two numbers it receives as
arguments:

```scala 
  "Calculator properties" should {
    "add" in prop { (x: Int, y: Int) =>
      Calculator.add(x, y) must_== x %2B y
    }
  }
```

If we save a file now, we'll notice that result run has 108 expectations
- this is because ScalaCheck generated 100 pairs of ints and verified if
  satisfies specified property. Thanks to explicit's black magic we can
use either boolean comparisons `x == y` or Specs2 matchers `x must_== y`
whichever suits better. Notice - we still haven't discovered integer
overflow here, just because we didn't write correct assertions, for
example that sum of positives is positive; but framework silently
generated input values which could help us discover that. But, as we're
just learning, let's go to something more obvious - for example,
division: 

```scala
    "divide" in prop { (x: Int, y: Int) =>
      Calculator.divide(x, y) must_== x / y
    }
```

If we try to run this, it will fail because of zero-division. There are
different ways of handling this. Let's imagine that we say 'we never
pass 0 as second argument to division, so we don't care what happens
there'. It can be easily done with ScalaCheck:

```scala
    "divide" in prop { (x: Int, y: Int) => (y != 0) ==> (
      Calculator.divide(x, y) must_== x / y
    )}
```

If you are scared by lot of ascii symbols and extra parenthesis - I
encourage you to read ScalaCheck user guide, which is a really nice
description how to write properties. If not - just believe me that this
property can be read as _for any pair of integer such that second
integer is not zero, following property is true_

That was rather a big post. Hopefully, you have some insight on Scala
infrastructure and building. You can try it on your own, or have a look
at [GitHub repository][5]. Next time lets do our WAAS!

 [1]:
http://www.codecommit.com/blog/scala/roundup-scala-for-java-refugees
 [2]: http://www.naildrivin5.com/scalatour
 [3]: http://www.scala-sbt.org/
 [4]: https://github.com/rickynils/scalacheck
 [5]: https://github.com/ytaras/scala_waas/commits/day0  
