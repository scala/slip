# Adding JSON AST Standard

** By: Matthew de Detrich, json4s organisation and Ivan Porto Carrero (original author) **

This proposal is an attempt to add a json-ast library for Scala.

## History

| Date          | Version       |
|---------------|---------------|
| Nov 4th 2015  | Initial Draft |

## Introduction to json-ast

Scala currently is in a bit of a quagmire when it comes to json libraries.
There are roughly 6 competing JSON ast libraries, all of them are slight
variations and they all attempt to solve 2 problems (one is speed, often
used for parsing and the other is safe immutable representation, 
which is often used for quering). Here is a good 
[link](http://manuel.bernhardt.io/2015/11/06/a-quick-tour-of-json-libraries-in-scala/)
which shows how absurd the current situation is, the amount of duplication is huge

Some time ago, casualjim (Ivan Porto Carrero) made an attempt to 
create a json ast library (called json4s-ast) 
in collaboration with the major scala webframeworks at 
the time, to try and create a unified API. casualjim however has moved on,
and so the project has stagnated

Recently I have reinstigated the project, which is currently hosted at
[json4s-ast] (https://github.com/json4s/json4s-ast). Most of the communication
has been done over the gitter channel, however we have had collaboration
from many of the developers/creators of popular Scala libraries, including
* Spray
* jawn
* rapture
* Scalatra
* Lift
* Play
* SBT

The issue presents itself more in frameworks/libraries that have to deal with JSON.
As a quick example, [slick-pg](https://github.com/tminglei/slick-pg), an extension ontop
of slick for Postgres extensions, needs to provide implements of the JSON type for each of
the commonly used JSON libraries, instead of just needing to support one

Conversation regarding the collaboration can be viewed by reading the gitter 
[history](https://gitter.im/json4s/json4s).

### Examples

Examples can be currently seen at 
the json4s-ast [repo](https://github.com/json4s/json4s-ast).

### Counter-Examples

Current json4s-ast is strictly a library that only provides an AST. It isn't designed to
address parsing/construction/querying. This is up to library/application developers.
The central problem being solved, is that regardless of where you get a `JValue`, you will
always be able to interopt `JValue` with other libraries.

## Drawbacks

JSON is still a rapidly moving field, and there are developments (such as BSON from mongodb)
which may be incompatible with json4s-ast (someone may need to verify this?).

There are currently drawbacks in regards to deciding whether to use `Array` or `js.Array`
for Scala.js implementation. Using `Array` maintains type equality over Scala/Scala.js, however
it will have less performance. Using `js.Array` maintains source equality over Scala/Scala.js (which
isn't as good as type equality), however has better performance

The `fast` version is unsafe (however this is intentional in design). It can blow up, at
runtime.

Trying to cement a universal type (in this case, essentially the `String` version of `JSON`),
poses risks if its not done correctly and/or if its not what people need/want. Don't need
a situation occuring similar to what happened with regards to Java time libraries (i.e.
[jodatime](http://www.joda.org/joda-time/)).

`Vector` may not be the best representation for the `safe` JArray, however this is what was decided
by for the community.

`safe` is currently very strict about only adhering to the JSON spec, i.e. it uses `Map` for JObject
which means it doesn't have a concept of ordering. Although this is correct as per json [spec](http://www.json.org/),
it does create practical issues. As an example, a past [issue](https://github.com/json4s/json4s-ast/issues/8) would have
been requiring ordering on a `JObject` so you can deterministically create a cheksum (in order to verify if 2 `JSON` 
objects are equal). With a representation of `Map` for `JObject`, it would not have been easily possible to solve this.
This was solved by providing the alternate `fast` API (uses `Array` which has a concept of ordering), however issues
like this can crop up

## Alternatives

There are a huge mumber of json ast alternative libraries, this includes
* [spray-json](https://github.com/spray/spray-json)
* [json4s old version](https://github.com/json4s/json4s)
* [jawn](https://github.com/non/jawn)
* [lift-json](https://github.com/lift/lift/tree/master/framework/lift-base/lift-json/)
* [argonaut](https://github.com/argonaut-io/argonaut)
* [rojoma](https://github.com/rjmac/rojoma-json)


## Design

For greater in-depth reasoning for the design goals, you can read the 
[README.md](https://github.com/json4s/json4s-ast/blob/master/README.md)
on the project site

The general consensus was, it wasn't really ideal to create a single unified API
that made everyone happy. People had different goals for a JSON AST library. Some wanted
the highest performance possible, and others wanted a library that was correct and safe.
Some people also wanted features that weren't strictly part of the JSON spec, but incredibly useful
(i.e. ordering for a JSON Object) and some people wanted to minimize memory usage 
(important in big data)

Due to this reason, the current design of the JSON AST has 2 implements, one called
`fast` and the other called `safe`. `fast` is designed to be as fast as possible, and hence
uses many mutable datastructures (such as `Array` for JSON Array/Object, JSON Number uses a 
String to avoid runtime penalty). Safe, on the other hand, represents an always correct 
representation of JSON. It is also safe in regards to performance
characteristics for lookups (i.e. JSON Array is represented as a `Vector`, which provides near
constant lookup time, and JSON Object is represented as a `Map`, which also provides near
constant lookup time for keys, as well as providing `Map` equality regardless of ordering)

There are also conversion functions between the two libraries, allowing you to 
easily go from `safe` to `fast` and vice versa.

An easy way to think about the difference between `safe` and `fast`, is that `safe` is your
`String` where as fast is your `Array[Byte]`. Both are equally important, and both are
needed for different reasons.

### Footprint

The footprint json4s-ast was intentionally designed to be as small as possible. The current
spec is trivial in design (and only a page long for each implementation, i.e. 
[safe implementation for jvm](https://github.com/json4s/json4s-ast/blob/master/jvm/src/main/scala/org/json4s/ast/safe/JValue.scala)

### Scala.js Support

The current version of json4s-ast has support for Scala.js, which also 
has exposed constructors for use in Javascript. There are some slight differences
(such as using `js.Array` instead of `Array` for performance reasons, note that
this isn't final)

### Implementation

There is good reasons (either way) about whether the json ast should be included in 
`stdlib` or as a separate supported module. The current json4s-ast has been deliberately
designed to have a strict implementation that should almost never change which gives
support for adding it into `stdlib`. It also has (strictly) zero dependencies, and the
implementation is really trivial

The only issue regarding implementation would be how to treat Scala.js support (ideally
the Scala.js section should be moved to the actual Scala.js implementation at [Scala.js]
(https://github.com/scala-js/scala-js), and the official Scala stdlib will only hold the
JVM implementation

The current state of the project is its a SNAPSHOT, this is for various reasons
* We are looking for a way to benchmark on Scala.js (this is to help in gauging whether to
use `js.Array` or `Array` for Scala.js implementation). Note that this isn't related, at all,
to the JVM implementation.
* Need to do some benchmarking to verify that the current conversion methods (`toSafe` and `toFast`)
provide the fastest possible implementations (they currently use builders)
* More feedback from community that its correct

Regarding timeframe, since a current Scala JSON standard doesn't officially exist, it can 
be added at any time. The library is written with idiomatic Scala, so its unlikely to break
(even between major Scala releases). The current json4s-ast implementation supports `Scala`
2.10 and 2.11. There should be no problems in supporting 2.12.

Current developer/contributor list can be seen at Contributers/Developers can 
be seen [here](https://github.com/json4s/json4s-ast/blob/master/build.sbt#L41-L115) and
implementation can be seen [here] (https://github.com/json4s/json4s-ast)

### Unresolved questions

As per the current [issues](https://github.com/json4s/json4s-ast/issues), there are some
minor questions about whether we should provide converters for common types (specifically
numbers). These currently exist, however there is an argument about whether to remove this from 
the current implementation to reduce its footprint. 
There are also questions about the `js.Array`/`Array` problem for Scala.js.

Since [scala-offheap](https://github.com/densh/scala-offheap) just came out, there is a
question whether the `fast` implementation should use scala-offheap (which would improve
performance). However this would break the current strict zero dependency claim and
scala-offheap is fairly new (there are also questions about whether it will exist in
future versions of the JVM)

For `fast`, should we make the internal `Array` private (i.e. unable to be modified once you construct
the `JArray`/`JObject`). `Array` itself is mutable, and fast is designed to provide speed, but there are
good arguments to make it private.

## References - Quotes from [gitter](https://gitter.im/json4s/json4s)
Note that these quotes are to demonstrate adoption/collaboration. If I have quoted out of context, please let me know, also if you want a quote
to be removed/clarified

@jroper (Play)
> Let me just say - from my point of view, if json4s AST gets released as it is currently in the reboot branch, I would be happy. and, if it gets released with some of the changes that are being mentioned (not just my own), I would be happy. We would seek to adopt it in Play (the question would be when, not if). Right now I think this is about iterating over an optimal solution to try and find a minimum that satisfies as many people as possible - ie, we’re close enough, let’s see if we can get closer.

@farmdawgnation (Lift)
>Anyway, @mdedetrich, thanks for the work so far on this. I think there’s a lot of value in unifying AST’s. Can’t guarantee we’ll adopt it for lift-json, but it would be nice to know as a Scala developer that I could use any of the Big JSON Libs and they can interoperate without too much work. 
 The current status quo has (unfortunately) bit me in the butt a few times.

@rossabaker (json4s/http4s)
>Two ASTs sounds complicated, but I guess it makes nobody happy to have a lack of safety paired with middling performance.

> I think there's a lot of room for disagreements and innovation in codecs and traversals and DSLs and which sum types get returned. That's why there are so many JSON projects today.
  There seems to be a pretty near consensus on how to map the JSON AST to Scala case classes. If this minimal AST is of any use to getting projects to standardize on that before adding their own library around it, cool.
  If not, the proposal is at least a much better foundation for a json4s reboot.

@JRudolf (Spray/Akka)
> @non yes, of course, it makes sense to keep additional features per library. AFAIU the goal is to find a common denominator between json libraries that is more structured than "just Scala". On top of that diversity has its advantages.

@non (jawn)
> @jrudolph so -- just to be clear, jawn will definitely support parsing to this AST, but i'm not necessarily committing to removing my own mutable AST.

@sirthias (Spray)
> I imagine that once we have this common AST and the AST really is minimal in what it provides (as it should be) end-users will turn to convenience tools that provide XPath-like querying, lenses, (de)serialisation, etc.

> Well, @mdedetrich took this on. And @non, @eed3si9n, @bryce-anderson, myself and others merely chipped in.

> Gentlemen,
  I’ve compiled a quick benchmark to compare the new AST proposals with what we currently have in spray-json and other JSON libs.
  The interesting questions for me are: Does an Array-based “basic” AST really yield better parsing performance over what we have now? How does it compare to a Vector-based “basic” AST?

@propensive (Rapture)
> Again, I don't have much to constrain me (and hence to contribute to the discussion). However the AST ends up, I'll have a typeclass layer, and that should be quite trivial to implement.  
> Agreed. If the AST changes more frequently than that, it undermines some of its purpose anyway...
> It wouldn't be a terrible thing to separate the JSON4S AST code into a minimal library. But there's more of a community/political job of getting other libraries to start using it...

@eed3si9n (SBT)
> @rossabaker yes. I was able to pull out json4s-core dependency after some copy-pasta from ParserUtil. I'd like to have binary-compatible long term AST project that encodes version number in the package name.

> it's the number 1 concern for me. that's why I use jawn
  but in the case of sbt, i don't think we deal with numbers much
  
> I'd actually say port all that good stuff from Yoshida-san into the new json4s-ast project and start over

> right. lean AST jar that's versioned should allow sbt plugins to use whatever version of json4s

## References

1. [Existing project (json4s-ast)][1]
2. [Issues][2]
3. [Gitter Channel][3]
4. [Casualjim][4]
5. [JSON organisation][5]

[1]: https://github.com/json4s/json4s-ast
[2]: https://github.com/json4s/json4s-ast/issues "Issues"
[3]: https://gitter.im/json4s/json4s
[4]: https://github.com/casualjim
[5]: https://github.com/json4s