# State of Valhalla

#### Section 3: JVM Model
#### Brian Goetz, Mar 2020

::: sidebar
Contents:

1. [The Road to Valhalla](01-background.html)
2. [Language Model](02-object-model.html)
3. [JVM Model](03-vm-model.html)
4. [Translation scheme](04-translation.html)
:::

This document describes the _Java Virtual Machine_ view of inline classes.  Note
that this is not necessarily the same as the Java Language view; readers are
advised to exercise care in drawing conclusions about the Java Language from
this document.

#### Inline classes

Prior to Valhalla, all objects -- instances of classes and arrays -- had a
unique _object identity_.  Valhalla allows classes to choose, by marking a class
with the `ACC_INLINE` flag, whether their instances have identity ("identity
classes") or not ("inline classes").  

In the abstract virtual machine (as described by classfiles), both inline and
identity objects are referred to exclusively through _object references_.
However, because the JVM knows that inline objects do not have identity, it can
routinely optimize layout (flattening), instantiation, member access, and
calling conventions (scalarization) for inline objects.

#### Carriers and basic types

JVM type descriptors (JVMS 4.3.2) use a leading letter to denote their _basic
type_.  There are currently eight basic types corresponding to the eight
primitive types (`I`, `J`, etc), and one basic type corresponding to object
references (`L`).  (The letter `V` appears in type descriptors but is not
considered a type.)

These nine basic types correspond to five _carrier types_ (int, long, float,
double, and object reference), which correspond to different representations of
a value in a stack slot or local variable slot.  (The difference between these
two sets comes from erasing `byte`, `short`, `char`, and `boolean` to `int` and
using the `I` carrier.)  The primitive carriers store a value directly in the
stack or local variable slot (long and double values use two adjacent slots),
and the object reference (`L`) carrier stores an _object reference_ in the
corresponding slot.

To describe inline types, we add a new basic type, `Q`, which denotes a
reference to an inline object.  `Q` descriptors have the same syntactic
structure as `L` descriptors (e.g., `Qjava/lang/int;`).  

In addition to reusing the syntactic form of `L` descriptors, `Q` descriptors
also reuse the `L` carrier -- in the JVM, there is no structural difference
between a reference to an inline object and a reference to an identity object,
other than the fact that references under a `Q` descriptor cannot be null.

The choice of `L` vs `Q` descriptors is tightly tied to whether the named class
resolves to an identity or inline class; it is a linkage error for an `L`
descriptor to resolve to an inline class, or for a `Q` descriptor to resolve to
an identity class.   (The need for a separate basic type designator derives in
part from the fact that inline classes must be preloaded more aggressively than
identity classes, such as during field layout.)

#### Supertypes

Inline classes may implement interfaces, and may extend certain restricted
abstract classes as well as the special class `Object`.  (Restrictions on
abstract classes include having no fields and an empty constructor.)  This means
that a variable of an interface type, suitable abstract class types, or `Object`
may be a reference to _either_ an identity object or an inline object.  The JVM
treats such extension as ordinary subtyping, and so references to inline objects
may be widened to references to `Object` or suitable interface or abstract class
type without casting.  Inline classes do not have constructors; instead, they
have static factory methods (whose name is `<new>`.)

#### Restrictions

Inline classes come with some restrictions compared to their identity
counterparts.  Instance fields of an inline class must be marked `ACC_FINAL`,
and they must implement the interface `InlineObject` (and may not implement
`IdentityObject`, directly or indirectly.)  If they extend a class other than
`Object`, that class must be abstract, have no fields, and have an empty
(possibly marked `ACC_ABSTRACT`) no-arg constructor.  (The static factory of the
inline class does not call the superclass constructor, because by definition it
must be empty anyway.)

The fields of an inline class `V` may not, either directly or indirectly, have a
field of type `V`.

## Bytecodes

The incorporation of inline classes affects a number of bytecodes.  Some are
new, and others gain additional behavior or restrictions when applied to inline
types.

The existing `a*` bytecodes are extended to uniformly support references to
both identity and inline classes.  

There are two new bytecodes used for constructing instances of inline classes.

 - `defaultvalue` is the analogue of `new` for inline class instances; it leaves
   a reference to the default (all zero) instance of an inline class on the
   stack.  (Uninitialized fields and array elements are also initialized to the
   default value for that type, as if by `defaultvalue`.)

 - `withfield` is the analogue of `putfield` for inline class instances; it
   takes a reference to an inline object and a new field value on the stack, and
   leaves  a reference on the stack to an instance of the inline class whose
   fields are all the same as the original instance, except for the specified
   field.

The `withfield` bytecode is restricted; it can only be executed by the class
that declares the inline class being modified.  This encapsulates creation of
novel values, so the class can ensure that any value escaping the implementation
is seen to adhere to its invariants.  The `new` and `putfield` bytecodes may
not be used with instances of inline classes, and the `withfield` bytecode may
not be used with identity classes.

The `aastore` instruction performs a null check on the element value before
storing a value into an array of inline class instances.

The `acmp*` instructions perform a more sophisticated comparison between their
operands.  Two object references are consider equal if they are both null, or
both references to the same identity object, or both references to inline
objects of the same type and all of their fields are "equivalent".  Equivalence
means recursively applying a suitable equality comparison to the fields, which
uses ordinary equality for primitives (except for `float` and `double` fields,
which are compared with the semantics of `Float::equals` and `Double::equals`)
and `acmp` for references.  The `if_acmpnull` instruction always yields false
when applied to references to inline objects.

## From Q-World to L-World

In previous iterations of the Valhalla prototype, `Q` descriptors had a separate
carrier, which was not interoperable with the `L` carrier (just as primitives
are not interoperable with reference types without explicit conversion).  And,
like primitives, Q types had no supertypes; access to interface and `Object`
members had to go through conversion to an `L`-typed companion class (which
functioned as boxes, but without the accidental identity of today's boxes).

Similarly, inline classes previously had their own data-movement bytecodes
(`v*`),  whereas in L-World, object references to both identity and inline
objects are uniformly moved through `a*` bytecodes.  Finally, in Q-world,
arrays of inline objects were not covariant with arrays of `Object`.

These distinctions treated inline classes much as "enhanced primitives"; they
could code like a class, but they truly behaved like an `int` -- in the good
ways and the bad.  This caused significant difficulty in migrating existing
identity classes to inline classes, as well as in  generify uniformly over
identity and inline objects, because there were seams between inline and
identity classes at every level (descriptors, bytecodes, subtyping
relationships).  The L World design was significantly informed by the challenges
that this approach created.
