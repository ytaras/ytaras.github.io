---
layout: post
title: "Scala Workshop - day 2, refactoring"
date: 2013-05-28 21:56
comments: true
categories:
 - Scala
 - WAAS
 - functional programming
 - monads
---
In this post I will do some refactoring of existing code, but first
we're going to implement validation of data model. To achieve that I
will show few interesting tricks. So why don't we get started with
`Maybe` type?

<!-- more -->

Moving nullability to type system
---------------------------------
How many times you did check for nulls in your code? Something like this:

```scala
val user = UserRepository.load(id)
var name = if (user != null) {
  name = user.name
} else ""
```
Doing that is a pain - especially if you know there's better ways to
do that. So, if you want to remain conscious and continue writing
Java - stop reading at that point.

Not every modern language have `null` value. In fact, Tony Hoare
called null invention [a billion dollar mistake][] - because it is
prone and not solid solution. For example, Haskell language doesn't
have null value by design - and that eliminates whole kind of errors
completely.

To represent a value which may be not present, Haskell has special type
called `Maybe`. It is so much more convenient and safe of null value,
that Scala has it's own `Maybe` in a standard library, but it's called `Option`.

`Option` is a generic type with one type parameter, so if your value
may not exist, you just make it of type `Option[A]`. For example:
```scala
val maybeUser: Option[User] = UserRepository.load(id)
```

`Option` is an abstract type, and there are two concrete classes:
`Some` which hold some result and `None` which represents non-existent value:
```scala UserRepository.scala
def load(id: Int) = {
  if (userTable.contains(id)) {
    // We know for sure that loaded is not null
    val loaded = userTable.findOne(id)
    Some(loaded)
  } else None
}
```
There are static guaranties that `Option` is either `Some` or `None`,
tertium non datum. Have a look at `sealed` keyword if you want to know
how that's done.

You can check if value is null explicitly with pattern matching, but
there's better option - you can use `map` method, which wraps
non-existence checks:

```scala
val maybeName: Option[String] = maybeUser map { _.name }
// Check Scala method calls syntax sugar
// and lambda syntax if you don't understand the code
```

Now `maybeName` contains user's name if there was any user in `maybeUser`. You
can also convert `Option[String]` to `String` by providing default value:
```scala
val nameOrEmpty: String = maybeName getOrElse ""
```
Let's compare put it in one call and compare to null-check:
```scala
val user1 = UserRepository.loadNull(id)
val name1 = if (user != null) {
  name = user.name
} else ""

val user2 = UserRepository.loadMaybe(id)
val name2 = user2 map { _.name } getOrElse ""
```

You may wonder - what's the difference between those two approaches? You
still have to wrap your computation into some sort of check - is it `map` method
or `if` chain. Still, there's a crucial difference even in this tiny example.
`user2` is not of a type `User`, so you can't call a method on it directly:
```scala
user1.name // This is a source of NullPointerException
user2.name // This won't compile - user2 doesn't have name method
```

So, Scala compiler doesn't allow you to write potentially failing operation
and forces you to think about nullability. Unfortunately, you can still have
`NullPointerException` in your code, because Scala still runs on JVM and has to
have null value:
```scala
val dangerousValue: Option[User] = Some(null)
dangerousValue map { _.name } // NPE :(
```
That's why if you see someone is wrapping `null` into `Some`, you should kill him
and save a mankind - those are two different models of representing non-existent
values and you should not mix them never ever.

There's more about `Option` type - I could talk about `flatMap` and
for-comprehensions syntax, but I prefer to go on with more practical things and
introduce those features once we will need them.

Validations
-----------

Currently we are able to pass workflow definitions which have valid
 syntax but are
 not valid workflow definitions. For example, we can define workflow
without start step. Let's target that.


Which different scenarios of invalid workflow definitions can we
 imagine? Probably workflow without steps is invalid, also workflow
 where there are no start steps or more that one start step is invalid
 as well. Also we can try to detect steps which are not accessible
 from other steps - in scope
 of this post we'll use very basic algorithm for that kind of
 validation. We don't know yet how our API will look like, but we can
 aleady write down
 those thoughts as pending specs.

```scala validator_spec.scala
class ValidationSpec extends Specification {
  "Workflow validator" should {
     "fail workflow without start stage" in { pending }
     "fail workflow with few start stages" in { pending }
     "fail wokrflow with orphan stages" in { pending }
     "pass successfull workflow" in { pending }
  }
}
```

In output of spec run we will see our scenarios marked as pending -
that will help us to stay focused. But before we flesh out test for
that large
complex validator, let's create a smaller validator which targets only
one validation
and then we'll see how we can compose them into larger one.

For example, I will take first scenario and think about validation of
steps presence. What signature would you use?

```scala
  def hasAnySteps(workflow: Workflow): Boolean = ???
```
First idea you probably had was to have validator that returns boolean
value indicating if workflow is valid. Anyway, it will be not very
convenient to compose as return value does not contain error message
and if we write a big validator using this one, we'll have to
transform false value into String message. It would be better if
return value would contain error message. Maybe we can resort to
exception handling?

```scala
   @throws(classOf[WidgetValidationException])
   def validateHasSteps(workflow: Workflow): Unit = ???
```

Now we can catch an exception and extract message from it. Anyway,
composing methods with exception is a pain. Instead let's return
validation message if we validation fails and empty value if passed
argument is correct:

```scala validation.scala
  def zeroStepsValidation(wf: Workflow): Option[String] = if (wf.steps.isEmpty) Some("Workflow has 0 steps") else None
```
```scala validator_spec.scala
  "zero step validator" should {
     import WorkflowValidator._
     "return None for valid workflow" in {
       zeroStepsValidation(valid) must beNone
     }
     "return Some for invalid workflow" in {
       zeroStepsValidation(Workflow("wf", Nil)) must beSome
     }
  }
```

I have created a tag in `git@github.com:ytaras/scala_waas.git`
repository called `day2-homework1`, which contains a stub and specs
for 3 other validators, so you may want check it out and try
implementing that on yourself. If you're lazy, you can have a look at
`day2-homework1-solved`. Note - it's not a shortest solution possible,
but it's something I hope will be easy to read for you if you still
have little experience with functional programming.


Combining validations
---------------------

Now we want to add combine those validations into one. We may continue
using `Option` approach as we did before, but instead I would like to
introduce another extremely useful and convenient type called
`Validation`, which ships with external library called scalaz.
Standard Scala library contains similar type called `Either`, but
`Validation` has much better namings and convenient DSL. So next our
step is to add dependency in our sbt file:

```scala project.sbt
libraryDependencies ++= Seq(
    "org.scalaz" %% "scalaz-core" % "7.0.0",
    "org.scalacheck" %% "scalacheck" % "1.10.1" % "test",
    "org.specs2" %% "specs2" % "1.14" % "test"
)
```

Now we will add imports to every file where we use Scalaz:
```scala
// Note - in real code use more fine-grained imports that Scalaz._
// that imports whole library
import scalaz._
import Scalaz._
```

`Option` type is our way of saying "I have or don't have a value for
you". With `Validation[B, A]` you can say "I either have a value of
type A or an error of type explaining what's wrong". With that value
we can imagine somewhat redundant overall validation function in our
case:

```scala
def validate(workflow: Workflow): Validation[List[String], Workflow] = ???
```

In case of successful validation we receive `Success(Workflow)`, but
if there are validation errors, we receive `Error(List[String]))`
which contains list of validation errors. Still, there's a sneak hole
here - we can create `Error([])` which obviously doesn't make sense.
Scalaz provides special type called `NotEmptyList[A]` which is,
surprisingly, list which is guaranteed to contain at least one
element.


Now, if we change return type to `Validation[NotEmptyList[String],
Workglow]` we are sure that validation result is either success or
error with at least one validation explaining what's wrong.


It turns out that that situation is so common that Scalaz provides
special type alias called `ValidationNel[E, A]` which is just a
shorter notation for `Validation[NotEmptyList[E], A]`.


```scala
def validate(workflow: Workflow): ValidationNel[String, Workflow] = ???
```

Testing
-------

Before writing specs I'll create utility class so we are able to work
better with Validation values. I have copied it from scalaz mailing
list:

```scala test/util.scala
package util

import scalaz._
import org.specs2.mutable._
import org.specs2.matcher._

trait ValidationMatcher { self: Specification =>

  /** success matcher for a Validation */
  def beSuccessful[E, A]: Matcher[Validation[E, A]] = (v: Validation[E, A]) => (v.fold(_ => false, _ => true), v+" successful", v+" is not successfull")

  /** failure matcher for a Validation */
  def beAFailure[E, A]: Matcher[Validation[E, A]] = (v: Validation[E, A]) => (v.fold(_ => true, _ => false), v+" is a failure", v+" is not a failure")

  /** success matcher for a Validation with a specific value */
  def succeedWith[E, A](a: =>A) = validationWith[E, A](Success(a))

  /** failure matcher for a Validation with a specific value */
  def failWith[E, A](e: =>E) = validationWith[E, A](Failure(e))

  private def validationWith[E, A](f: =>Validation[E, A]): Matcher[Validation[E, A]] = (v: Validation[E, A]) => {
    val expected = f
    (expected == v, v+" is a "+expected, v+" is not a "+expected)
  }

}
```
Now we can use those matchers in our spec:
```scala
  "Workflow validator" should {
    "fail workflow without any steps" in {
      validate(emptyWorkflow) must failWith("Workflow has 0 steps".wrapNel)
    }
    "fail workflow without start step" in {
      validate(noStartWorkflow) must
        failWith("Workflow has 0 start steps".wrapNel)
    }
    "fail workflow with few start steps" in {
      validate(fewStartsWorkflow) must
        failWith("Workflow has more than 1 start step".wrapNel)
    }
    "fail workflow with orphan steps" in {
      validate(orphansWorkflow) must
        failWith("There are steps that can't be reached".wrapNel)
    }
    "pass successfull workflow" in { validate(valid) must succeedWith(valid) }
  }
```
`.wrapNel` method is a shorthand syntax added by Scalaz to _wrap_ a
value into `NonEmptyList`.


How do we implement that? Let's follow with types. We have a number of
validators, which can be expressed as
`List[Workflow => Option[String]]`. We can apply all of them to
`Workflow` resulting into `List[Option[String]]`. Now we can flatten
that list to probably empty `List[String]`. Now we would like to
convert that to `NonEmptyList`, but we have a possibility of failure,
in case if our list is empty, so we have to use
`Option[NonEmptyList[String]]`. From there we can go to
`ValidationNel[String, Workflow]`, converting `None` (no errors) to
`Success[Workflow]` and `Some` to `Failure`.


Here's the working implementation:
```scala validator.scala
  def validate(workflow: Workflow) =
    validators.map { _.apply(workflow) }.flatten.toNel.toFailure(workflow)
```

You may want to read method signatures to ensure it does exactly the same flow we expressed in previous paragraph.

Still, we have errors in our test suite. This is because if, for
instance, we have 0 steps, we have 0 start steps as well. There are
few ways to fix that - provide smarter matcher or throw only one error
at time, but I will simply replace expectations with correct values:

```scala validator_spec.scala
    "fail workflow without any steps" in {
      validate(emptyWorkflow) must
        failWith(nels("Workflow has 0 steps", "Workflow has 0 start steps"))
    }
    "fail workflow without start step" in {
      validate(noStartWorkflow) must
        failWith(nels("Workflow has 0 start steps",
          "There are steps that can't be reached"))
    }
```
To use `nels` helper method, you need to mix trait
`NonEmptyListFunctions` to your spec.


Now you can make your validators private and remove specs for them -
only method which is going to be used in your API is `validate`. Also
I would like to use some Scalaz boolean syntax goodies to shorten
individual validators code:

```scala validator.scala
  private def zeroStepsValidation(wf: Workflow) =
    wf.steps.isEmpty.option("Workflow has 0 steps")
```
Go ahead and refactor other methods in same fashion.

Parser result
-------------

We remember that our parser returns `ParserResult`, which can be
 replaced with `Validation` for consistency.

[a billion dollar mistake]: http://lambda-the-ultimate.org/node/3186
