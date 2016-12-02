# Document standard libraries

This discussion issue centres around [this gitter
conversation](https://gitter.im/scala/slip?at=5618e3b71b0e279854bdb383) and
aims to summarise the approach to standard-like libraries used in various
languages.

The [wikipedia article on the philosphies of standard
libraries](https://en.wikipedia.org/wiki/Standard_library#Philosophies) is a
useful starting place, with two opposing views

- C/C++:  "..relatively small standard library, containing only the constructs
  that "every programmer" might reasonably require". Not mentioned, but Rust,
  LLVM and Javascript probably live here
- Scripting languages like Python/Perl, and those that use a VM, Java/C#
  "batteries included". Not mentioned, but Perl, Ruby (and R?) live here.

I'm not sure where Haskell lives, and I'm not sure where Scala is, nor wants
to live. Hence this discussion.


## My Opinion

Is just my opinion, based on what I know today. It may change, and is
certainly __not my advice__ for Scala:

- Have a low level library needed for the SLS
- Have a package repository(PKG) - linking to code on Github, etc - that tags the 
  libs, specifies dependencies, etc
- Have a process for some libs from PKG to be peer reviewed - PR-PKG
- Allow companies (eg Typesafe) to provide long term support (eg guarantee
  compatabilty between version,etc) of a subset of PR-PKG
- Introduce a small number of other sets of libraries for specific target users. Eg: Script-Libs, FP-Libs. 
- Introduce a minimal std library, based on an interface, where libraries are
  explicitly loaded and can be provided by many providers
- Introduce a "Scala Typeclass Library" - this may actually just be the
  previously mentioned "Std Library"

## Target Users

Corporates - No internet packages are alloed and tend to create their own "Base" libraries behind a
firewall, even for SBT. The advantage to them of a "single" standard library is considered an win.

Beginners/New users - need a "head start", historically provided by a good std-lib

Advanced users need all the latest goodies, often unavailable as they are not
in the std-lib (ie on the wrong side of the firewall) and besides, all the
projects have been written using the standard library in any case. 

# Languages

## C++

Designed with a bias toward system programming

Originally had no standard library, so developers had their own "Base Library"
that they carried around. Largely replaced by in-house maintained Base library
that all had to use. Roguewave was an expensive library often used as the core for
these base libs. 

The Standard Template Library(STL) was introduced as the library for
generic programming, divided into Containers ,Iterators, Algorithms and
Functors. As as sepecification, companies (including Rogue
Wave) were able to produce STL implementations. STLPort was a free version
that became popular as it's maturity increased, and pay versions became more
expensive.

In 1998 the C++ Standard library was introduced, based on
STL conventions. Eventually it became an ISO standard.

The boost library was proposed in 1998 as a "a C++ Library Repository Web
Site" by a few members of the standards committee. The vision was to have peer
reviewed libraries, existing individually but released as a single, versioned
package, to speed up up the process of getting new libraries into the standard
library.

### C++ Opinions

- "In mid-2011, the new C++ standard (dubbed C++11) was finished. The Boost
library project made a considerable impact on the new standard, and some of
the new modules were derived directly from the corresponding Boost
libraries. Some of the new features included regular expression support".  C++
had no regex until 2011, provided by Boost, a library started in 1998 to speed
up the adoption of new libraries to std. Daft?

- "Boost is proof that a standard library does not stop the development of new
  ideas." But Boost was only born _because_ the standard library had stopped.

- Hearsay suggests that not having a wide std-library was a key reason why
  Java was able to dominate so quickly (in the Enterprise)

- C++ had a huge entry barrier, both down to the language itself and the lack
  of a standard library - and a "standard" way of doing anything. Both a
  blessing and a curse, the "Corporates" made sure this was a curse.

- C++ ended up to compicated for engingeers, who seem to love Python, and too 
  "bulky" for low level implementatins where C is preffered. And for Corporates,
  Java/C# was just a better "fit". So C++ lost its identity.

### C++ Takeaway

As a low-level systems languages, born before the WWW, it's perhaps not
surprising that "design by committee" ruled. But it's still widely in use.

Decentralisation via Boost was smart, and is a great compromise between an
official "Std-Lib" and a package repository. Perhaps there is room for a scala
"boost", a subset of which provided by Typesafe as a maintained 5-year
library, loaded as a pre-approved package. 

The STL actually sparked their std-library, perhaps Scala could have a new
"Scala Typeclass Library".

### C++ Links
https://www.quora.com/Why-is-Java-standard-library-larger-when-compared-to-C++s-standard-library
https://en.wikipedia.org/wiki/C%2B%2B_Standard_Library
https://www.softwariness.com/articles/cpp-libraries/
https://en.wikipedia.org/wiki/Rogue_Wave_Software
http://www.cplusplus.com/info/history/
https://en.wikipedia.org/wiki/Standard_Template_Library#History
http://www.boost.org/users/proposal.pdf
https://en.wikipedia.org/wiki/Standard_Template_Library#Algorithms

## Haskell
https://en.wikibooks.org/wiki/Haskell/Modules
https://www.haskell.org/onlinereport/haskell2010/
https://en.wikibooks.org/wiki/Haskell/Libraries
https://hackage.haskell.org/package/rdf4h

## Java

A general purpose language that runs on the JVM. 
https://en.wikipedia.org/wiki/Java_Class_Library

### Java Opinions

- Seen by many as a low-level, "hard" language with lots of boilerplate required to get anything done. 

## Javascript

A high level, dynamic, untyped, and interpreted programming language, one of
the three essential technologies of WWW.

Has a small library, and more "Frameworks" than perhaps any other language 

http://zef.me/blog/2856/javascript-a-language-in-search-of-a-standard-library-and-module-system

# .Net
https://msdn.microsoft.com/en-us/library/mt472912(v=vs.110).aspx

## Perl
https://www.google.ch/search?q=cpan&oq=cpan&aqs=chrome..69i57j0l5.1047j0j4&sourceid=chrome&es_sm=93&ie=UTF-8
https://metacpan.org/pod/RDF::NS

## Python
https://docs.python.org/2/library/
https://pypi.python.org/pypi
https://pypi.python.org/pypi/rdf/0.9a6

## R

AN old language, but still very popular to the number of  packages available
https://cran.r-project.org/web/packages/RNeXML/index.html

## Rust
https://doc.rust-lang.org/std/index.html

## Scala
http://www.scala-lang.org/files/archive/spec/2.11/12-the-scala-standard-library.html#standard-reference-classes


