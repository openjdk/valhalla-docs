# Valhalla: Unifying the Java Type System

The Java type system circa Java 1.0 has eight primitive types (integer and
floating point numbers of various sizes, booleans, and characters),
user-declared classes and interfaces (including the special root class
`Object`), and a structural type operator `[]` for "array of" that can operate
on any type (including array types.)  As an OO language, this already embodied a
significant compromise: an OO language would like to start from the premise of
"everything is an object".  But, an `int` is not an object; it is something
special and magical unto itself (and so are its arrays), and this nonuniformity
ripples throughout the language, libraries, and runtime.

The compromise made in 1995 was that _everything the user can define_ is an
object, but there are eight additional built-in types that are not objects.  It
was surely a forced move; at the time, it was not yet known how to get away with
"everything is an object" and still offer reasonable numeric performance, and
not having reasonable numeric performance would have been a serious impediment
to Java's usability.  It didn't seem so bad at the time, and we've been able to
accomplish great things despite it, but it is an ongoing tax on developers,
library designers, and users.

When generics came along in 2004, it got slightly better -- and a lot worse.
The "better" part is that autoboxing papered over some of the visible seams
(though at a significant cost to the complexity of overload resolution), so we
could freely use an `int` where an `Integer` was expected and vice-versa.  But,
this addressed only the surface problem, not the underlying rift;  the set
of places where we had to be aware of the rift between primitive and reference
types grew, because primitives could not be used as generic type parameters.
Again, this was a pragmatic compromise -- and the only way known at the time to
add generics to Java without massive compatibility pain -- but the ongoing tax
only grew larger.

It got worse again when lambdas came along in in 2014.  Lambdas build heavily on
generics, so many of the consequences faced by generics were inherited by
lambdas.  This rippled into the libraries; `java.util.function` suffers a
combinatorial explosion of hand-specialized versions (`IntPredicate`,
`IntToLongFunction`), rather than being able to parameterize more general types
(`Predicate<int>`, `Function<int, long>`.)  The goal of generics is to abstract
over representational differences, but the primitive-reference divide was
getting harder to bridge.

#### Enumerating the costs

The primitive-reference divide has consequences at nearly every level of the
platform.

At the classfile level, a significant amount of the surface area is dedicated to
primitives.  Not only do the primitive types each have their own special type
descriptors, but over 60% of the bytecodes are some sort of primitive-specific
operation (e.g., `iload`, `dmul`, `fstore`, `i2l`).  This helped in the days
when Java was interpreted, and the JVM could identify the primitive operations
easily, but interpretation gave way to JIT compilation only two years later.

At the language level, primitives and objects differ in almost every way.
Objects have members such as fields and methods; primitives do not.  Objects
have identity; primitives do not.  Object references are nullable; primitives
are not.  Objects participate in polymorphism through superclasses and
interfaces; primitives cannot (even though it would be quite sensible for `int`
to implement `Comparable`).  `Object` is the top type -- but only for classes
(and the same problem presents for arrays; `Object[]` is the top type for arrays
of class types, but not for arrays of primitives).  The lack of subtyping with
`Object` means that primitives cannot directly participate in dynamically typed
libraries such as reflection (where everything is expressed as `Object` or
`Object[]`) -- they can only do so through their wrapper companions.

Having to go through the wrapper companions is not intrinsically bad; the
meaning of `ArrayList<Integer>` is clear enough, and autoboxing lets us deal
with such types in a syntactically convenient way.  But objects have identity
whereas primitives do not, and boxing is not able to fully paper over this gap.
Each time we convert from `Integer` to `int` the identity is lost, and each time
we convert from `int` to an `Integer`, a fresh (but accidental) identity is
created.  While `int` boxes to `Integer`, `int[]` does not box to `Integer[]`.
And the relationship between primitives and their corresponding wrappers are
entirely ad-hoc (they're even sometimes, but not always, spelled the same way!);
you just have to keep this in your head (and in your code.)

At the library level, developers have difficult choices.  The most fundamental
libraries -- collections and streams -- are prime examples of the tradeoffs that
library designers have to navigate.  Collections reasonably made the choice to
avoid specializing (there are libraries in the ecosystem, such as `trove` or
Eclipse Collections, that go the other way, and that's fine too), and streams
tried to walk a narrow line with hand-rolled specializations for `int`, `long`,
and `double`, but the existence of `IntStream` at all is evidence of the
contortions that library designers often have to twist themselves into.  Worse,
hand specialization begets more hand specialization (`IntStream` gave rise to
`IntToLongFunction` and `PrimitiveIterator.OfInt`, and there are always calls
for more ("Where's my `CharStream`?").)  And hand-specialization almost always
introduces asymmetries.  Finally, the mere existence of hand-specialized stream
types was a significant constraint on the design and implementation of the
library.

> Library designers are too often faced with the bad choice between good memory
behavior and good abstraction.

Users are not immune from having to reason about the gap between primitives and
objects either.  Nearly every Java developer has written an ad-hoc, hand-rolled
equivalent of `ArrayList<int>`, because `ArrayList<Integer>` is (perceived to
be) not good enough for the situation.  And this hand-rolled version rarely has
any connection to `List`, which often distorts any APIs that want to use it.
The tradeoff between good memory behavior and good abstraction hits users as
hard as it does library designers.

> The gap between primitives and objects is the original sin of Java; it's time
to address this.

#### The path forward

The path forward is to address the problem at the root -- the gap(s) between
primitives and classes.  Boxing purported to fill this gap, but in doing so, it
creates multiple new gaps of its own -- syntactic (having to convert between
`int` and `Integer`), semantic (converting back and forth doesn't always mean
the same thing, primarily because boxes have accidental object identity), and
performance (boxing is expensive, again primarily because of accidental object
identity.)  Autoboxing may paper over the syntactic gap, but the semantic and
performance gaps remain.  

Given that boxing seems to involved in all of the problems, it might be tempting
to say "then let's just get rid of boxing", but of course such "bold" thinking
skips over the hard part which is having a clear story of what to replace it
with.  Instead, we restore order by generalizing primitives to be full classes,
called _primitive classes_, but special classes that lack object identity.
(Finally, "everything is an object.")  Then we upgrade (or downgrade, depending
on your perspective) the existing primitives and their wrappers to be primitive
classes.  We can allow users to write their own primitive classes,  which have
the abstractive power of classes, but the performance of primitives.  Finally,
we  _generalize generics_, so that all types (including primitives) can be used
as type arguments, eliminating the need for hand-rolled specializations such as
`ToIntFunction` and `LongStream`.  The complete story will play out in phases.

#### The root cause: object identity

We've already cited a number of differences between objects and primitives:

 - Objects can only be accessed through an indirection known as an _object
   reference_, whereas primitives are accessed and stored directly ("by value");
 - Object references can be null, whereas primitives cannot;
 - Objects have identity, whereas primitives do not;
 - Objects can be polymorphic by extending classes or implementing interfaces,
   whereas primitives cannot;
 - Classes can have members (methods and fields), whereas primitives cannot;
 - Classes can have mutable state, whereas primitives cannot.

Given this array of differences, it seems daunting to try and unify primitives
with classes.  But most of these differences stem from a single root cause --
_object identity_.  If we can incorporate object identity (or the lack thereof)
more explicitly into the object model, we can derive most of these behaviors
from a unified view of classes and objects.

Object identity exists primarily to serve mutability and polymorphism (even
though not all classes will avail themselves of these features.)  The need to
support these essentially forces us to keep objects "at arms length", only
interacting with them through _object references_.  Null is not an instance of
any class; nullability is a property of accessing instances through references,
and it is the notion of object reference that introduces nullability.

We can reframe our view of classes and objects to support the notion that not
all classes intend for their instances to have object identity.  Whether or not
instances have identity can be independent of many other features of classes,
such as fields, methods, type variables, supertypes, etc.

If we take away the assumption that all objects have identity, this opens up a
new representational possibility; objects without identity can be represented
directly, or inline, as primitives are today.  On the other hand, the same
objects can be represented indirectly, or via references, as the primitive
wrappers are today.  In this view, the primitives we know today are analogous to
a direct representation of an instance of an identity-free class, and the
corresponding primitive wrapper primitive wrapper is analogous to an indirect
representation of _that same instance_.  What was a wholesale conversion to an
unrelated form, can be turned into the simpler difference between an indirect or
direct representation of the same instance.  (Most of the time, we will want to
leave the choice of representation to the runtime, but in some cases it may be
desirable to explicitly control it.) This is more semantically transparent (no
accidental identity is created or discarded), and the JVM can optimize this far
more effectively.

## Primitive classes

We call these classes whose instances have no identity _primitive classes_.
(This name may take some getting used to, since we're so used to thinking of
primitives as being the eight built-in types.)  Instances of primitive classes
are then called _primitive objects_.  (And so we finally arrive at the OO
nirvana of "everything is an object", only 25 years later.) The dichotomy of
"primitives vs classes" gives way to "primitive classes vs identity classes."
Having given up identity, the JVM is free to _inline_ (flatten) their layout
into other object and array layouts, and in general, represent or pass them by
value.

```
primitive class Point {
    int x;
    int y;

    public Point(int x, int y) {
        this.x = x;
        this.y = y;
    }
}
```

Primitive classes are implicitly `final`, cannot be `abstract`, can only extend
`Object` and certain abstract classes, and their fields are implicitly `final`.
(Another way to say this is that primitive classes model _values_.)  They can
still have almost anything else classes do -- interfaces, type variables, static
and instance fields, constructors, static and instance methods, nested classes,
etc.  Member inheritance works exactly the same way for primitive classes as for
identity classes.

At runtime, the behavior of certain core operations have been generalized to
distinguish between identity and primitive objects.  The `==` operator asks
whether the two operands are references to the same object.  We can generalize
this by observing that `==` performs a _substitutibility test_ on objects -- it
asks whether there is any way to distinguish the two objects.  Now that
primitives are objects too, we need to extend this test to cover them.  And this
is fairly straightforward: two primitive objects are considered `==` if they are
instances of the same class with the same state.  (For classes modeling values,
there is no upside -- in fact, there is downside -- to having multiple distinct
instances that model the number "3".)  Similarly, the `identityHashCode`
operation (used in the implementation of `Object::hashCode`) computes the hash
code for primitive objects based on their state rather than their identity.

There was a sensible way to totalize equality and hashing to incorporate
primitive classes, but not all operations are so lucky.  For example, totalizing
synchronization is far more difficult (and far less useful), so we will simply
say that synchronization on primitive objects is prohibited.

#### Types, and terminology

The current situation allows us to be relatively sloppy in our terminology,
because primitives and classes are different in almost every way.  The
distinctions between objects and references to those objects, or between classes
and types, are often blurred.  Similarly, "reference type" and "class types" and
"object types" mean (almost) the same thing and are often used interchangeably
with little loss of precision.  In the new world, we'll need some more precise
terminology, because two significant things have changed:

 - not all objects have identity
 - not all objects must be referred to by reference (though all can)

Classes are either _identity classes_ or _primitive classes_; instances of
classes are either _identity objects_ or _primitive objects_.  Identity objects
can only be described by reference; primitive objects can either be described
directly (by value) or by reference.  

Classes give rise to _types_; when we declare `class C`, this gives rise to a
type called `C` which consists of references to instances of `C`.  In Java 1.0,
classes and types existed in 1:1 relation; when generics were added, a single
class `C<T>` gave rise to an infinite family of types (`C<String>`,
`C<Integer>`, `C<?>`.)  We now extend this one-to-many relationship further, by
saying that a primitive class gives rise to two (families of) types, one
consisting of primitive objects, and the other consisting of references to those
primitive objects.  We call these _primitive value types_ and _primitive
reference types_.  Most of the time, we'll want the primitive value type, so for
a primitive class `P`, the type `P` will be the primitive value type
corresponding to `P` -- but in some cases we may want to specify this more
explicitly.  The term _reference type_ continues to mean any type which consists
of references to objects, which now also includes the primitive reference types.
Reference types continue to be nullable; primitive value types are not nullable.
The eight legacy primitives will be called the _built-in primitives_ going
forward.

The terminology of value vs reference type is intended to evoke the familiar
distinction between _pass by value_ and _pass by reference_, but is generalized
to cover not only passing of values between methods, but the storage properties
of variables (whether in the heap, on the stack, or in registers) as well.

For a primitive class `P`, we will denote the corresponding primitive value
class by `P.val` and the corresponding primitive reference class by `P.ref`, and
`P` will an alias for `P.val` (in most cases.)  It is only in rare cases that
code will have to explicitly use `.ref` and `.val`.

There is an automatic widening conversion from `P.val` to `P.ref` (semantically
similar to boxing) and a narrowing conversion from `P.ref` to `P.val`
(semantically similar to unboxing.)

The slogan for primitive classes is _codes like a class, works like an int_.

#### Boxing is dead; long live boxing

It is sensible to ask at this point, why do we need the primitive reference
types at all?  Why not just have primitive value types, and reference types
only for identity classes?

There are several reasons why references to primitive objects are important to
have, even if they will be used somewhat rarely.  

 - **Nullability.**  Nullability is a property of object references, not objects
   themselves.  Most of the time, it makes sense for primitive classes to be
   non-nullable (as the primitives are today), though there may be situations
   where null is a semantically important value, but for which we want to
   disavow identity (and gain the benefits that flow from that.)  Using `P.ref`
   when nullability is required is semantically clear, and avoids the need to
   invent new sentinel values for "no value."

 - **Non-flattening is sometimes desirable.**  Most primitive classes will be
   relatively small aggregates, such as the `Point` example above.  However,
   some may be larger.  If we have an array of a "large" primitive class, the
   memory footprint of the array will be proportionally larger than of an array
   of the corresponding reference type.  This may not always be what is desired;
   if the array is sparse, this creates undue memory pressure, and even if not,
   sorting such an array may actually be slower, since swapping two elements
   involves moving much more data than swapping two references.  Sometimes we
   really want an array of `P.ref`.  (Similarly, when generics are eventually
   specialized, we may wish to explicitly specify that we don't want to inline
   the representation of the type parameter into the resulting type;
   `Foo<P.ref>` is a natural way to specify that.)

 - **Interoperation with Object and interface types.**  If primitive classes can
   implement interfaces, than an interface type is going to be polymorphic over
   both identity and primitive objects.  Interfaces achieve this polymorphism
   through object references, so in order to bring primitive objects under the
   umbrella of interfaces, we need to be able to represent them with object
   references.  The special top type `Object` is the same; it is often helpful
   to think of `Object` as an "honorary interface."

 - **Self-reference.**  Some types, such as the "next" field in the "node" type
   for a linked list, may want to directly or indirectly refer back to the type
   of the enclosing type, but circularities in the definition of primitive
   classes are not allowed (because that would make their layout of
   indeterminate size).  This can be done via a primitive reference type:

```
primitive class Node<T> {
    T element;
    Node.ref<T> next;
}
```

 - **Compatible migration.**  Some classes, such as `Optional`, are good
   candidates for migration to being primitive classes, and we would like for
   this to be a binary- and source-compatible change.  The type `Optional` today
   consists of references to identity objects; to make this a compatible
   migration, we would have to make the type `Optional` tomorrow consist of
   references to primitive objects.  We don't want to burden all primitive
   classes with the additional weight necessary to support compatible migration;
   we want only the migrated classes to bear this burden.  

 - **Compatibility with existing boxing.**  Autoboxing is convenient, in that it
   lets us pass a primitive where a reference is required.  But boxing affects
   far more than assignment conversion; it also plays into method overload
   selection, for example.  The rules are carefully design to prefer overloads
   that require no conversions to overloads that require boxing (or varargs)
   conversions.  Having both a value and reference type for every primitive
   class means that these rules can be cleanly and intuitively extended to cover
   user-written primitive classes.

It may appear that all we did here was to "rename" boxing in a complicated way.
And if we did this entirely through compiler trickery, that would be true.  What
has changed here is that the JVM has also been upgraded to understand the notion
of primitive classes, and the relationship between primitive value types and
their corresponding primitive reference type (and more), so that user-written
primitives can have the performance of the built-in ones, and this "new boxing"
has much better performance than the old boxing.  This mostly comes down to the
JVM understanding that primitive objects do not have object identity, and
therefore the JVM can more easily optimize how they are stored in memory, how
their method invocations are dispatched, and how they are passed between methods
(such as by scalarization.)

> Rather than back away from boxing, we double down on it by making it more
> regular, and faster.

#### Rationalizing arrays

If `C extends D` (or `C implements D`), then `C[] <: D[]` -- this is called
_array covariance_.  But, there are some gaps when it comes to arrays of
primitives.  We can box an `int` to an `Integer`, but we can't box an `int[]` to
an `Integer[]`.  `Object[]` is the top array type -- because of covariance --
but only for arrays whose components are classes, not for primitives, because
primitives do not (today) extend `Object`.   When primitives become classes, we
can fix that.  For a primitive class `P` that extends (or implements) `D`, `P[]
<: P.ref[]`, and `P[] <: D[]`.  

#### Migration from identity classes to primitive classes

Some classes, such as `Optional`, were originally implemented as identity
classes, but with the intention that some day they could become primitive
classes.  (These classes often include a disclaimer in their specification that
they are
[_value-based_](https://docs.oracle.com/javase/8/docs/api/java/lang/doc-files/ValueBased.html);
this captures most of the requirements for a compatible migration to primitive
classes.)

However, to the extent these classes appear in APIs (such as the return type of
a method), the existing semantics are that of reference types, not primitive
types; it is allowable for an `Optional` to be `null`.  So when these classes
are migrated to primitives, we must arrange that the old name corresponds to the
primitive reference type, rather than corresponding to the primitive value type
(which is the sensible default for new code.)  We might declare this as:

```
primitive-reference class Optional<T> { ... }
```

The only difference between a primitive class and a `primitive-reference` class
is that the undecorated name -- `Optional` -- is an alias for `Optional.ref`
rather than `Optional.val`.  This allows such a migration to be source- and
binary-compatible; implementations would likely want to use `Optional.val`
internally for describing variables to get maximal flattening and density, but
the API could continue to work in terms of `Optional`, and the widening and
narrowing conversions would make up the difference.

#### Migrating the built-in primitives

There are a few rough edges we need to file down in order to model the existing
built-in primitive types as primitive classes.  We would like to be able to
declare  `Integer` as an ordinary (migrated) primitive class, just as we did
with `Optional`:

```
primitive-reference class Integer { ... }
```

Because of the ad-hoc relationship between `int` and `Integer`, we need to do a
little more.  The type `int` becomes an alias for `Integer.val`, and `int.ref`
becomes an alias for `Integer`.  

Another aspect of "special pleading" required for this migration is that the
eight primitive wrapper types are allowed to refer to their corresponding
built-in primitive type in their declaration; ordinarily this sort of
circularity would be banned.

There is one significant price to pay for this migration: synchronization on
instances of `Integer` will no longer be supported, instead throwing
`IllegalMonitorStateException`.  This is not a behaviorally compatible change,
but based on analysis of existing codebases, synchronizing on wrappers is rare
-- and usually a mistake.  While we do not take making any incompatible change
lightly, overall this is a reasonable price for unifying primitives with
objects.

#### Decoder chart

This approach unifies primitives with objects, by allowing for the fact that
some classes lack object identity, and brings us to a world where "everything is
an object", with the JVM able to optimize primitive objects more effectively.
Of course, having two kinds of classes is still a distinction, even if primitive
classes and identity classes are now closer together.  The following table
summarizes the transition.

| Current World                                  | Valhalla                                                     |
| ---------------------------------------------- | ------------------------------------------------------------ |
| entities: objects and primitives               | primitive and identity objects                               |
| types: reference and primitive types           | reference types and primitive value types                    |
| fixed set of primitives                        | open-ended set of primitives                                 |
| primitives have boxes                          | primitive objects can be described by references             |
| boxes have identity, are visible as getClass() | primitive references have no identity                        |
| boxing and unboxing conversions                | primitive reference and value conversions, but same rules    |
| primitives are built in and magic              | primitives are mostly just primitive classes with VM support |
| primitives don't have methods, supertypes      | primitives are classes, have methods & supers                |
| primitive arrays are monomorphic               | arrays are covariant                                         |

## Unified generics

The above outlines a story for unifying primitives and classes, while retaining
(and expanding) the memory efficiency benefits of primitives.  But there is
still another gap to be addressed -- that generics cannot have primitive types
as their type parameters.  This is a problem even just with the eight built-in
primitives, and, when the domain of primitive types grows, will become an even
bigger problem.

Our strategy is to address this problem in two phases.  The first phase is a
unified syntactic and semantic treatment of generics, where any type can be used
as a type parameter; you can say `Foo<String>` or `Foo<int>` or `Foo<P>` without
restriction.  The second phase pushes awareness of generics into the JVM, so
that object layout, code bodies, and calling conventions for generic classes
instantiated over primitive classes can be specialized for those instantiations.

The details will be provided in a separate document.

<!--
then, about them going hand in hand - we start the "double down on erasure" innuendo
pointing out that every other alternative would not seal the rift, it would in fact make it worse


- The generics we have
  - erasure
  - no primitives

While programmers may curse erasure, the second is actually an enormously bigger
problem.  Programmers might thnink the big problem is perf, but really the big
problem is expressiveness.

[ inlines are layout- and calling-friendly classes ]
[ ]

[ templates vs reified generics vs optimized generics ]
[ expressiveness goals ]
[ double down on erasure ]
[ double down on boxing ]
[ Object is the new Any ]

-->
