# SLIP-NNN - Redesigning Collection Views

**By: Joshua Suereth**

This proposal is an attempt to improve the usefulness and consistency of
scala's "view" API.

## History

| Date          | Version       |
|---------------|---------------|
| Aug 13th 2015 | Initial Draft |

## Introduction to Scala Views

Scala's collection library is one of the most widely used in the entire Scala
ecosystem, and is a primary tool for a lot of computation.  A large part of the
code base revolves around a relatively little used feature: Views.

Scala's views are designed to allow users to efficiently chain together a sequence
of operations over data, and perform them `lazily` regardless of whether the
underlying collection is lazy.  Scala's views are designed to have these aspects:

1. Calling `.view.force` on a collection returns something that is functionally
    equivalent to the original collection, i.e. preserving as much of the
    type as was known before.
2. Calling `.view`, executing a sequence of transformations, and then calling
    `.force` should return a collection that is functionally equivalent to
    just executing the sequence of transformations directly on the original
    collection.
3. Calling `.view` and a sequence of transformations should not immediately
   execute any transformation, but instead defer the operation, e.g. `map`,
   `filter`, `slice`
4. Calling any operation which must be strictly evaluated will
   result in immediate evaluation, e.g. `sum`, `fold`.

In addition to these core principles, Scala's current view library also attempts
to memoize intermediate values of views.  This means that if we call `map` on
a view, and then `take(5)`, the first five `map`ed values will be remembered
by the view for any future operations.

## Problems with the current design.

Let's walk through a few of the problems that exist with the current view
design, which has motivated several of us to each try our own hand at alternatives.

### Problem 1 - Unwanted Memoization

Currently views will sporadically memoize intermediate values.   One of the goals
of the library is to consistently defer computation until it's needed.  However,
sometimes the intermediate state is not something that needs to be remembered.

An example:

```scala
case class User(..., homeCity: String)
val x: Traversable[Record] = DatabaseReader.readTable("foo")
def readUser(r: Record): User = ...
val pittsburghUserCount =
   x.view.map(readUser).filter(_.homeCity == 'Pittsburgh').sum
```

In this scenario, the view is simply used to chain together two simple collection
transformations and one computation.  The view is being used in an attempt to
prevent creating intermediate collection values, thereby improving performance
of the whole computation.   It is not expected that the view will be shared/used
beyond this simple definition.  We expect this to be the primary use case
for views.

While we also expect persistent views of collections to be a valid use case,
we think the library should be tuned for the above use case, while allowing the
second.


### Problem 2 - Mutable collections


Currently, views inconsistently handle mutable collections.  This is, in part,
due to memoization and, in part, due to supporting operations which cannot
be correctly handled on mutable collections.   Let's look at the first issue:

```scala
val x = ArrayBuffer(1,2,3,4,5)
val y = x.view.take(1)
pritnln(x.take(1).mkString("")) // prints: "1"
pritnln(y.mkString("")) // prints: "1"
x.clear()
println(x.take(1).mkString("")) // prints: ""
println(y.mkString("")) // throws IndexOutOfBoundsException
```

This is basically due to the view remembering indicies when the underlying
collection is mutable.   However, the current view implementation also
requires memoizing the underlying collection for certain operations, here's
an example:

```scala
object foo extends Traversable[Int] {
   override def foreach[U](f: Int => U): Unit = {
      System.err.println("Making stuff"); f(1); f(2)
   }
}
val y = foo.view.scanLeft(0)(_+_) // Note: this prints `Making stuff`
y.head // Note: This will never call the original foreach.
```

*Note: This issue is actually due to memoizing indicies in the view implementation.*

### Problem 3 - Maintenance issues

Probably the primary issue with the view lirbary, as it currently exists, is
due to maintenance problems.   Views are meant to mimic the collection API
as closely as possible.  Each operation of the collection API must be
implemented such that it is NOT executed, but instead staged for later
execution (if possible).   To do so, currently views directly extend their
collection counterparts and override all implementations, e.g.

```
trait TraversableViewLike[+A,
                          +Coll,
                          +This <: TraversableView[A, Coll] with TraversableViewLike[A, Coll, This]]
  extends Traversable[A] with TraversableLike[A, This] with ViewMkString[A]
{
   ...
   override protected[this] def newBuilder: Builder[A, This] =
       throw new UnsupportedOperationException(this+".newBuilder")
}
```

Here we see a method which is not meant to be used in the view hierarchy must be
overridden (with a runtime exception), otherwise the default implementation from
our parent would get used.  Because all of Scala's collection libraries provide
as much default implementation as possible, this means that ANY new method
added to the collection library also involves unimplementing the method
in the view library.   This can get particularly hairy as optimizations
which get placed into "lower" collection classes must then be "undone" in the
view hierarchy.   Views are constantly in danger of accidentally inheriting
an operation which they cannot support.

Furthermore, Each new node in the class tree requires extension of the memoized
transformations that preceded it.  Here's an instance from `IndexedSeqViewLike`:

```
trait SeqViewLike[+A,
                  +Coll,
                  +This <: SeqView[A, Coll] with SeqViewLike[A, Coll, This]]
  extends Seq[A] with SeqLike[A, This] with IterableView[A, Coll] with IterableViewLike[A, Coll, This]
{ self =>
  /** Explicit instantiation of the `Transformed` trait to reduce class file size in subclasses. */
  private[collection] abstract class AbstractTransformed[+B] extends Seq[B] with super[IterableViewLike].Transformed[B] with Transformed[B]
  trait Transformed[+B] extends SeqView[B, Coll] with super.Transformed[B] {
    def length: Int
    def apply(idx: Int): B
    override def toString = viewToString
  }  
```

This `Transformed` trait is one of the tricks used to ensure that views can stage
operations and share implementations between each staged result.

This results in bugs like [SI-4190](https://issues.scala-lang.org/browse/SI-4190)
and [SI-4332](https://issues.scala-lang.org/browse/SI-4332)
whereby the parent-behavior of a view is getting used through inheritance, when
in fact we'd prefer a compile time error telling us to implement a missing method.

```
val x: ArrayBuffer[String] = ArrayBuffer("a", "b", "c")
val y: IndexedSeq[String] = x.view map (_ + "0") // Class cast exception at runtime
```


## Abstract

Here we'd like to propose a new view library with the following goals:

* Views should solve the original goals for scala collections, sepcifically:
  1. Calling `.view.force` on a collection returns something that is functionally
      equivalent to the original collection, i.e. preserving as much of the
      type as was known before.
  2. Calling `.view`, executing a sequence of transformations, and then calling
      `.force` should return a collection that is functionally equivalent to
      just executing the sequence of transformations directly on the original
      collection.
  3. Calling `.view` and a sequence of transformations should not immediately
     execute any transformation, but instead defer the operation, e.g. `map`,
     `filter`, `slice`
  4. Calling any operation which must be strictly evaluated will
     result in immediate evaluation, e.g. `sum`, `fold`.
* Views should NOT extend the collection API directly, rather they should
  have their own interface, whereby implementations can be specifically tuned
  for views.
* Views should NOT memoize results by default, but instead allow the User to
  selectively memoize.
* Views are meant to optimise a set of use cases where a chain of
  collection-transforming operations is performed, specifically, where a
  highly customized `fold` could be used to replace the operation, and where
  the standard `Iterator` interface is insufficient or less clear.

The new library should take advantage of the lessons learned in Scala's
current view library.  They should provide a maximum of flexibility,
efficiency without causing maintainability issues.

The library will consist of a new `View` class that simple stores:

1. an underlying collection
2. A transducer stack of staged operations
3. A builder it can use to construct a new collection from the staged
   operations.


### Transducers
For those unfamiliar with transducers we'll do a quick introduction here.
The word Transducer is used to describe a function which operates against
the folding function used when folding a collection.  For example:

```
List(1,2,3).foldLeft(0) { _ + _ }
```

Here, the foldLeft function takes an accucmulator (starting at 0), and a folding
function `{ _ +  _}`.  The folding function is responsible for joining each
collection element to the accumulator, returning the next accumulator.

Generically, we can treat this folding function as having a type:
`(Accumulator, Element) => Accumulator`.

Transducers are functions which manipulate these folding functions.  Here is
an example interface for a Transducer:

```
type Fold[Accumulator, -Element] = (Accumulator, Element) => Accumulator]
trait Transducer[-A, +B] {
 def apply[Accumulator](in: Fold[Accumulator, B]): Fold[Accumulator, A]
}
```

We can define a new transducer which mimics the collection `map` operation
for a fold.

```

case class MapTransducer[A,B](f: A => B) extends Transducer[A,B] {
 override def apply[Accumulator](in: Fold[Accumulator, B]): Fold[Accumulator, A] = {
   (acc, el) => in(acc, f(el))
 }
}
```

This implementation simple creates a new folding function which will first
transform any element it receives using the mapping function `f` before
delegating down to the original folding function.

Here's an example for how to use this transducer:

```
def count(acc: Int, el: Int) = acc + el
val stringLengthTd = MapTransducer[String, Int](_.length)
val countChars = stringLengthTd(count _)
List("Hello", "World").foldLeft(0)(countChars) // 10
```

Transducers represent an elegant way to stage operations against collection
that can be performed later.


### Examples

Views should allow the staging of operations against collections that
can be performed later.  Here's a simple example:

```scala
val x = List(1,2,3,4).view.map(_+2).filter(_%3==0).flatMap(_.toString)
val y = x.sum  // 9
val z = x(0)   // '3'
```

Additionally, views should be slightly more friendly in the presence of
mutability.   That is, views should read the state of the collection at the
time a forced operation is run.  

```scala
val x = ArrayBuffer(1,2,3,4,5)
val y = x.view.take(1)
pritnln(x.take(1).mkString("")) // prints: "1"
pritnln(y.mkString("")) // prints: "1"
x.clear()
println(x.take(1).mkString("")) // prints: ""
println(y.mkString("")) // prints: "", previously threw IndexOutOfBoundsException
```

TODO - More examples

### Comparison Examples

Code demonstrating why the library is needed, how equivalent functionality
might be provided without it.

TODO - Iterators?

### Counter-Examples

A view library is not a substitute for proper lazily evaluated collections.
Views are solely about deferring operations to be performed later, that is
chaining together a sequence of low-order transformation into a higher-level
algorithm for your computation.


On the other hand, scala.Stream is about lazily evaluating elements in a
collection.  scala.Stream attempts to make infinite collections viable, for
example, the fibonacci sequence can be defined as:

```scala
 val fibs: Stream[BigInt] =
   BigInt(0) #:: BigInt(1) #:: fibs.zip(fibs.tail).map { n => n._1 + n._2 }
```

This recursive usage is completely fine for lazy collections.  Additionally, it
is lazy collections which benefit more from foldRight optimizations.

The proposed view library is tuned to provide a simple and elegant method to
defer operations to be later performed against collections.   It is not designed
to solve the realm of lazy collections and algorithms that would benefit
from efficient foldRight implementations.   We consider these
collection types and algorithms deserving of their own library, that pays
special attention to the complexities of lazy computation.

The proposed view library is finely tuned for deffering foldLeft operations,
and specifically for collections which are finite.

## Drawbacks

The implementation proposed here has a few limitations which we'll discuss in order:

1. Parallelization
2. Specialization
3. Slicing


### Parallel Collections and views
While this library doesn't make the parallel view situation much worse, it
limits the ability for views to ever perform their computations in parallel
on a parallel collection.  This is due to the nature of the Fold operation
requiring sequential access, i.e. `(Accumulator, Element) => Accumulator`.

It is possible for `Fold` operations to be parallelized, but it requires the
following restrictions:

1. The `Fold` must not capture any state external to the `Accumulator`
2. Instances of `Accumulator` must form a SemiGroup
    (i.e. there is an idempotent function:  
       `(Accumulator, Accumulator) => Accumulator`)

There is no static mechanism in scala to enforce either of these two
restrictions, therefore Parallel view design will be a bit trickier to support,
and may require a much-restricted subset of the currently proposed
collection methods *or* use less efficient implementations.

It is a possibility that we re-orient the interfaces of the library to
require/enforce parallel semantics, at the expense of non-parallel performance.

We have opted to ensure single-threaded performance is optimal, while allowing
parallel behavior to exist, which we feel mirrors a lot of design choices
already made in the Scala collections library.


# Speicalization

TODO - Discuss

# Slicing

This library looses a modest amount of performance in the presence of slicing.
For example, if we have the following:

```scala
val x = ArrayBuffer(...).slice(20,500)
```

Then the current mechanism will ALWAYS have a constnat O(N) (wehre N =
  the initial slice number, e.g. 20) overhead on evaluation.  The current
  view implementations can reduce this overhead.   While the overhead can be
  significant for some operations, there are two things we've found:

1. The overhead is less than the cost of attempting to special-case slice
    within the library.
2. The overhead, for some particularly offensive scenarios, is on the order of
   30% over a custom solution.   We think this represents a minor drawback, and
   something that could be recovered via some specific "slice view" mecahnism
   that is outside the scope of this proposal.

## Alternatives

There were a few alternatives attempted before the current proposal:

1. Continue fixing the current view code base.  This is still an option,
   and remains on the table.
2. Leveraging macros and syntax tree lifting to stage an AST which would
   be interpreted (at runtime or compile-time) into a more efficient
   operation later.   While this is still possible, we feel adding a
   transducer + alternative view implementation does not prevent such a
   solution from also being viable.  Additionally this proposal represents,
   what we feel, is a much easier to maintain solution.
3. Using `java.util.stream`.   Java provides a vew similar mechanism to
   Transducers in Java 8, called `Collector`.  These collectors represent
   a more limited subset of what's expressible in Transducer, Additionally
   the current proposal could also be used from JDK7.  `java.util.Stream`
   does allow more specialization than the current proposal, but we think
   this is fixable and Transducers provide enough flexibility (with little
     maintenance) to warrant their inclusion.  Additionally, we think
     Scala's view API to be superior in elegance to the `java.util.stream`
     API, specifically in the case where `flatMap`s are required.


Additionally, there are a few Transducer libraries out in the wild:

TODO - List these

## Design

The new view design starts with a very simple implementation of Transducers,
specifically optimized for left-fold operations.  These transducers can be
used to stage collection manipulation operations.

A new `view` instance, therefore, is set of:

* A backing collection
* A stack of transducer operations which have been staged.
* A Builder for the result collection

We can provide an almost identical API to the one which exists for views today,
simply by capturing the `CanBuildFrom` that is implicit in most collection
operations and updating our state to retain it.   We do not have to remember
the input collection type, as long as we have our stack of transducers and
a builder for our intended output type, which dramatically simplifies the type
craftign in this library.

Additionally, the library provides flexible memoization via an explicit
`memoize` method, which will run the collection + transducers through
a fold, constructing a collection using the stored builder and saving
all of this in its state for the next operation.


### Footprint

The API should have a minimal footprint on Scala, and can be released as an
optional module.   The API consists of three core interfaces:

* Transducer
* StagedCollectionOperations
* View

In addition to these core interfaces there will be a set of Transducer
implementations for all collection operations as well as an implicit class
which will provide the `stagedView` method onto collections, shown here:

```scala
object View {
  import scala.collection.generic.IsTraversableLike
  /** This is our set of extension methods for existing collections. */
  final class ExtensionMethods[A, Repr](val staged: StagedCollectionOperations[A])
    extends AnyVal {
      def stagedView(implicit cbf: CanBuildFrom[Repr, A, Repr]): View[A, Repr] =
        ...
    }
  /** This implicit makes staged & stagedView available on all traversables. */
  implicit def withExtensions[Repr](
    coll: Repr)(implicit traversable: IsTraversableLike[Repr]) =  ...
}
```

This provides all the necessities of a View API with little impact to existing
scala code.   The new views will be completely optional, and not part of the
core collections library.

### Library Design and implementation

The library will be contained in a new `scala.colelction.view` package.
This package will contain three public class/module pairings:

*scala.collection.view.Transducer*
```
/**
 * A transformer over Folds.  Instead of calling it a "fold transformer" or "fold mapper" or anything so blaise, we'll
 * use a nicely marketed term which doesn't convey the meaning/implications immediately, but does well with SEO.
 *
 * Note:  A transducer is ALMOST a straight function, but we need to leave the Accumulator type unbound.
 *        In scala, these unbound universal types can be expressed via abstract classes + polymorphic inheritance.
 *        There is no convenient "lambda" syntax for these types, but we attempt to give the best we can for this concept.
 */
sealed abstract class Transducer[-A, +B] {
  /**
   * Alter an existing fold function as specified by this transducer.
   *
   * Note:  The resulting function may or may not be pure, depending on the Transducer used and whether the
   *        input function is pure.
   */
  def apply[Accumulator](in: Fold[Accumulator, B]): Fold[Accumulator, A]

  /** Run another transducer after this transducer. */
  final def andThen[C](other: Transducer[B,C]): Transducer[A,C] = new JoinedTransducer(this, other)
}
/** A container for all the default implementations of transducers available for scala views. */
object Transducer {
  def identity[A]: Transducer[A,A] = ...
  def map[A,B](f: A => B): Transducer[A,B] = ...
  def collect[A,B](f: PartialFunction[A,B]): Transducer[A,B] = ...
  def filter[A](f: A => Boolean): Transducer[A,A]  = ...
  def flatMap[A,B](f: A => GenTraversableOnce[B]): Transducer[A,B] = ...
  ... more default transducer factories...
}
```

*scala.collection.view.StagedCollectionOperations*
```
/** A collection of staged operations (transducers) that we build up against a source collection type.
  *
  * E.g., this could be a set of Map/FlatMap/Filter/slice operations to perform on a collection.   We assume
  * all collection methods can be aggregated into transducers.
  */
abstract class StagedCollectionOps[E] {

  /** The type of the elements of the source sequence */
  type SourceElement

  /** The source collection.  TODO - can we leave this unbound? */
  val source: GenTraversableOnce[SourceElement]

  /** The fold transformer, as a collection of operations. */
  def ops: Transducer[SourceElement, E]

  ... existing staged collection methods ...
}
```

*scala.collection.view.View*
```
/** A view represents a set of staged collection operations against an original collection.
  * Additionally, a view attempts to keep track of the most optimal collection to return from the
  * series of operations, using the `CanBuildFrom` pattern.
  *
  * @tparam E  The elements in the collection.
  * @tparam To The resulting collection type of the staged operations acrued so far.
  */
abstract class View[E, To] {
  /** The underlying implementation of the original collection.
    * We need this type to have existed at one time, but we no longer care what it is.
    */
  protected type From

  /** The currently saved builder we will use to build the result of the staged operations. */
  protected val cbf: CanBuildFrom[From, E, To]

  /** The source collection and its staged operations. */
  val underlying: StagedCollectionOperations[E]

  /** Runs the staged operations and returns a new view containing the cached result. */
  def memoize: View[E, To]

  ... existing staged collection methods ...
  ... existing forced collection methods ...
}
object View {
  import scala.collection.generic.IsTraversableLike
  /** This is our set of extension methods for existing collections. */
  final class ExtensionMethods[A, Repr](val staged: StagedCollectionOperations[A])
    extends AnyVal {
      def stagedView(implicit cbf: CanBuildFrom[Repr, A, Repr]): View[A, Repr] =
        ...
    }
  /** This implicit makes staged & stagedView available on all traversables. */
  implicit def withExtensions[Repr](
    coll: Repr)(implicit traversable: IsTraversableLike[Repr]) =  ...
}
```

Where the existing staged collection operations are:

- map
- flatMap
- collect
- filter
- filterNot
- slice
- take
- takeWhile
- drop
- dropWhile
- tail
- init
- zipWithIndex
- zip
- partition (returns 2 views)

Each of these staged collection operations have corresponding
Transducer operations.  These operations will be implemented by
capturing arguments and a new Builder for the collection, and appending
the staged operation onto the transducer stack, for example:

```scala
// abstract class View ...
  final def map[B, NextTo](f: E => B)(implicit cbf: CanBuildFrom[To, B, NextTo]): View[B, NextTo] =
     SimpleView(
       underlying.map(f),
       cbf.asInstanceOf[CanBuildFrom[From, B, NextTo]]

// abstract class StagedCollectionOps
final def map[B](f: E => B): StagedCollectionOps[B] =
  new StagedCollectionOps(underlying, ops andThen Transducer.map(f))
```

Where `andThen` is a method on `Tranducer` which can append a new operation
onto the transducer stack.



The existing forced collection operations are:

- last
- head
- foldLeft (/:)
- foldRight (\:)
- find
- size
- exists
- forall
- count
- collectFirst
- reduceLeft
- reduceRight
- reduce
- reduceLeftOption
- reduceRightOption
- reduceOption
- aggregate
- sum
- product
- min
- max
- maxBy
- minBy
- to*  (toArray, toTraversable, `to[_]`, etc.)
- mkString

Each of these will have an associated Fold/Accumulator function that implementation
them.  For example:

```scala  
final def foldLeft_![Accumulator](acc: Accumulator)(f: (Accumulator, E) => Accumulator): Accumulator =
  source.foldLeft(acc)(ops.apply(f))  // Apply the transducer stack to f.
```


### Forced vs. staged operations
The library denotes a difference between forced and staged operations.

Staged operations will not be computed, instead they will add a new Transducer
onto the operation stack.

Forced operations will cause the current transducer stack to be run against the
underlying collection, leading to an immediate value.

It is a design decision whether or not to specially denote forced operations in
the API.  While the current view API does not do this, it is because they are
directly inheriting from the collections where each operations is always
forced/performed immediately.   We'd like to propose that any operation
which forces evaluation of a view be terminated with a uniform suffix, e.g. "!"


## Implementation

There is a [prototype library available][1] for those who would like to
see this new implementation.   The library has implemented enough of the
framework to prove:

1. The implicit extension is viable, and can capture collection types
   accurately.
2. Storing a Builder is sufficient for ensuring the appropriate collection
   is returned at the end of a chain of operations, according to the
   `CanBuildFrom` logic.
3. We can acheive sufficient (or better) performance than existing collection
   views.
4. Memoization is possible, and does not provide enough performance gain to
   always be enabled.

We believe this library could quickly be cleaned up and turned into a new
Scala view module, while the existing lirbary be deprecated.  We would like to:

1. Scala 2.12.x
   - Include a scala-view module in the next 2.12 milestone released
   - Mark existing `view` methods deprecated on all collections in scala master.
2. Scala 2.13.x
   - Remove existing View API from the standard library
   - Expose custom Transducer library if there is demand.

## Unresolved questions

This proposal leaves open for discussion/contribution several things:

1. Should operations which force evaluation have a common suffix (like `_!``) or
   should they mirror the existing collections methods.
2. More investigation into making the transducer/view design friendly for
   parallel collections.
3. Additional performance analysis/consideration for Transducer implementations
   not in the prototype.

## References

1. [viewducers][1]
2. [API Documentation][2]
3. [Academic/Research papers/supporting material][3]
4. [Alternative Libraries/Implementations][4]
5. [Discussion forum/post/gitter/IRC][5]

TODO

[1]: http://github.com/jsuereth/viewducers "Viewducers"
[2]: http://www.scala-lang.org/api/ "Scaladoc"
[3]: http://en.wikipedia.org/wiki/Academic_publishing "Academic/Research"
[4]: https://github.com/dogescript/dogescript "Alternatives"
[5]: https://gitter.im "Gitter"
