---
layout: post
title: "Scala Workshop - day 1, parsers"
date: 2013-05-10 14:27
comments: true
categories: 
 - Scala
 - WAAS
 - functional programming
 - parser combinators
---
Let's continue on WAAS (workflow-as-a-service). In this post I'm going
to show how to use parsers combinators to parse our workflow
definitions. 

Have a look at sample workflow definition file we're going to parse. It
contains 2 worflow definitions - one for issue tracking system and other
for deals. 

```
workflow issue {
  start state open
                 goes to started;
  state started  goes to open, resolved;
  state resolved goes to closed, open;
  state closed;
};

workflow project {
  start state negotiation
               goes to signed, failed;
  state signed goes to failed, done;
  state done   goes to paid, failed;
  state paid;
  state failed;
};
```

At the end of this post we're going to have a parser, which transforms
textual definitions to our internal model, ready for persistense. 

<!-- more -->

## Parsers in ANTLR

To target that task, let's start with something more conventional than
Scala parsers. [EBNF][1] format is widely used for describing grammar, so
we're going to start with that:
``` 
goes       = 'goes', ws, 'to', ws, { identifier, ',', ws }, identifier;
state      = [ 'start', ws ], 'state', ws, goes, ws, ';'
workflow   = 'workflow', ws, identifier, ws, '{', ws, { state, ws },  ws, '}', ws, ';'
workflows  = { workflow } 
ws         = ? whitespace characters ?
identifier = letter { letter | number | '_' }
```

In order to generate executable parsers, we should define the grammar
in some DSL specific for a parser framework we choose. Let's define a
grammar in [ANTLR4][2] - well known parser in Java world:
```
grammar workflowGrammar;
file: workflowDefinition* EOF;
workflowDefinition: 'workflow' ID '{' stateDefinition*?'}' ';';
stateDefinition: 'start'? 'state' ID goes? ';';
goes: 'goes' 'to' (ID',')* ID;
ID: [a-zA-Z]([a-z0-9_])* ;
WS: [ \t\r\n]+ -> skip;
```
Based on that grammar we are ready to generate lexer and parser. For
ones who are not familiar with those terms - lexer is a converter from
input stream to stream of tokens and parser is converter from a stream
of tokens to a parse tree. Anyway, this distinction is not critical in
our case, so I will refer to lexer+parser combination as simply parser.

I leave ANTLR parser generation and execution as an exercise for a reader.
Still, here's sample parsed tree you could view using [ANTLRWorks][3]:
```
(file
	(workflowDefinition workflow issue {
		(stateDefinition start state open (goes goes to started) ;)
		(stateDefinition state started (goes goes to open , resolved) ;)
		(stateDefinition state resolved (goes goes to closed , open) ;)
		(stateDefinition state closed ;)
	 } ;)
	(workflowDefinition workflow project {
		(stateDefinition start state negotiation (goes goes to signed , failed) ;)
		(stateDefinition state signed (goes goes to failed , done) ;)
		(stateDefinition state done (goes goes to paid , failed) ;)
		(stateDefinition state paid ;)
		(stateDefinition state failed ;)
	 } ;)
<EOF>)
```
You can easily see that this tree is much easier to convert to our model
than a flat string we had on our input. 

ANTLR is a great tool and is used in most cases when we need parsing
capabilities in Java world. Anyway, there's a thing you should notice
about grammar definition - it is so called 'external DSL' - it means
that this is a separate language, which needs it's own parser and
compiler. If instead we would try to define grammar using Java directly,
it would be a huge pain to write, read and support. 

## Parser combinators framework

Scala goes with it's own parser framework, very much
inspired by Haskell's [Parsec][4] framework and is a part of standart
Scala library. In a contrast, Scala's parsers are written using plain
Scala using some syntatical features and tricks of a language. This
approach is called [Internal DSL][5]

Idea behind parser combinators is very simple: every parser is defined
as a function from its input to parse result. Parser result contains
output value or failure message and rest of input:
```scala
def parse(input: Input): Result = ???
sealed trait Result
case class Success(result: Output, next: Input) extends Result
case class Failure(failure: String, next: Input) extends Result
```
Returning rest of the input, which wasn't consumed by a parser, turns
out to be a very usefull feature, because it enables easy parser
composition. BTW, composition is kind of fetish in functional
programming community - ability to define small primitives and compose
larger thing of it is recognized as design success. With that in mind,
parser combinators are definitelly a success - all parser tutorials
start with creating a parser which parses a single char and than create
much more expressive parsers on top of that primitive. I won't do that
in this post, instead I will act as a parser library user. Anyway, I
strongly recommend reading great [series of posts on that topic][6]. If
you don't afraid Haskell (and you shouldn't!), there are two chapters on
our topic in 'Real World Haskell' book - 
[Code case study: parsing a binary data format][7] and
[The Parsec parsing library][8]. This is a really interesting stuff to
read - and I give up explaining parser combinators to that guys. 

So let's create a file `src/main/scala/parser/parser.scala` and add
following content to it:
```scala
package parsers

import scala.util.parsing.combinator._

object SampleParser extends JavaTokenParsers {
  def number = floatingPointNumber
  def twoNumbers = number ~ number
}
```

Let's start from the beginning. We import all required imports and then
extend our custom object from `JavaTokenParsers` class. This class is an
extension of default parsers class, `scala.util.parsing.combinator.Parsers`. 
What's useful in that class in our case is that it does whitespaces handling 
and provides few useful parsers, like `ident` or `floatingPointNumber` we're using in this example.

Launch console with `sbt console` and have a look at types:
```
scala> import parsers._
import parsers._

scala> import SampleParser._
import SampleParser._

scala> :type number
parsers.SampleParser.Parser[String]

scala> :type twoNumbers
parsers.SampleParser.Parser[parsers.SampleParser.~[String,String]]
```
Ok, now it should be easier to tell what's going on. Each parser has a type -
what is returned on successful parse. For example, `number` parser
returns (somewhat surprisingly) String. But what's interesting is a second parser,
which is constructed as a combination of 2 parsers and return `~[String, String]` - yet
another type to present data tuple. We can use `twoNumbers` parser on it's own or
construct more parsers on top of it - and this is what composition is about.

After defining parsers we can use them. There are 2 methods - `parse` and `parseAll`.
Difference is that second fails if parser didn't consume all the input. Here are some
examples of using that:
```scala
object ParserExample extends App {
  import SampleParser._

  println("Parsing whole string '123' as a number")
  println(parseAll(number, "123").get)
  println("Parsing whole string '123 321' as a number")
  parseAll(number, "123 123") match {
    case x: NoSuccess => println(x.msg)
  }
  println("Parsing string '123 321' as a number")
  println(parse(number, "123").get)
  println("Parsing whole string '123 321' as two numbers")
  println(parseAll(twoNumbers, "123 321").get)
}
```

Go ahead and run this program - there are 1 failing and 3
successful examples.

So far so good, but how do we convert parser output to types we need? That's what `^^`
method is for. Let's make our first parser to convert result to float and second
one to multiply floats it parses:
```scala
  def number: Parser[Float] = floatingPointNumber ^^ { _.toFloat }
  def twoNumbers: Parser[Float] = number ~ number ^^ { x => 
    x._1 * x._2
  }
```

As shown here, `^^` operator takes parser on a left and a converter function
on a right, and a result is a parser which returns converted result. 

If you are afraid of pseudo-graphic operators, don't be - there's more to come.

### Creating a model

Before we can parse, I advise to create a model for our workflows, so we have a stable ground
to work on. 

```scala model/model.scala
package model

case class Step(name: String, goesTo: List[String], start: Boolean)
case class Workflow(name: String, steps: List[Step])
```

This is just enough to get started.

### Converting grammar

Let's start with creating simple non-converting parser:

```scala parsers/parser.scala
object WorkflowParsers extends JavaTokenParsers {
  def workflows = workflow.*
  def workflow = ("workflow" ~> ident <~ "{") ~ step.* <~ ("}" ~ ";")
  def step = ("start".? <~ "step") ~ ident ~ goesTo.? <~ ";"
  def goesTo = ("goes" ~ "to") ~> (ident <~ ",").* ~ ident
}
```

I promised there will be more pseudo-graphics coming. Let's me explain few new things:

+ `parser.*` means zero-or-many combinator, so created parser will be executed zero or many times. In 
example, workflows file contains zero or more workflow definitions
+ strings are implicitly converted to parsers, so `"workflow"` is actually a parser which accepts 'workflow' string
+ `ident` is a parser, which parses a Java identifier
+ `parser1 ~> parser2` means that both parsers are executed, but only result of right one is stored,
so `"workflow" ~> ident` ensures there's a `workflow` word before an identifier, but result will
store only identifier value. Same for `<~`, which drops the right value and stores left
+ `parser.?` is optional combinator, so `goesTo.?` means there can be or not be `goes to` part
in step definition

With that in mind, you probably should be able to read through parser definition and see
it's the same we expressed in ANTLR4 grammar. Let's load this into console and have a look at types generated for us.

```
scala> :type goesTo
parsers.WorkflowParsers.Parser[parsers.WorkflowParsers.~[List[String],String]]

scala> :type workflows
parsers.WorkflowParsers.Parser[List[parsers.WorkflowParsers.~[String,List[parsers.WorkflowParsers.~[parsers.WorkflowParsers.~[Option[String],String],Option[parsers.WorkflowParsers.~[List[String],String]]]]]]]
```

Wow, if for simple parser `goesTo` it's theoretically readable, for top-level `workflows` parser it quickly gets out of control.
We should use our converter functions to get return result to something more usable. I'm going to start with `goesTo`. First, I'll
explicitly set a type to more desired one:

```scala
def goesTo: Parser[List[String]] =  ("goes" ~ "to") ~> (ident <~ ",").* ~ ident
```

If we try to compile that, we will get a compile error. I want to fix it, but I don't want to
write working implementation right away, as I want to use TDD here. So I came for help to `???`
operator I described earlier:
```scala
  def goesTo: Parser[List[String]] = ("goes" ~ "to") ~> (ident <~ ",").* ~ ident ^^ {
     _ => ???
  }
```

What I'm writing here - I want to convert parse result to `List[String]`, but I'm not sure
about implementation yet, so compile this but throw an error in runtime if someone calls this. 

It's time to write some tests:

```scala parser/WorkflowParserSpec.scala
package parsers

import org.specs2.mutable._
import org.specs2.ScalaCheck
import WorkflowParsers._

class WorkflowParserSpec extends Specification {
  "Parser" should {
    "parse goes to" in { 
       parseAll(goesTo, "goes to one, two, three").get must_== 
         List("one", "two", "three")
    }
  }
}
```

Note `.get` method, which is a sneak around type system to give us ability
to take parse result without working on failure conditions. If parse fails, `.get`
throws exception in runtime, which is good enough for testing. Anyway, in
production code we should use some more consistent approach and I'll go to that in next posts.

If we run a test, we'll receive `implementation is missing` error, which is exactly what we expected. Let's
now flesh out an implementation: 

```scala parser/WorkflowParserSpec.scala
  def goesTo: Parser[List[String]] = ("goes" ~ "to") ~> (ident <~ ",").* ~ ident ^^ {
     case list ~ last => list :+ last
  }
```

We saw our non-converting parser returns `~[List[String], List]` class and it is implemented in a way
it makes possible pattern-matching on that. It makes possible a trick `case list ~ last`, where
`list` and `last` are deconstructed values of parser result. What we need to do is to append
last result to a list - and it's done with `:+` list operator. Run a test and it succeeds. 

If we load a console once more, we'll notice that not only type of `goesTo` parser changes, but
of all the other parsers make on top of that:

```
scala> :type goesTo
parsers.WorkflowParsers.Parser[List[String]]

scala> :type workflows
parsers.WorkflowParsers.Parser[List[parsers.WorkflowParsers.~[String,List[parsers.WorkflowParsers.~[parsers.WorkflowParsers.~[Option[String],String],Option[List[String]]]]]]]
```

### Doing other parser

Let's convert other parsers. 

```scala parser/parser.scala
  def workflows: Parser[List[Workflow]] = workflow.*
  def workflow: Parser[Workflow] = ("workflow" ~> ident <~ "{") ~ step.* <~ ("}" ~ ";") ^^ {
    _ => ???
  }
  def step: Parser[Step] = ("start".? <~ "step") ~ ident ~ goesTo.? <~ ";" ^^ {
     _ => ???
  }
```

And now let's have some tests:

```scala parser/WorkflowParserSpec.scala
    "parse start step" in {
      parseAll(WorkflowParsers.step, "start step name goes to one, two, three;").get must_==
         Step("name", List("one", "two", "three"), true)
    }
    "parse empty step" in {
      parseAll(WorkflowParsers.step, "step name;").get must_==
         Step("name", List(), false)
    }
    "parse empty workflow" in {
      parseAll(workflow, """workflow wf {};""").get must_==
         Workflow("wf", List())
    }
    "parse not-empty workflow" in {
      parseAll(workflow, """workflow wf {
                              start step one goes to two;
                              step two;
                            };""").get must_==
         Workflow("wf", List(Step("one", List("two"), true), Step("two", List(), false)))
    }
```

I know there's a lot duplications and I'm going to do some refactoring in my next post.

I encourage you to go ahead and try to fix those tests by yourself. I have created a tagged revision
so it should be easier for you to setup environment:

```sh
git clone https://github.com/ytaras/scala_waas.git
cd scala_waas
git checkout day1-homework
```

I also submitted a solution for the impatient - have a look at tag `day1-homework-solved`, 
anyway you'll learn better if you try to do the homework yourself.

### Conclusion

This was a long post. Hopefully, it was useful, as I'm going to continue hacking our new and shiny WAAS service. 
I'm looking forward for receiving comments and questions from you - and will meet in the next post.

[1]: http://en.wikipedia.org/wiki/Extended_Backus%E2%80%93Naur_Form
[2]: http://www.antlr.org/
[3]: http://tunnelvisionlabs.com/products/demo/antlrworks
[4]: http://legacy.cs.uu.nl/daan/parsec.html
[5]: http://martinfowler.com/bliki/DomainSpecificLanguage.html
[6]: http://henkelmann.eu/2011/01/13/an_introduction_to_scala_parser_combinators
[7]: http://book.realworldhaskell.org/read/code-case-study-parsing-a-binary-data-format.html
[8]: http://book.realworldhaskell.org/read/using-parsec.html
