# State of Valhalla

#### Section 1: The Road to Valhalla
#### Brian Goetz, Nov 2020

::: sidebar
Contents:

1. [The Road to Valhalla](01-background.html)
2. [Unifying the Java Type System](02-object-model.html)

:::

## Background

[_Project Valhalla_][valhalla] got its start in 2014, with the goal of bringing
more flexible flattened data types to JVM-based languages, in order to  restore
alignment between the programming model and the performance characteristics of
modern hardware.  (In some ways, it got started much earlier; the designers of
Java wanted to include value types in the initial version of the language.)
Initially, Valhalla was described in terms of the features we anticipated adding
to meet this goal: _primitive classes_ (known in various iterations as _inline
classes_ or _value types_) and _specialized generics_.  In the initial phase of
the project, we focused primarily on understanding how the language and JVM
would have to change to support these features, and what the migration
compatibility implications for user code would be.

We now believe we have a coherent path to enhance the Java language and virtual
machine with primitive classes, have a compatible path for migrating both the
existing primitive types and existing value-based classes to primitive classes,
have primitive classes interoperate cleanly with erased generics, and migrate
existing generic classes to specialized generics.  This set of documents
summarizes that path, to be  delivered in stages.  (If you want to compare with
where we started, see our [founding document][values0].)

However, Project Valhalla is not just about the features it will deliver, or
about improving performance; it has the more ambitious agenda to _unify the Java
type system_ -- unify primitive types with classes, and to allow generics to
range over any type.

#### The costs of indirection

The JVM type system includes primitive types (`int`, `long`, etc.), classes
(heterogeneous aggregates with identity), and arrays (homogeneous aggregates
with identity).  This set of building blocks is flexible -- you can model any
data structure you need to.  Data that does not fit neatly into the available
primitive types (such as complex numbers, 3D points, tuples, decimal values,
strings, etc.), can be easily modeled with objects.  However, objects are
allocated in the heap (unless the VM can prove they are sufficiently narrowly
scoped and unaliased), require an object header (generally two machine words),
and must be referred to via a memory indirection.  For example, an array of XY
point objects has the following memory layout:

![Layout of XY points](xy-points.png){ width=100% }

When the Java Virtual Machine was being designed in the early 1990s, the cost of
a memory fetch was comparable in magnitude to computational operations such as
addition.  With the multi-level memory caches and instruction-level parallelism
of today's CPUs, a single cache miss may cost as much as 1000 arithmetic issue
slots -- a huge increase in relative cost.  As a result, the pointer-rich
representation favored by the JVM, which involves many indirections between
small islands of data, is no longer an ideal match for today's hardware.  We aim
to give developers the control to match data layouts with the performance model
of today's hardware, providing Java developers with an easier path to _flat_
(cache-efficient) and _dense_ (memory-efficient) data layouts without
compromising abstraction or type safety.  

What we would like is to have the option to get a layout more like this:

![Flattened layout of XY points](flattened-points.png){ width=60% }

This layout is both flatter (no indirections) and denser (no headers) than the
previous version.

(Related to object layout is _calling convention_, which is how the JVM passes
values from one method to another.  In the absence of heroic JVM optimizations,
when a method passes a `Point` to another, it passes a pointer, and then the
callee must dereference the pointer to access the object's state.  Enabling a
flatter representation is also likely to enable a flatter calling convention,
where a `Point` can be passed by passing the `x` and `y` components by value, in
registers.)

One of the key questions that Project Valhalla addresses itself to is: _what
code would we want to write to get this layout?_

To understand our options, we must first understand where the current
unfortunate layout comes from.  The root cause here is _object identity_: all
object instances today must have an object identity.  (In the early 90s,
"everything is an Object" was an attractive mantra, and the performance costs of
identity were not onerous.)  Identity enables mutability; in order to mutate a
field of an object, we must know _which_ object we are trying to modify.  And
even for the many classes that eschew mutability, identity can
be observed by various identity-sensitive operations, including object
equality (`==`), synchronization, `System::identityHashCode`, weak references,
etc.  The result is that the VM frequently must pessimistically preserve
identity just in case someone might eventually perform an identity-sensitive
operation -- even if that is never going to happen.  Thus, identity leads to the
pointer-rich memory layout we have today.

Separately, classes today support subclassing (unless the author opts out with
the `final` keyword).  When a class can be extended, its type has an
unpredictable layout -- at any point during program execution, some subclasses
might not even be loaded yet!  So it's impossible to allocate a fixed amount of
memory large enough to store any future subclass instance.

The direction that Valhalla has pursued here is to say that some classes can
disavow identity, mutation, and extension; instances are entirely represented by
their fixed-size state, and therefore can be routinely flattened into arrays and
into other objects, and passed as flat vectors between methods.

Such classes are called _primitive classes_, to highlight the fact that they
have the runtime behavior of primitives.  (The terminology has changed during
the course of the project; previously-used synonyms include _inline classes_ and
_value classes_.)  The slogan for such classes is

> Codes like a class, works like an int.

Despite the restrictions on mutability and subclassing, primitive classes can use
most mechanisms available to classes: methods, constructors, fields,
encapsulation, supertypes, type variables, annotations, etc.

There are applications for primitive classes at every level.  Aside from the
obvious -- turning primitive types into real classes -- many API abstractions,
such as numerics, dates, cursors, and wrappers like `Optional`, are naturally
modeled as identity-free classes.  Additionally, many data structures, such as
`HashMap`, can profitably use primitive classes in their implementations to
improve efficiency.  And language compilers can use them as a compilation target
for features like built-in numeric types, tuples, or multiple return.

## What about generics?

One of the early compromises of Java Generics is that generic type variables can
only be instantiated with reference types, not primitive types.  This is both
inconvenient (we have to say `List<Integer>` when we mean `List<int>`) and
expensive (boxing imposes a performance overhead).  With eight primitive types,
this restriction is something we learned to live with, but if we can write our
own flattenable data types like our `Point` above, having an `ArrayList<Point>`
not be backed by a flattened array of `Point` seems to defeat, well, the point.

Parametric polymorphism always entails tradeoffs between code footprint,
abstraction, and specificity, and different languages have chosen different
tradeoffs.  At one end of the spectrum, C++ creates a specialized class for each
instantiation of a template, and different specializations have no type-system
relationship with each other. Such a _heterogeneous translation_ provides a high
degree of specificity, to the point where expressions such as `a+b` can be
interpreted relative to the behavior of `+` on the instantiated types of `a` and
`b`, but entails a large code footprint as well as a loss of abstraction --
there is no type that is the equivalent of `Foo<?>` in Java.

At the other end of the spectrum, we have Java's current erased implementation
which produces one class for all reference instantiations and no support for
primitive instantiations.  Such a _homogeneous translation_ yields a high degree
of reuse, since there is only one class and one object layout for all
(reference) instantiations.  It carries the restriction that we can only range over types
that have a common runtime representation, which in Java is the set of reference
types.  This restriction has its roots deep in the design of the JVM; there
are different bytecodes for operations on reference vs primitive values.

While most developers have a certain degree of distaste for erasure, this
approach has a powerful advantage that we could not have gotten any other way:
_gradual migration compatibility_.  This is the ability to compatibly evolve a
class from non-generic to generic, without breaking existing sources or binary
class files, and leaving clients and subclasses with the flexibility to migrate
immediately, later, or never.  Offering users generics, but at the cost of
throwing away all their libraries, would have been a bad trade in 2004, when
Java already had a large and vibrant installed base (and would be a worse trade
today).

Our goal today is even more ambitious than it was in 2004: to extend generics so
that we can instantiate them over primitive classes with specialized
(heterogeneous) layouts, while retaining gradual migration capability.
Further, migration compatibility in the abstract is not enough; we want to
actually migrate the libraries we have, including Collections and Streams.

This goal will be achieved through a combination of i) allowing primitive class
instances to be treated as lightweight Objects; ii) supporting interoperation of
primitive class types with existing erasure-based generics; and iii) developing
optimized heterogeneous _species_ in the JVM for cases in which enough type
information is available at run time.

## A brief history

Project Valhalla has ambitious goals, and its intrusion is both deep and broad,
affecting the `class` file format, JVM, language, libraries, and user model.  In
2014, James Gosling described it as "six Ph.D theses, knotted together." Since
then, we've built six distinct prototypes, each aimed at a understanding a
separate aspect of the problem.

The first three prototypes explored the challenges of generics specialized
directly to primitives, and worked by bytecode rewriting.  The first ("Model 1")
was primarily aimed at the mechanics of specialization, identifying what type
behavior needed to be retained by the compiler and acted on by the specializer.
The second ("Model 2") explored how we might represent wildcards (and by
extension, bridge methods) in a specializable generic model, and started to look
at the challenges of migrating existing libraries.  The third ("Model 3")
consolidated what we'd learned, building a sensible `class` file format that could
be used to represent specialized generics.  (For a brief tour of some of the
challenges and lessons learned from each of these experiments, see [this
talk][adventures] ([slides here][adventures-slides]).

The results of these experiments were a mixed bag.  On the one hand, it worked
-- it was possible to write specializable generic classes and run them on an
only-slightly-modified JVM.  On the other hand, roadblocks abounded; existing
classes were full of assumptions that were going to make them hard to migrate,
and there were a number of issues for which we did not yet have a good answer.

We then worked the problem from the other direction, with the "Minimal Value
Types" prototype, whose goal was to prove that we could implement flat and dense
layouts in the VM.  The turning point came with the prototype known as "L World"
(so called because it lets primitive classes share the `L` carrier with object
references).  In the earliest explorations, we assumed a VM model where
primitive classes were more like today's primitives -- with separate type
descriptors, bytecodes and top types -- in part because it seemed too daunting
at the time to unify references and primitives under one set of type
descriptors, bytecodes, and types.  L-world gave us this unification, which
addressed a significant number of the challenges we encountered in the early
rounds of prototypes.

## Moving forward

We intend to divide delivery of Project Valhalla into two broad phases:
primitive classes, and specialized generics.  (These may be further divided into
delivery milestones.)  The first phase will focus on support for primitive
classes in the Java language and virtual machine, and migrating the existing
primitive types to primitive classes.  This phase will also lay the groundwork
for the use of primitive classes in the JDK, and even migrating some existing
value-based classes (such as `Optional` or `LocalDate`).

The second phase will focus on generics, extending the generic type system to
support instantiation with primitive classes and extending the JVM to support
specialized layouts.


[valhalla]: http://openjdk.java.net/projects/valhalla
[values0]: http://cr.openjdk.java.net/~jrose/values/values-0.html
[adventures]: https://www.youtube.com/watch?v=TkpcuL1t1lY
[adventures-slides]: http://cr.openjdk.java.net/~briangoetz/valhalla/Adventures%20in%20Parametric%20Polymorphism.pdf
[model3]: http://cr.openjdk.java.net/~briangoetz/valhalla/eg-attachments/model3-01.html
