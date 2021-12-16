# State of Valhalla

#### Section 2: Language model
#### Brian Goetz, Dec 2021

::: sidebar
Contents:

1. [The Road to Valhalla](01-background.html)
2. [Language model](02-object-model.html)
3. [JVM Model](03-vm-model.html)

:::

This document describes the directions for the Java _language_ charted by
Project Valhalla.  (In this document, we use "currently" to describe the
language as it stands today, without value classes or extended primitives.)

Valhalla started with the goal of providing user-programmable classes which
could be flat and dense in memory.  Numerics are one of the motivating use
cases, but adding new primitive types directly to the language has a very high
barrier.  As we learned from [Growing a Language][growing] there are infinitely
many numeric types we might want to add to Java, but the proper way to do that
is as libraries, not as language features.

## Primitive and reference types in Java today

Java currently has eight built-in primitive types (`void` is not a type).
Primitives represent pure _values_; any `int` value of "3" is equivalent to (and
indistinguishable from) any other `int` value of "3".  Values are atomic (its
parts cannot be updated individually) and have no canonical location, and so are
_freely copyable_.  With the exception of the unusual treatment of `NaN` values
for `float` and `double`, the `==` operator asks "are these two values the same
value".

Java also has _objects_, and each object has a unique _object identity_. Because
of identity, objects are not freely copyable; each object lives in exactly one
place (at any given time), and to access its state, we have to go to that place.
But we mostly don't notice this because objects are not manipulated or accessed
directly, but instead through _object references_.  Object references are also
values -- they encode the identity of the object to which they refer, and the
`==` operator on object references asks "do these two references refer to the
same object."  Accordingly, object _references_ can be freely copied, but the
objects themselves cannot.  

Primitives and objects differ in almost every conceivable way:

| Primitives                                 | Objects                            |
| ------------------------------------------ | ---------------------------------- |
| No identity (pure values)                  | Identity                           |
| `==` compares values                       | `==` compares object identity      |
| Built-in                                   | Declared in classes                |
| Not nullable                               | Nullable                           |
| No members (fields, methods, constructors) | Members (including mutable fields) |
| No supertypes or subtypes                  | Class and interface inheritance    |
| Accessed directly                          | Accessed via object references     |
| Default value is zero                      | Default value is null              |
| Arrays of primitives are monomorphic       | Arrays are covariant               |
| Tearable under race                        | Initialization safety guarantees   |
| Have reference companions                  | Don't need reference companions    |

The design of primitives represents various tradeoffs aimed at maximing
performance and usability of the primtive types.  Reference types default to
`null`, meaning "referring to no object"; primitives default to a usable zero
value (which for most primitives is the additive identity.)  Reference types
provide initialization safety guarantees against a certain category of data
races; primitives allow tearing under race for larger-than-64-bit values.  

The following figure illustrates the universe of Java's current types.  The
upper left quadrant is the built-in primitives; the rest of the space is
reference types.  In the upper-right, we have the abstract reference types --
abstract classes, interfaces, and `Object` (which usually acts more like an
interface than a concrete class.)  The built-in primitives have reference
companions (wrappers or boxes).

FIGURE 1 HERE

Valhalla unifies primitives and objects in that that they can both be declared
with classes, but maintains the special runtime characteristics primitives have.
But as we explored the design space, we found ourselves pulled in multiple
directions; while everyone likes the flatness and density that user-definable
value types promise, in some cases we want them to be more like classical
objects (nullable, non-tearable), and in other cases we want them to be more
like classical primitives.  It was clear that there was not a one-size-fits-all
answer.  

## Value classes: separating references from identity

Many of the impediments to optimization that Valhalla seeks to remove center
around _unwanted object identity_.  The primitive wrapper classes have identity,
but not only is this identity not directly useful, it can be a source of bugs.
(For example, due to caching, integers can be accidentally compared correctly
with `==` just often enough that people keep doing it.)  Similarly, [value-based
classes][valuebased] such as `Optional` have no need for identity, but pay the
costs of having identity anyway.  

Our first step is allowing class declarations to explicitly disavow identity, by
declaring themselves as _value classes_.  The instances of a value class are
called _value objects_.  

```
value class ArrayCursor<T> { 
    T[] array;
    int offset;

    public ArrayCursor(T[] array, int offset) { 
        this.array = array;
        this.offset = offset;
    }

    public boolean hasNext() { 
        return offset < array.length;
    }

    public T next() { 
        return array[offset];
    }

    public ArrayCursor<T> advance() { 
        return new ArrayCursor(array, offset+1);
    }
}
```

This says that an `ArrayCursor` is a class whose instances have no identity.  As
a consequence, they must give up the things that depend on identity; they and
their fields are implicitly final.  

But, value classes are still classes, and can have most of the things classes
can have -- fields, methods, constructors, type parameters, superclasses (with
some restrictions), nested classes, class literals, interfaces, etc.  They can
only extend certain classes: `Object` or abstract classes with no fields, empty
no-arg constructor bodies, no other constructors, no instance initializers, no
synchronized methods, and whose superclasses all meet this same set of
conditions. (`Number` is an example of such a class.)

Value class types are still reference types; we refer to value objects via
object references.  This means that object references can refer to either
identity objects or value objects; for the types in the upper-right quadrant
(interfaces, abstract classes, and `Object`), references to these types might
refer to either an identity object or a value object.  (Historically, JVMs were
effectively forced to represent object references with pointers; for references
to value objects, JVMs now have more flexibility.)

Because they are reference types, value class types are nullable, their default
value is null, and loads and stores of references are atomic even in the
presence of data races, providing the initialization safety we are used to with
classical objects.

Because they are values, `==` compares value objects by state rather than
identity.  This means that value objects, like primitives, are _freely
copyable_; we can explode them into their fields and re-aggregate them into
another value object, and we cannot tell the difference.  (Because they have no
identity, some identity-sensitive operations, such as synchronization, produce
runtime exceptions.)

Value classes take aim at the first two lines of the table of differences above;
rather than identity being a property of all objects, it becomes just another
user-declared property of classes, such as finality or mutability.  Many classes
don't make use of identity, but all classes pay for it in a variety of ways.  By
allowing classes that don't need identity to exclude it, we free the runtime to
make better layout and compilation decisions -- and avoid a whole category of
bugs.

In looking at the code for `ArrayCursor`, we might mistakenly assume it will be
inefficient, as each loop iteration appears to allocate a new cursor:

```
for (ArrayCursor<T> c = Arrays.cursor(array); 
     c.hasNext(); 
     c = c.advance()) {
    // use c.next();
}
```

One should generally expect here that _no_ cursors are actually allocated.
Because an `ArrayCursor` is just its two fields, these fields will get hoisted
into registers, and the constructor call in `advance` will typically compile
down to incrementing one of these registers.

#### Migration

The JDK (as well as other libraries) has many [value-based classes][valuebased],
such as `Optional` and `LocalDateTime`.  Value-based classes adhere to the
semantic restrictions of value classes, but are still identity classes -- even
though they don't want to be.  Value-based classes can be migrated to true value
classes simply by redeclaring them as value classes.  This is both source- and
binary-compatible.

We plan to migrate many value-based classes in the JDK to value classes.
Additionally, the primitive wrappers can be migrated to value classes as well,
making the conversion from `int` to `Integer` cheaper.  (In some cases, this
may be _behaviorally_ incompatible for code that synchronizes on the primitive
wrappers.  [JEP 390][jep390] has supported both compile-time and runtime
warnings for synchronizing on primitive wrappers since Java 16.)  

FIGURE 2: ADD NEW QUADRANT FOR VC (with Integer as VC)

#### Equality

Earlier we said that `==` compares value objects by state rather than by
identity.  More precisely, two value objects are `==` if they are of the same
type, and each of their fields are pairwise equal, where equality is given by
`==` for primitives (except `float` and `double`, which are compared with
`Float::equals` and `Double::equals` to avoid the `NaN` anomalies), `==` for
references to identity object objects, and recursively with `==` for references
to value objects.  In no case is a value object ever `==` to an identity object.

#### Value records

While records have a lot in common with value classes -- they are final and
their fields are final -- they are still identity classes.  Records embody a
tradeoff: give up on decoupling the API from the representation, and in return
get various syntactic and semantic benefits.  Value classes embody another: 
give up identity, and get various semantic and performance benefits.  If
we are willing to give up both, we can get both sets of benefits:

```
value record NameAndScore(String name, int score) { }
```

Value records combine the data-carrier idiom of records with the improved 
scalarization and flattening benefits of value classes.  

In theory, it would be possible to apply `value` to certain enums as well, but
this is not currently possible because the `java.lang.Enum` base class that
enums extend do not meet the requirements for superclasses of value classes (it
has fields and non-empty constructors.)

## Extended primitives

Value classes allow developers to give up one thing -- identity -- and gain a
host of performance benefits.  They are an ideal replacement for many of today's
value-based classes, fully preserving their semantics.  But they represent one
point a spectrum of tradeoffs between abstraction and performance, and other
desired use cases -- such as numerics -- may want a different set of tradeoffs.

Specifically, value classes are still _reference types_.  This means they are
nullable, and therefore must account for null somehow in their representation,
which may have a footprint cost.  Similarly, they still offer the initialization
safety guarantees that we come to expect from classes, which also has a cost to
preserve.  For certain use cases, it may be desire to additionally give up
something else to gain the maximum flatness and density that we get -- and that
something else is reference-ness.

The built-in primitives are best understood as pairs of types: the primitive
type and its reference companion (its wrapper or box.)  If we need the
affordances of reference-ness (subtyping with `Object` or interfaces, dynamic
type tests with `instanceof` or pattern matching, use as type parameters,
self-referential layout), we use the reference companion to access these
affordances; the rest of the time we use the primitive type.  The reference
companion is (or, will be) a value class; it has no need of identity.  

_Extended primitives_ allow us to define new pairs of types that share this
behavior:

```
primitive Point implements Serializable {
    int x;
    int y;

    Point(int x, int y) { 
        this.x = x;
        this.y = y;
    }

    Point scale(int s) { 
        return new Point(s*x, s*y);
    }
}
```

This gives rise to _two_ types: a primitive type `Point`, and a reference
companion type `Point.ref`, which is a value type.  They have the same set of
members (fields and methods), and the same implicit conversions between them as
primitives do today with their boxes.  The default value of the primitive is
just the default value of all its fields; the default value of the reference
companion is, like all reference types, null.  (This means that there is always
exactly one more observable value for the reference companion than for the
primitive -- null -- just like primitives today.)  The constraints on declaring
a primitive are essentially the same as for value classes (e.g., a `primitive`
can have an `extends` clause, subject to the same superclass constraints as for
value classes).  Similarly, the fields of a `primitive` are implicitly final.

In our diagram, these new types show up as another entity that straddles the
line between primitives and references, alongside the legacy primitives: 

FIGURE 3: ADD EXTENDED PRIMITIVES

#### Member access

When we declare a primitive, we are simultaneously declaring both the primitive
and its reference companion, and both have the same members.  Unlike today,
primitives can be used as receivers to access fields and invoke methods (modulo
accessibility): 

```
Point p = new Point(1, 2);
assert p.x == 1;

p = p.scale(2);
assert p.x == 2;
```

#### Polymorphism

When we declare a class today, we set up a subtyping (is-a) relationship between
the declared class and its supertypes (we write `A <: B` to indicate A is a
subtype of B.)  When we declare a primitive, we set up a subtyping relationship
between the _reference companion_ and the declared supertypes.  This means that 
if we declare:

```
primitive UnsignedShort extends Number 
                        implements Comparable<UnsignedShort> { 
   ...
}
```

then `UnsignedShort.ref <: Number` and `UnsignedShort.ref <:
Comparable<UnsignedShort>`.  What happens if we ask such a question of 
the primitive type?

```
UnsignedShort us = ...
if (us instanceof Number) { ... }
```

Since subtyping is defined only on reference types, the `instanceof` operator
will behave as if the right-hand side is converted to its reference companion,
and then the question can be answered in the affirmative.  (This may trigger
fears of expensive boxing conversions, but in reality no actual boxing is
required.)

We introduce a new relationship based on `extends` / `implements` clauses, which
we'll call "extends"; we define `A extends B` as meaning `A <: B` when A is a
reference type, and meaning `A.ref <: B` when A is a primitive type.  The
`instanceof` relation, reflection, and pattern matching are updated to use
"extends".

#### Arrays

Arrays of reference types are _covariant_; this means that if `A <: B`, then
`A[] <: B[]`.  This allows `Object[]` to be the "top array type", at least for
arrays of references.  But arrays of primitives are currently left out of this story.  
We can unify the treatment of arrays by defining array covariance over the new
"extends" relationship; if A extends B, then `A[] <: B[]`.  This means that for
a primitive P, `P[] <: P.ref[] <: Object[]`.

#### Equality

Just was with `instanceof`, we define `==` on primitives by appealing to the
reference companion (though no actual boxing need occur.)  Evaluating `a == b`,
where either or both operands are primitives, can be defined as if the primitive
operands are first converted to their reference companions, and then the
comparison is performed.  This means that:

```
Point p = new Point(3, 4);
Point.ref pr = p;
assert p == pr;
```

The base implementation of `Object::equals` is to delegate to `==`; for a
primitive class that does not explicitly override `Object::equals`, this is the
default we want.  


#### Serialization

If a `primitive` implements `Serializable`, this is really a statement about the
reference companion.  Just as with other aspects described here, serialization of
primitives can be defined by converting to the reference companion and serializing
that, and reversing the process at deserialization time.

Serialization currently uses object identity to preserve the topology of an
object graph.  This generalizes cleanly to objects without identity, because
`==` on value objects treats two identical copies of a value object as equal.  
So any observations we make about topology prior to serialization, are
consistent with those after deserialization.

#### Default values

For a value class `C`, the default value of variables of type `C` is the same as
any other reference type: `null`, and the same is true for the reference
companion `P.ref` of a primitive `P`.  For the primitive type `P` itself, the
default value is the one where all of its fields are initialized to their
default value.

Since the default value for all built-in primitives is some form of zero, and
the default value for all reference types is `null` (also a form of zero), the
default value for an extended primitive is just the all-zeroes value of its
representation.  

The built-in primitives reflect the design assumption that zero is a reasonable
default.  Similarly, if we choose to model an entity with an extended primitive,
we are making the same assumption: that the zero representation is a reasonable
default.  

For some abstractions, such as `LocalDate`, there _is_ no reasonable default
other than `null`.  If we choose to represent a date as the number of days since
some epoch, there will invariably be bugs that stem from uninitialized dates;
we've all been mistakenly told by computers that something will happen on or
near 1 January 1970.  Even if we could choose a default other than the zero
representation, an uninitialized date is still likely to be an error.  (For this
reason, `LocalDate` is better suited to being a value class than a primitive.)

The choice to use a zero default instead of null was one of the central
tradeoffs in the design of the built-in primitives.  It gives us a usable
initial value (most of the time), and requires less storage footprint than a
representation that supports null (`int` uses all 2^32 of its bit patterns, so a
nullable `int` would have to either make some 32 bit signed integers
unrepresentable, or use a 33rd bit.)  This was a reasonble tradeoff for the
built-in primitives, and is also a reasonable tradeoff for many (but not all)
other types (such as complex numbers, 2D points, half-floats, etc.)

#### Tearing

For the primitive types longer than 32 bits (long and double), it is not
guaranteed that reads and writes from different threads (without suitable
coordination) are atomic with respect to each other.  The result is that, if
accessed under data race, a long or double field or array element can be seen to
"tear", and a read might see the low 32 bits of one write, and the high 32 bits
of another.  (Declaring the containing field `volatile` is sufficient to restore
atomicity, as is properly coordinating with locks or other concurrency control.)

This was a pragmatic tradeoff given the hardware of the time; the cost of
atomicity on 1995 hardware would have been prohibitive, and problems only arise
when the program already has data races -- and most numeric code deals with
thread-local data.  Just like with the tradeoff of nulls vs zeros, the design of
the built-in primitives permits tearing as part of a tradeoff between
performance and correctness, where primitives chose "as fast as possible" and
objects chose more safety.

Today, most JVMs give us atomic loads and stores of 64-bit primitives, because
this is now cheap enough.  But extended primitives bring us back to 1995; atomic
loads and stores of larger-than-64-bit values are still expensive, leaving us
with a choice of "make operations on primitives slower" or permitting tearing
when accessed under race.  For extended primitives, we choose to mirror the
behavior of existing primitives, and permit tearing under race.

Just as with null vs zero, this choice has to be made by author of a class.  
For classes like `Complex`, all of whose bit patterns are valid, this is very
much like the choice around `long` in 1995.  For other classes that might have
nontrivial representational invariants, these may be better off choosing value
classes, which offer tear-free access because loads and stores of references are
atomic.

#### Legacy primitives

As part of generalizing primitives, we want to adjust the built-in primitives to
behave as consistently with extended primitives as possible.  While we can't
change the fact that `int` boxes to `Integer`, we can give `Integer` a better
alias -- `int.ref` -- so that we can use a single rule for naming the reference
companion.  Similarly, we can extend member access to the legacy primitives, and
treat `int[]` as being a subtype of `Integer[]`.  

## Why a reference companion at all?

It is common to ask: why do we need a reference companion at all?  The need for
reference companions is analogous to the need for boxes in 1995: we'd made a set
of tradeoffs, favoring performance, for primitives: they are non-nullable, their
default is zero, they can tear under race, they are unrelated to `Object`, etc.
Most of the time, we ignored the box types, but sometimes, we needed to
temporarily suppress one of these properties, such as interoperating with code
that expects an `Object`.  The reasons we needed boxes in 1995 still apply to
extended primitives; most of the time, we will deal with them as primitives, but
sometimes we need the properties of references (nullability, non-tearability
under race, polymorphism), and in those cases, we appeal to the reference
companion.  The expectation is that using `P.ref` will be about as rare as using
`Integer` explicitly today.

Reasons we might have to appeal to the reference companion include: 

 - **Interoperation with reference types.**  If primitive classes can implement
   interfaces and extend classes (including `Object` and some abstract classes),
   then some class and interface types are going to be polymorphic over both
   identity and primitive objects.  This polymorphism is achieved through object
   references; a reference to `Object` may be a reference to an identity object,
   or a reference to a value object.  

 - **Nullability.**  Nullability is an affordance of object _references_, not
   objects themselves.  Most of the time, it makes sense that primitive types
   are non-nullable (as the primitives are today), but there may be situations
   where null is a semantically important value.  Using `P.ref` when nullability
   is required is semantically clear, and avoids the need to invent new sentinel
   values for "no value."

   This need may come up when migrating existing classes, such as the method
   `Map::get`, which uses `null` to signal that the requested key was not
   present in the map.  But, if the `V` parameter to `Map` is a primitive class,
   `null` is not a valid value.  We can capture the "`V` or null" requirement
   by changing the descriptor of `Map::get` to be:

```
public V.ref get(K key);
```

   where, whatever type `V` is instantiated as, `Map::get` returns the reference
   companion. (For a type `V` that already is a reference type, this is just the
   instantiation of `V`.) This captures the notion that the return type of
   `Map::get` will either be a reference to a `V`, or the `null` reference.
   (This is a compatible change, since both erase to the same thing.)


 - **Self-referential types.**  Some types may want to directly or indirectly
   refer back to the type of the enclosing type; an example of such a situation
   is the "next" field in the "node" type of a linked list:

```
class Node<T> {
    T theValue;
    Node<T> next;
}
```

   We might want to represent this as a primitive, but then the layout of `Node`
   would be self-referential, and since we want to flatten primitives into the
   layout of their enclosing types, this would lead to an infinite regress.  The
   solution is to explicitly opt for a reference, where we can use `null` to 
   indicate that there is no next node:

```
primitive Node<T> {
    T theValue;
    Node.ref<T> next;
}
```

 - **Protection from tearing.**  We may want to use the reference companion when
   we are concerned about tearing; we can use `P.ref` as a field or array
   component type to request reference semantics; because loads and stores of
   references are atomic, `P.ref` is immune to the tearing under race that `P`
   might be subject to.

 - **Compatibility with existing boxing.**  Autoboxing is convenient, in that it
   lets us pass a primitive where a reference is required.  But boxing affects
   far more than assignment conversion; it also affects into method overload
   selection.  The rules are carefully design to prefer overloads that require
   no conversions to overloads that require boxing (or varargs) conversions.
   Having both a value and reference type for every primitive class means that
   these rules can be cleanly and intuitively extended to cover user-written
   primitive classes.

#### Identity-sensitive operations

Certain operations are currently defined in terms of object identity.  As we've
already seen, some of these, like equality, can be sensibly extended to cover
all object instances.  Others, like synchronization, will become partial.  These
identity-sensitive operations include:

  - **Equality.**  We extend `==` on references to include references to value
    objects.  Where it currently has a meaning, the new definition coincides
    with that meaning.
  - **System::identityHashCode.**  The main use of `identityHashCode` is in the
    implementation of data structures such as `IdentityHashMap`.  We can extend
    `identityHashCode` in the same way we extend equality -- deriving a hash on
    primitive objects from the hash of all the fields.
  - **Synchronization.**  This becomes a partial operation.  If we can
    statically detect that a synchronization will fail at runtime (including
    declaring a `synchronized` method in an primitive class), we can issue a
    compilation error; if not, attempts to lock on a primitive instance results in
    `IllegalMonitorStateException` at runtime.  This is justifiable because it
    is intrinsically imprudent to lock on an object for which you do not have a
    clear understanding of its locking protocol; locking on an arbitrary
    `Object` or interface instance is doing exactly that.
  - **Weak references.**  If we made creating weak references a partial
    operation on `Object`, weak references become almost useless, as every class
    that wants to maintain some sort of weak data structure would have to
    bifurcate into separate paths for identity and primitive objects.  (This would
    be similar to partializing `identityHashCode`.)  Weak references to primitive
    objects that contain no references to identity objects should never be
    cleared; weak references to primitive objects that contain references to
    identity objects should be cleared when those objects are no longer strongly
    reachable.


#### Identifying identity

To distinguish between primitive and identity classes at compile and run time,
we introduce two restricted interfaces `IdentityObject` and `PrimitiveObject`.
`IdentityObject` is implicitly implemented by identity classes;
`PrimitiveObject`  is implicitly implemented by primitive classes; no class can
implement both.  This enables us to write code that dynamically tests for object
identity before performing identity-sensitive operations:

```
if (x instanceof IdentityObject) {
    synchronized(x) { ... }
}
```

as well as statically reflecting the requirement for identity in variable types
(and generic type bounds):

```
static void runWithLock(IdentityObject lock, Runnable r) {
    synchronized (lock) {
        r.run();
    }
}
```

If an interface or abstract class implements `IdentityObject` or
`PrimitiveObject`, this serves as a constraint that it may only be extended by
the appropriate sort of class.

#### What about Object?

The root class `Object` poses an unusual problem, in that every class must
extend it directly or indirectly, but itself is (currently) an identity class,
and it is common to use `new Object()` as a way to obtain a new object identity
for purposes of locking.  If `Object` were to implement `IdentityObject`, then
primitive classes could not extend `Object` (and therefore could not
interoperate with dynamically typed libraries such as reflection).  We address
this problem by treating `Object` like we do interfaces and certain abstract
classes -- they can be extended by both identity and primitive classes -- but
redefine the idiom `new Object()` to evaluate to a fresh instance of an
_anonymous identity subclass_ of `Object`.

#### Bringing primitives and objects closer together

While primitives and objects still have some differences, we can dramatically
reduce the size of the table of differences we started with.  Rather than
identity being a property of all objects, identity becomes a property objects
can opt into -- and the semantics of `==` follows from that choice.  Both
primitives and objects can be declared using classes.  We can give primitives
members, supertypes, and array covariance.  Which leaves us with a much smaller
set of differences: 

| Primitives                          | Objects                          |
| ----------------------------------- | -------------------------------- |
| Not nullable; default value is zero | Nullable; default value is null  |
| Tearable under race                 | Initialization safety guarantees |
| Have reference companions           | Don't need reference companions  |

## Value types vs primitives

It is reasonable to ask, why would we introduce _two_ new forms of declaration,
value classes and primitives?  Couldn't one or the other be good enough?  

While we could of course get away with only one of these (we've been getting
away with neither for 25 years), whichever one we picked would be unsatisfying
for some of the desired use cases.  If we picked value classes only, new numeric
types would be burdened by the requirement to represent nulls (which has a
footprint cost) and to manage atomicity of loads and stores.  If we picked
primitives only, it would be very tempting to use primitives even when they are
not entirely appropriate, and users would be stuck with an inconvenient default
value or with objects that cannot protect their invariants when accidentally
shared under a data race.  There's a reason for the remaining rows in our
primitives-vs-objects table; sometimes you want nulls, and sometimes not;
sometimes you can tolerate tearing to get maximum performance, and sometimes
not.  

How would we choose between declaring an identity class, value class, or
primitive?  Here are some quick rules of thumb: 

 - Use identity classes when we need mutability, extension, or locking;
 - Consider value classes when we don't need identity, but need nullity or have
   cross-field invariants; 
 - Consider primitives when we don't need identity, nullity, or cross-field
   invariants, and can tolerate the zero default and tearability that comes with
   primitives.

## Summary

[ unifies primitives and objects: declared with classes, extends Object ] 

The following table summarizes the transition from the current world to
Valhalla.

| Current World                             | Valhalla                                                     |
| ----------------------------------------- | ------------------------------------------------------------ |
| primitives values and objects             | primitive values and objects                                 |
| primitives and reference types            | primitive and reference types                                |
| all objects have identity                 | some objects have identity                                   |
| fixed set of primitives                   | open-ended set of primitives                                 |
| primitives have ad-hoc boxes              | primitives have regularized reference companions             |
| boxes have identity                       | reference companions have no identity                        |
| boxing and unboxing conversions           | primitive reference and value conversions, but same rules    |
| primitives are built in and magic         | primitives are mostly just primitive classes with VM support |
| primitives don't have methods, supertypes | primitives are classes, have methods & supers                |
| primitive arrays are monomorphic          | arrays are covariant                                         |


## Universal generics

 - Null pollution
 - Meaning of extends
 - Ref conversions
 - Type inference






Project Valhalla will subtly realign this division so that we can unify
primitives with classes, but maintain the runtime behavior of today's
primitives and support the declaration of new primitives.


#### Performance considerations



[ B2 faster due to noid ]
[ but still initialization-safe (nulls) and non-tearable ]
[ this has costs: footprint, limited heap flattening ]

[ primitives today: flat, dense, tearable, non-nullable, boxes ]








## Summary

This approach unifies primitives with objects, by allowing for the fact that
some classes lack object identity, and brings us to a world where "everything is
an object", with the JVM able to optimize primitive objects more effectively.
Of course, having two kinds of classes is still a distinction, even if primitive
classes and identity classes are now closer together.  The following table
summarizes the transition from the current world to Valhalla.

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



[valuebased]: https://docs.oracle.com/javase/8/docs/api/java/lang/doc-files/ValueBased.html
[growing]: https://dl.acm.org/doi/abs/10.1145/1176617.1176621
