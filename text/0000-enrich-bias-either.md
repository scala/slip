# SLIP-NNN - Implicit enrichment of `scala.util.Either` to support monadic bias

**By: Steve Waldman**

This document proposes backwards the addition of implicit enrichment 
of `scala.util.Either` that address shortcomings of that class for its core
use-case as a monadic container for failable operations that carry information
about failures through a sequence of operations.

## History

| Date          | Version       |
|---------------|---------------|
| Sep 19th 2015 | Initial Draft |

## Motivation

`scala.util.Either` is a two-sided disjunction type that could be used for many
purposes. But among the most basic uses for such a type, and the one most commonly
associated with `Either`, is to serve as a more informative alternative for `Option`
or the maybe monad, enabling sequences of potentially failable operations to be
performed or fail cleanly, without the hard-to-reason-about control flow of Exceptions,
but with more information that a mute `None` on failure. 

In order to support this use-case, `Either` currently supports a projection API. 
Left- or more conventionally right-projections support Scala idiomatic `flatMap` 
and `map` methods, permitting them to be used in `for` comprehensions. Unfortunately
the existing API offers a not-at-all idiomatic `filter` method, whose return value is
an `Option` rather than a filtered `Either`. This is understandable, since it is not
immediately obvious what a "filtered `Either`" should be. But it is nevertheless badly
broken. `Either` projections are incompatible with for comprehensions that attempt to
perform pattern matches, use conditional guards, or assign variable. See also [SI-7222](https://issues.scala-lang.org/browse/SI-7222) 
and [SI-5589](https://issues.scala-lang.org/browse/SI-5589).

This document proposes the addition of implicit enrichments that allow API users
to specify a *bias* for their Eithers. A conventionally right-biased `Either`
supports a 
[full suite of methods](http://www.mchange.com/work/enrich-bias-either/enrich-bias-either-2015-09-19/index.html#scala.util.Either$$RightBias$)
analogous to those offered by `Option`, with `map` and `flatMap` flowing only through
`Right` values and terminating with informative `Left` values on failure. This proposal
retains the current symmetry of `Either`, and would not enforce the conventional right-biasing.
Users may choose to import an equivalent, mirror-image 
[left bias](http://www.mchange.com/work/enrich-bias-either/enrich-bias-either-2015-09-19/index.html#scala.util.Either$$LeftBias$)
if they so choose.

The remarkable flexibility of Scala implicits permits an elegant resolution of the 
probem that forced `Either`'s original projection APIs to be broken. In order to
support the semantics of `filter` / `withFilter`, a token is required to represent
an empty `Either`. That token should not be a value itself, but a distinct signifier
that no value is available, like `Option`'s `None`. For an `Either` that contains good values
at `Right`, non-value descriptor should
naturally be represented by some `Left`. But at the library level, `Either` cannot know even what
type `Left` will be parameterized to contain. The library can hardly foresee what an appropriate token for
empty would be. 

To address this, we propose to define biases as objects, each of whose instances
specifies its own empty token. The syntax required to support this is minimal:

```scala
// will impart a right-bias suitable to Either[String,B]
val RightBias = Either.RightBias.withEmptyToken("EMPTY")
import RightBias._
```
Thereafter, any values whose compile-time type is `Either[String,B]` for any `B`
will be usable in for comprehensions directly, with all features supported.

```scala
val a : Either[String,Int] = Right(1)
val b : Either[String,Int] = Right(99)

for( v <- a; w <- b ) yield v+w          // Right(100)
for( v <- a; w <- b if v > 10) yield v+w // Left(EMPTY)
```


### Conveniences

#### Package-wide conventions

Many developers adopt conventions for managing failures which they use frequently
and consistently within projects. In order to help developers [avoid the import tax]
(http://jsuereth.com/scala/2011/02/18/2011-implicits-without-tax.html), it is proposed
that RightBias/LeftBias be defined as traits that can be extended, most usefully by
package objects, to enable defining of project-wide conventions with very little syntax.

```scala
import scala.util.Either;

package object starship extends Either.RightBias {
  // everywhere within the package starship, Either[Error, B] will be usable in for comprehensions
  override val EmptyTokenDefinition = Either.RightBias.withEmptyToken(Error.Empty);

  object Error {
     case object Empty extends Error;
     case class WarpDriveFailure( catastrophic : Boolean ) extends Error;
     // etc.
  }
  sealed trait Error;
}
```

#### Fully operational for comprehensions for lazy developers

For `Either` to be a proper monad, there must be some empty token defined. But (wisely or not),
often developers don't care so much about functinal correctness, and simply want access to the
full power of Scala's for comprehension with a minimum of hassle or syntax. So, it is proposed
that the new API support default biases with no empty token defined, which simply throws a
`NoSuchElementException` if an operation would filter a value to empty. Enabling this would be
a one-liner:

```scala
import Either.RightBias._
```

Thereafter, any `Either` value can be used in a for comprehenion with no further ceremony,
with `map` and `flatMap` operating through `Right` values.

```scala
val a : Either[String,Int] = Right(1)
val b : Either[String,Int] = Right(99)

for( v <- a; w <- b ) yield v+w          // Right(100)
for( v <- a; w <- b if v > 10) yield v+w // throws NoSuchElementException
```

Throwing an Exception in this case is not ideal, but nor should it be surprising to developers,
as there is no natural value for `w` if no empty token is specified. This functionality
could be omitted to encourage good behavior, but many (probably most) uses of `Either` in
a for comprehension do not include filter operations that might fail, and it strikes me
as useful and pragmatic to offer developers an easy way to make `Either` fully operational 
in for comprehensions with minimal work or thought.

#### Clean syntax

Convenience is largely a matter of taste, but one might argue that the proposed
API would represent an improvement over the existing API on that dimension.
The existing projection API also introduces unnecessary syntactic noise into the most common
use-case for `Either`. To every operation in a series of failable operations that
return `Either` must be added the suffix `.left` or `.right` to extract the appropriate
projection. This is a minor nuisance, and would not be sufficient on its own to motivate new
API in my view. But the current proposal, while doing the important work of fixing 
the broken monadic API, also yields a better, cleaner API in this admittedly minor
way.

## Motivating Examples

Let's examine an example of the core use-case for biased `Either`, performing a series of potentially
failable operations in a for comprehension, yielding a value if the full series succeeded, and a token
representing failure if it did not.

Consider a hypothetical process that must read some binary data from a URL, parse the data into 
a `Record`, ensure that record's year is no older than 2011, and then operate upon the data via 
one of several external services, depending on the `opCode`, ultimately producing a `Result`. 
Things that can go wrong include network errors on loading, parsing problems, failure of the 
timestamp condition, and failures with the external service.

With `Option`, one might define a process like:

```scala
case class Record( opCode : Int, data : Seq[Byte], year : Int )
trait Result

def loadURL( url : String )                   : Option[Seq[Byte]] = ???
def parseRecord( bytes : Seq[Byte] )          : Option[Record]    = ???
def process( opCode : Int, data : Seq[Byte] ) : Option[Result]    = ???

def processURL( url : String ) : Option[Result] = {
  for {
    rawBytes                     <- loadURL( url )
    Record( opCode, data, year ) <- parseRecord( rawBytes )
    result                       <- process( opCode, data ) if year >= 2011
  } yield result
}
```

That's fine, but if anything goes wrong, clients will receive a value `None`, and have 
no specific information about what the problem was. Instead of `Option`, we might try 
to use `Either`, defining error codes for things that can go wrong. Conventionally, 
we'll let good values arrive as `Right` values and embed our error codes in `Left` objects.

```scala
object Error {
  case object Empty extends Error;
  case class IO( message : String, stackTrace : List[StackTraceElement] = Nil ) extends Error;
  case class NoParse( message : String, stackTrace : List[StackTraceElement] = Nil ) extends Error;
  case class BadYear( year : Int ) extends Error;
  case class ProcessorFailure( message : String, stackTrace : List[StackTraceElement] = Nil ) extends Error;
}
sealed trait Error;
```

Now, we might try to use the existing projection API to manage our pipeline.

```scala
case class Record( opCode : Int, data : Seq[Byte], year : Int )
trait Result

def loadURL( url : String )                   : Either[Error,Seq[Byte]] = ??? // fails w/Error.IO
def parseRecord( bytes : Seq[Byte] )          : Either[Error,Record]    = ??? // fails w/Error.NoParse
def process( opCode : Int, data : Seq[Byte] ) : Either[Error,Result]    = ??? // fails w/Error.ProcessorFailure

// we try to use the right projection but the
// for comprehension fails to compile
def processURL( url : String ) : Either[Error,Result] = {
  for {
    rawBytes                     <- loadURL( url ).right
    Record( opCode, data, year ) <- parseRecord( rawBytes ).right
    result                       <- process( opCode, data ).right if year >= 2011
  } yield result
}
```

Unfortunately, this fails to compile, because the existing `scala.util.Either.RightProjection` 
does not support pattern-matching (used to extract `opCode`, `data`, and `year`) or filtering 
(which we perform based on `year`).

Instead, under the proposed API, we would right-bias the Either:

```scala
// right-bias Either
val RightBias = Either.RightBias.withEmptyToken( Error.Empty )
import RightBias._

case class Record( opCode : Int, data : Seq[Byte], year : Int )
trait Result

def loadURL( url : String )                   : Either[Error,Seq[Byte]] = ??? // fails w/Error.IO
def parseRecord( bytes : Seq[Byte] )          : Either[Error,Record]    = ??? // fails w/Error.NoParse
def process( opCode : Int, data : Seq[Byte] ) : Either[Error,Result]    = ??? // fails w/Error.ProcessorFailure

// use Either directly, rather than a projection, in the for comprehension
def processURL( url : String ) : Either[Error,Result] = {
  for {
    rawBytes                     <- loadURL( url )
    Record( opCode, data, year ) <- parseRecord( rawBytes )
    result                       <- process( opCode, data ) if year >= 2011
  } yield result
}
```

This does compile, is almost good, but there is a subtle problem. If the year is bad,
the result of `processURL` will be `Error.Empty` rather than the more appropriate `Error.BadYear`.
To address the fact that an empty `Either` will have context dependent meaning, a `replaceIfEmpty`
method is proposed for biased `Either` instances:

```scala
// right-bias Either
val RightBias = Either.RightBias.withEmptyToken( Error.Empty )
import RightBias._

case class Record( opCode : Int, data : Seq[Byte], year : Int )
trait Result

def loadURL( url : String )                   : Either[Error,Seq[Byte]] = ??? // fails w/Error.IO
def parseRecord( bytes : Seq[Byte] )          : Either[Error,Record]    = ??? // fails w/Error.NoParse
def process( opCode : Int, data : Seq[Byte] ) : Either[Error,Result]    = ??? // fails w/Error.ProcessorFailure

// use Either directly, rather than a projection, in the for comprehension
def processURL( url : String ) : Either[Error,Result] = {
  val mbEmpty = for {
    rawBytes                     <- loadURL( url )
    Record( opCode, data, year ) <- parseRecord( rawBytes )
    result                       <- process( opCode, data ) if year >= 2011 
  } yield result
  mbEmpty.replaceIfEmpty( Error.BadYear( -1 ) ) //oops! -1 is not a great value here
}
```

This works fine. But it highlights a shortcoming in the existing implementation: 
There is no way for `replaceIfEmpty` to be embedded within the for comprehension, so that
construction of the `Error.BadYear` instance can have access to the value of `year`. 

This shortcoming should be addressed, if it can be. Unfortunately, I have been unable
to come up with an elegant solution that preserves the spirit of monadic operations (we
don't mutate some variable from which `year` might be recovered) but allows us to recover
local variables from the generated nested flatMap. I'm hoping there is an elegant 
solution that I have failed to consider.

## Drawbacks

The main drawback is the one cited above. The whole purpose of using `Either` rather
than `Option` is so that error values can include information. Under the solution
proposed, however, values that result from a filter operation or pattern match failing
can report only an opaque token, with no further information. When the cause of the
error is fully described by the context in which it occurs at compile time, programmers
can remedy this, either by using the `replaceIfEmpty` method as decribed above, or
by defining a meaningfully empty token and restricting the scope of the bias to the
context in which that meaning would hold.

However, it is not obvious how one can implement incorporation of information
known only at runtime that causes a filter or pattern-match to fail in the generated
error token. This is a serious deficiency.

## Alternatives

The convention of a right-bias could be [hardcoded into Either](https://groups.google.com/forum/?hl=en#!topic/scala-debate/K6wv6KphJK0%5B1-25%5D),
although that does not resolve the issue of what value should be returned when a filter operation fails to a `Right` value, and it violates
the symmetry of `Either`, baking into place what had previously been only a convention. This would break no existing code, however.

[New left and right projection APIs could be defined](https://github.com/robcd/scala.github.com/blob/master/sips/pending/_posts/2012-06-29-fixing-either.md),
so that the result of operations in a projection is a projection directly. This would eliminate some of the boilerplate of the existing API,
but does not in and of itself solve the problem of the empty token definition, and requires deprecating and replacing existing API. This proposal (by Rob Dickens)
does include an interesting suggestion of defining the empty token by requiring an implicit conversion from elements filtered by conditional
guards to an element of the non-value type of Either. However, this requires declaration of an implicit conversion for every type filtered, and the information that one
might wish to embed in the empty token may be defined in an enclosing scope, rather than in the `Either` value on which `withFilter(...)` will
be called, so this solution cannot adequately address the problem of an insufficiently informative empty token. (See the example above. This solution
could not have embedded the variable `year` in an `Error.BadYear` object. Alas.)

Use of some kind of [special case source rewriting](https://github.com/scala/scala/pull/4547#issuecomment-111312231) has been suggested to fix the 
problems with `Either`. It is not obvious to me how this would be implemented, and putting special cases into the compiler
strikes me as a not great approach to fixing broken API.

A final alternative would be to simply not solve this problem. Many other libraries have now implemented their own `Either`-like monadic
disjunction types. Users could be pointed to those libraries, or can roll their own. It's not an especially challenging task. `Either` might
simply be deprecated.

## Design

The main design goals here are:

* No breaking of existing source or need to deprecate and replace existing libraries
* Provision of a better API for those who choose to use it:
  * Full, natural support of for comrehension features
  * Convenient, low-boilerplate syntax
  * Correct monadic behavior
* Good type inference over hierarchies of polymorphic error types

## Implementation

A proposed implementation exists. Either values are implicitly enriched with correct monadic functions,
biased according to users' choice of import (`LeftBias` or `RightBias`). Alternatively, users can define
a bias for Eithers within a class, object, or package by having the enclosing scope implement `LeftBias`
or `RightBias` traits.

The library is intended to be implemented as a set of objects and traits within the `Either` companion
object, so that its functionality is naturally paired with Either. (This is optional: The library
would work fine if placed elsewhere. But `Either` is the natural place for it to live.)

The implementation is nondisruptive of existing code. It is proposed to be incorporated in the
next release of Scala permitting new API.

## Implementation Drawbacks

The current implementation violates DRY principles, for two reasons:

1. As with other Either API, implementation of the new API is repeated in mirror-image
   form for `Left` and `Right` (`LeftBias` and `RightBias` here), as the complexity of trying
   to abstract over Left and Right seems more trouble than it's worth.

2. Although the core logic of `LeftBias` and `RightBias` are each centralized
   in a typeclass, for each left and right there are two separate implementations that
   delegate to the typeclass. In other words, for each of `LeftBias` and `RightBias`, the main
   API is repeated in three places: a type class, an `AbstractOps` class that delegates
   to the typeclass but requires specification of an empty token as a constructor argument,
   and an `Ops` class that does not require any constructor argument and so can be implemented
   as a value class. (The empty token in this case is hardcoded to be a thrown `NoSuchElementException`)
   It is a judgment call whether the performance benefit of having an unboxed value class for a very common
   case outweighs the maintenance cost of the trifurcation of Left and Right implementations into
   a typeclass and two stubs. The three classes could be condensed into one if the benefit of the unboxed
   value class is foregone. 

## Unresolved questions

The main unresolved question is whether there is an elegant way to embed information available only at
runtime about what caused a filter operation to fail into the token representing empty. See "Drawbacks"
above.

## References

1. [Scala 2.12.0 fork with enriched Either implementation][1]
2. [Extensively updated Either API documentation][2]
3. [SI-7222: Pattern match typing fail in for comprehension][3]
4. [SI-5589: For-comprehension on Either.RightProjection with Tuple2 extractor in generator fails to compile][4]
5. [SIP-20 Fixing Either by Rob Dickens][5]
6. [Pull Request: Implicit enrichment as alternative to broken Either projection APIs][6]

[1]: https://github.com/swaldman/scala/tree/enrich-bias-either "Enriched Either implementation"
[2]: http://www.mchange.com/work/enrich-bias-either/enrich-bias-either-2015-09-19/index.html#scala.util.Either "API Documentation"
[3]: https://issues.scala-lang.org/browse/SI-7222 "SI-7222"
[4]: https://issues.scala-lang.org/browse/SI-5589 "SI-5589"
[5]: https://github.com/robcd/scala.github.com/blob/master/sips/pending/_posts/2012-06-29-fixing-either.md "SIP-20"
[6]: https://github.com/scala/scala/pull/4547 "Pull Request"

