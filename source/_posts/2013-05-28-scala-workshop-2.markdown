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
In this post I will do some refactoring of existing code, but first we're going
to implement validation of data model. To achieve that I will show few interesting
tricks. So why don't we get started with `Maybe` type?
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
Doing that is a pain - especially if you know there's better ways to do that. So,
if you want to remain conscious and continue writing Java - stop reading at that
point.

Not every modern language have `null` value. In fact, Tony Hoare called null invention
[a billion dollar mistake][] - because it is prone and not solid solution.
For example, Haskell language doesn't have null value by design - and that
eliminates whole kind of errors completely.

To represent a value which may be not present, Haskell has special type
called `Maybe`. It is so much more convenient and safe of null value,
that Scala has it's own `Maybe` in a standard library, but it's called `Option`.

`Option` is a generic type with one type parameter, so if your value may not exist,
you just make it of type `Option[A]`. For example:
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
there's better option - you can use `map` method, which wraps non-existence checks:
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


[a billion dollar mistake]: http://lambda-the-ultimate.org/node/3186
