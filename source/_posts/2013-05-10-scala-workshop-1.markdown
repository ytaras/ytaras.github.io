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
  def twoNumbers = floatingPointNumber ~ floatingPointNumber
}
```

[1]: http://en.wikipedia.org/wiki/Extended_Backus%E2%80%93Naur_Form
[2]: http://www.antlr.org/
[3]: http://tunnelvisionlabs.com/products/demo/antlrworks
[4]: http://legacy.cs.uu.nl/daan/parsec.html
[5]: http://martinfowler.com/bliki/DomainSpecificLanguage.html
[6]: http://henkelmann.eu/2011/01/13/an_introduction_to_scala_parser_combinators
[7]: http://book.realworldhaskell.org/read/code-case-study-parsing-a-binary-data-format.html
[8]: http://book.realworldhaskell.org/read/using-parsec.html
