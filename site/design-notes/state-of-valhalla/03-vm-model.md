# State of Valhalla

#### Section 3: JVM Model
#### John Rose and Brian Goetz, Apr 2021

::: sidebar
Contents:

1. [The Road to Valhalla](01-background.html)
2. [Language Model](02-object-model.html)
3. [JVM Model](03-vm-model.html)
:::

This document describes the _Java Virtual Machine_ view of primitive classes.  Note
that this is not necessarily the same as the Java Language view; readers are
advised to exercise care in drawing conclusions about the Java Language from
this document.

#### Primitive classes

Prior to Valhalla, all objects -- instances of classes and arrays -- had a
unique _object identity_.  Valhalla allows classes to choose, by marking a class
with the `ACC_PRIMITIVE` flag, whether their instances have identity ("identity
classes") or not ("primitive classes").  

In the abstract virtual machine (as described by classfiles), both primitive and
identity objects are referred to exclusively through _object references_.
However, because the JVM knows that primitive objects do not have identity, it can
routinely optimize layout (flattening), instantiation, member access, and
calling conventions (scalarization) for primitive objects.

#### Carriers and basic types

JVM type descriptors (JVMS 4.3.2) use a leading letter to denote their _basic
type_.  There are currently eight basic types corresponding to the eight
primitive types (`I`, `J`, etc), and one basic type corresponding to object
references (`L`).  (The letter `V` appears in type descriptors but is not
considered a type.)  The character `[` in type descriptors introduces an array
type that is hardwired to an internally defined array class, but for our
purposes it is a variation of the `L` syntax, introducing a special type of
object reference.

These nine basic types correspond to five _carrier types_ (int, long, float,
double, and object reference), which correspond to different representations of
a value in a stack slot or local variable slot.  (The difference between these
two sets comes from erasing `byte`, `short`, `char`, and `boolean` to `int` and
using the `I` carrier.)  The primitive carriers store a value directly in the
stack or local variable slot (long and double values use two adjacent slots),
and the object reference (`L`) carrier stores an _object reference_ in the
corresponding slot.

#### Q descriptors

To describe primitive types, we add a new basic type, `Q`, which denotes a
reference to a so-called _exotic class_.  (Presently, the only kind of exotic
class is a primitive class.)  `Q` descriptors have the same syntactic structure
as `L` descriptors (e.g., `Qjava/lang/int;`).

In addition to reusing the syntactic form of `L` descriptors, `Q` descriptors
also reuse the `L` carrier -- in the JVM, there is no structural difference
between a reference to an primitive object and a reference to an identity object.

Both `Q`-carried and `L`-carried values are references are operated on by the
same set of `a*` bytecodes.  The JVM verifier ensures, however, that `Q`-carried
and `L`-carried values, even of the same class, are kept distinct.  This allows
`Q`-carried references to participate reliably in flattening and scalarization,
optimizations impossible (even in principle) for `L`-carried values.

`Q`-carried values cannot be null if the underlying class excludes null -- which
is the case for all primitive classes in the present design.  This allows a `Q`
type to "work like an `int`", without troubling the JVM to represent a `null`
value in a 32-bit integer.  Also, the JVM is permitted (indeed, required) to
load classes named in `Q` descriptors much earlier than the case of `L`
descriptors; such early loading allows the definition of a `Q`-defined field or
argument to be scalarized as soon as the variable for the field or argument is
created, instead of later as a back-filled optimization.

The choice of `L` vs `Q` descriptors is tightly tied to the corresponding
_contract_ for descriptors of either kind.  The contract for an `L` descriptor
is as follows:

 - The `null` reference is the initial value of an `L`-carried field or array
   element.
 - Thus, `null` is always permitted in a `L`-carried value.
 - Nearly all uses of `L` descriptors are non-resolving.  (Thus, the JVM _may
   not_ load the class of an `L` descriptor except in response to specified
   resolution requests.  This is why `null` is required as an initial value.)
 - Data structures can be circular if linked via `L` descriptors.  (This is
   another corollary of non-resolution of `L` descriptors.)
 - Applications do not suffer from bootstrap circularities because of `L`
   descriptors.  (Yet another corollary of non-resolution of `L` descriptors.)
 - The normal execution of a `checkcast` is designed to treat its type operand
   as if it were an `L` descriptor if the reference operand is `null`.  (There
   is no resolving in this case.)
 - `L` fields and array elements must not be flattened; they are pointers.
 - `L` fields and array elements have strong atomicity, even during races.
 - `L` descriptor names participate in class loader constraints.

The contract for `Q` descriptors is, in essence, a matching "anti-contract" to
the `L` descriptor contract:

 - The `null` reference is excluded from (most or all) `Q`-carried values.
 - Nearly all uses of `Q` descriptors are resolved eagerly, before the moment
   when a field of that type is first created, or a method using that type (as
   argument or return value) is first prepared.  (Thus, the JVM _must_ load the
   class of an `Q` descriptor in many cases.  This is how `null` can be excluded
   as a value, and how flat fields and scalarized method calling sequences can
   be arranged.)
 - Data structures _may not_ be circular if linked via `Q` descriptors.
   (Instead, the JVM may nest `Q`-carried values in a single block of memory.)
 - Applications _may suffer from bootstrap circularities_ because of `Q`
   descriptors.  (Over-use of `Q` descriptors carries risks of an infinite
   regress in class loading.)
 - The execution of a `checkcast`, when applied to a `Q` descriptor (instead of
   a class name) performs eager resolution.
 - `Q` fields and array elements _may be_ (and are routinely) flattened.
 - Multi-word `Q` fields and array elements may exhibit "struct tearing" during
   races, unless other measures are taken to preven this.
 - `Q` descriptor names are not required to store class loader constraints.
   (Because of eager loading, the constraints can be checked using resolved
   classes when methods are prepared.  This is a variation of class loader
   constraint checking which does not store constraint records in the system
   dictionary.)

With all this said, it would seem that any class whatever might somehow benefit
from having its values stored in `Q` descriptors, along with `L` descriptors.
That is not the case, and the JVM forbids "frivolous" `Q` descriptors to
non-exotic classes, checking that whenever a `Q`-carried class name is resolved,
it in fact refers to an exotic class.  (In the present design, which features
only primitive classes, a `Q` descriptor may only be applied to a primitive
class.  There may be future uses of `Q` descriptors which make use of the `Q`
contract to support other kinds of exotic types, such as specialized generics or
enhanced primitives.)

Because of their general robustness and usefulness, `L` descriptors may be
applied to any kind of class, and indeed perform many useful jobs with primitive
classes, in cases where nullability or circularity or lazy loading is a
requirement.  But because `L` descriptors are lazy, it is impossible to ensure
that an `L` descriptor of a primitive class can describe that class's properties
with enough precision (and timeliness) to allow all uses of an `L` descriptor to
reliably flatten and scalarize.  This is why Valhalla needs an "anti-`L`".  It
it not possible simply to adjust the behavior of `L` descriptors to unlock those
optimizations, because the required eager loading violates some of the basic
properties of `L` descriptors.

(The roles of `Q` and `L` descriptors can be recalled easily if one
notes that `Q` descriptors load _quickly_, while `L` descriptors defer
loading until _later_.  The true origin of `L` is unclear; perhaps it
follows the second letter of "class" since `C` for `char` was taken.
The letter `Q` was chosen to refer to a "quantity" during the design
of value types.  The `Q` also stands for "quid", Latin for "what?"  so
it has also been noted that the eager resolution of a `Q` descriptor
seeks the "quiddity" of the type by quickly loading its class file.)

#### Supertypes

Primitive classes may implement interfaces, and may extend certain restricted
abstract classes as well as the special class `Object`.  This means that a
variable of an interface type, suitable abstract class types, or `Object` may be
a reference to _either_ an identity object or an Primitive object.  The JVM treats
such extension as ordinary subtyping, and so references to primitive objects may be
widened to references to `Object` or suitable interface or abstract class type
without casting.  Primitive classes do not have constructor methods; instead,
source level constructors are translated to static factory methods with the same
argument list as the declared constructor, whose name is `<new>`.  Unlike an
`<init>` method, which must return `void`, a static factory method must return
the `Q` descriptor type of the class being constructed.  (A static factory
method named `<new>` is also allowed to return `Object` or perhaps one of the
supertypes of that class, as an `L` descriptor; this is necessary for exotic
hidden classes.)

There is no need for any supertype of an exotic (i.e., primitive) class to
itself be exotic in any way.  Thus, while `Q` (and sometimes `L`) descriptors
will play a role with primitive values, the descriptors associated with their
supertypes will never be `Q` descriptors.  In particular, there is no legal use
for `Qjava/lang/Object;`.

An abstract class is a _primitive superclass candidate_ if it has no fields,  no
static initializer (`<clinit>`) method, its no-arg constructor is empty (marked
`ACC_ABSTRACT`), and its superclass is either `Object` or a primitive superclass
candidate.  (The static factory of the primitive class does not call the superclass
constructor, because by definition it must be empty anyway.)

The properties of being a primitive superclass candidate are "contagious" up the
superclass chain.  That is, if some abstract class has an empty no-arg
constructor, its immediate superclass must _also_ have an empty no-arg
constructor.  It would seem to follow that even `Object` must have a empty
(`ACC_ABSTRACT`) constructor, but `Object` is special, because it alone (of all
classes) is allowed to have a truly empty concrete constructor method.  Thus,
`Object` is exempt from the "contagion" of the requirement of empty
constructors.

#### Restrictions

Primitive classes come with some restrictions compared to their identity
counterparts.  Instance fields of an primitive class must be marked `ACC_FINAL`.
Primitive classes must implement the interface `PrimitiveObject` (and may not
implement `IdentityObject`, directly or indirectly.)  If they extend a class
other than `Object`, that class must be a primitive superclass candidate.  The
non-`static` fields of an primitive class `V` may not, either directly or
indirectly, have a field of type `V` using a `Q` descriptor.  (Such an object
would have no definite size; each one would contain an infinite regression of
sub-objects of the same type.)

An `L` descriptor can be applied to a primitive class in any case; any infinite
regress is stopped immediately by a `null` reference.

A `static` field of type `V` using a `Q` descriptor in `V` is not problematic,
although it requires careful preparation, and (in general) a hidden indirection
to manage lazy initialization.

## Bytecodes and constants

The incorporation of primitive classes affects a number of bytecodes.  Some are
new, and others gain additional behavior or restrictions when applied to primitive
types.  The existing `a*` bytecodes are extended to uniformly support references
to both identity and primitive classes.

There are two new instruction used for constructing instances of primitive classes.

 - `defaultvalue` is the analogue of `new` for primitive class instances; it leaves
   a reference to the default (all zero) instance of an primitive class on the
   stack.  (Uninitialized fields and array elements are also initialized to the
   default value for that type, as if by `defaultvalue`.)

 - `withfield` is the analogue of `putfield` for primitive class instances; it
   takes a reference to an primitive object and a new field value on the stack, and
   leaves  a reference on the stack to an instance of the primitive class whose
   fields are all the same as the original instance, except for the specified
   field.

The `withfield` instruction is restricted; it can only be executed by the class
that declares the primitive class being modified.  This encapsulates creation of
novel values, so the class can ensure that any value escaping the implementation
is seen to adhere to its invariants.  The `new` and `putfield` bytecodes may
not be used with instances of primitive classes, and the `withfield` bytecode may
not be used with identity classes.

The `aastore` instruction performs a null check on the element value before
storing a value into an array of primitive class instances.

The `acmp*` instructions perform a more sophisticated comparison between their
operands.  Two object references are consider equal if they are both `null`,
both references to the same identity object, or both references to primitive
objects of the same type and all of their fields are "equivalent".  Equivalence
means recursively applying a suitable equality comparison to the fields, which
uses ordinary equality for primitives (except for `float` and `double` fields,
which are compared with the semantics of `Float::equals` and `Double::equals`)
and `acmp` for references.  The `if_acmpnull` instruction always yields false
when applied to references to primitive objects.

The `checkcast` instruction is extended to apply to a `Q` descriptor, with the
existing `checkcast` interpreted as following the rules for `L` descriptors.  In
particular, the normal behavior of `checkcast` is to pass a stacked `null` value
immediately _without loading the class_.  (This is an unusual case of a plain
`CONSTANT_Class` item _not_ being resolved; it treats that constant pool item
almost as if it were an `L` descriptor.)  But `CONSTANT_Class` items are now
allowed to contain `Q` descriptors as well as the two existing choices: class
names and array descriptors.  (It may be seen that these three kinds of names
are disjoint as strings.)  A `checkcast` applied to a `Q` descriptor _always_
resolves its `Q` descriptor, and any `null` present will be processed against
the requirements found in the class file (which at present will always reject a
`null`).

The `anewarray` instruction is similarly extended to apply to a `Q` descriptor;
this determines whether the resulting array is flattened (`Q`) or not (`L`).

The `ldc` instruction, applied to a `CONSTANT_Class` item which contains a `Q`
descriptor, will return the secondary (`val`) mirror for the class.  The normal
syntax (of a plain class name in a `CONSTANT_Class` item) will return the
primary (`ref`) mirror; again, it is as if the normal syntax works like an `L`
descriptor.

## Verifier rules

The verifier keeps track of the "flavor" of each reference value (`L` vs `Q`) as
derived from the descriptor at its source.  Because the contract of `L`
descriptors is in every way more lenient (for value sets) than that of `Q`
descriptors, the verifier behaves as if `QFoo;` is a subtype of the plain
reference `Foo` (also known as `LFoo;`), for every `Foo`.

There is no need for a verifier rule that demands `checkcast` when promoting
from `QFoo;` to `LFoo;` or any of its supers, all of them also plain `L`
descriptor types.  By its nature, an `L` descriptor can never constrain anything
but the class named by the descriptor.  There may be technical reasons to
voluntarily issue a `checkcast` against a super of `QFoo;` other than `LFoo;` or
`Object`, since that would avoid having the verifier load `Foo.class` in order
to ensure that the conversion is a widening conversion.  Note that the verifier
is not required to load a class when it is being implicitly converted to
`Object`, to an interface, or to a class of the same name but different
"flavor", `Q` vs. `L`.

In the verifier, the `null` type is a subtype of every plain (`L` descriptor)
type, but is not a subtype of any `Q` type.  (Note that as a result of these
rules, every `Q`-carried value is sourced from either a previous `Q`-carried
value, or else is the result of a `checkcast`.  These rules are sufficient to
ensure that stray `null` value can be excluded eagerly in the interpreter,
before they would pass into fields that are flattened or method arguments that
are scalarized, and even before an optimizer begins to scalarized calling
sequences.)

In the verifier, a type `QFoo;` has no subtypes at all, and no immediate
supertypes other than plain `Foo` (`LFoo;`).  Of course, `QFoo;` shares all of
`Foo`'s supertypes.  Thus, for example, a `QFoo;` value can be immediately
stored in an `Object` variable, or passed through an `Object` argument, or
returned as an `Object` return value.

If it ever becomes possible to store `null` in a primitive class, as has been
proposed from time to time, then the verifier will not know about these new
rules for `null`.  An `aconst_null` would then always be followed by a
`checkcast QFoo;` in order to store a legitimate `null` into a flattened field,
if that becomes possible.  Because the verifier allows any reference as an input
to a `checkcast`, there is no reason to further reason about the relations
between the `null` reference type and any `Q` descriptor type.  The design seems
future proof with respect to the status of `null` relative to exotic classes.

## Arrays

An array type is a reference type (using the `L` contract) generated
from a descriptor which begins with `[`.  The addition of `Q`
descriptors simply implies that an array element type can be either
`L` or `Q`.  The JVM does not attempt to flatten an array with an `L`
component descriptor; such arrays are sometimes useful.  (They may
have faster sort times for certain sizes of primitives, and their
updates are guaranteed to be atomic.)  However, if an array has a `Q`
component descriptor, the JVM may be expected to flatten the elements
of that array.  Such a flattened array will (probably) be smaller or
larger than a non-flat array, just as the flattened primitive elements
are smaller or larger than an object reference.  Memory referencing
patterns (say, during a loop traversing the array in order) are likely
to be more predictable in the flattened case.

## Mirror, mirror

When a class is reflected, its field and method types directly represent the
corresponding descriptors by using a distinct mirror (of type `Class`) for each
distinct descriptor.  (There are even mirrors for `int` and `void`!)  And since
an exotic (i.e., primitive) class `Foo` can appear with either kind of
descriptor (`L` or `Q`), it follows there must be two mirrors associated with
any primitive class `Foo`.  At the source level these correspond to `Foo.ref`
and `Foo.val` (or more likely the convenience alias `Foo`).  These are
translated as `LFoo;` and `QFoo;`, respectively, and mirrored as distinct
mirrors `Foo.ref.class` and `Foo.val.class`.  The JVM generates both mirrors
automatically when loading an exotic (i.e., primitive) class.

In all paths through the reflective APIs which classically (before
Vahalla) yield class mirrors, the JVM will continue to return `ref`
projection mirrors, even if the class in question is exotic.  This is
more compatible than returning the "more accurate" value.  The
situation is similar, in some ways, to that of `Integer.class` and
`int.class`, if one can imagine that `Integer` works like `int.ref`,
and that `I` is really (somehow) an exotic (`Q`-like) descriptor for
`int.val`.  The `L` descriptor returns `int.ref.class`, while the
exotic descriptor returns `int.val.class`.

In particular, `Class::forName` and `Object::getClass` will always return a
`ref` projection.  If you want the mirror that "works like `int.class`", you
have to look somewhere special, as with `int` today, or  ask the primary mirror
for its corresponding secondary mirror.  There will be an API point on `Class`
for obtaining the `val` mirror from the `ref` mirror, and vice versa.

## From Q-World to L-World

In previous iterations of the Valhalla prototype, `Q` descriptors had a separate
carrier, which was not interoperable with the `L` carrier (just as primitives
are not interoperable with reference types without explicit conversion).  And,
like primitives, `Q` types had no supertypes; access to interface and `Object`
members had to go through conversion to an `L`-typed companion class (which
functioned as boxes, but without the accidental identity of today's boxes).
Similarly, primitive classes previously had their own data-movement bytecodes
(`v*`),  whereas in L-World, object references to both identity and primitive
objects are uniformly moved through `a*` bytecodes.  Finally, in Q-world, arrays
of primitive objects were not covariant with arrays of `Object`.

These distinctions treated primitive classes much as "enhanced primitives"; they
could code like a class, but they truly behaved like an `int` -- in the good
ways and the bad.  This caused significant difficulty in migrating existing
identity classes to primitive classes, as well as in  generify uniformly over
identity and primitive objects, because there were seams between primitive and
identity classes at every level (descriptors, bytecodes, subtyping
relationships).  The L-World design was significantly informed by the challenges
that this approach created.

A previous version of L-World worked hard to keep `Q` descriptors separate from
`L` descriptors.  The verifier did not support converting `QFoo;` to `Foo` (aka.
`LFoo;`), and it was viewed as an error to use field and method types which
mentioned `LFoo;` if in fact `Foo` was a primitive class.  (Because of `L`'s
endlessly deferred "later loading", it was possible that the JVM would never
detect such an error, in some cases.)  This might be summarized as "L-XOR-Q"
world, where no class uses both descriptors.  It was, for a time, a simplifying
assumption for JVM prototyping.

But sometimes you really need the `L` contract, even for primitive values.  (You
may require nullability, or data structure recursion, or non-flatness, or some
other benefit related to `L`.)  In "L-XOR-Q" world, the `L` version of a
primitive type is not available, so a split classfile model was designed, the
so-called "Eclair" pattern, where the `val` projection of a primitive class was
defined in `Foo.class` while the `ref` projection was in `Foo$ref.class`.  Then
the "L-XOR-Q" condition is satisfied by parcelling out the type uses as `QFoo;`
and `LFoo$ref;`.  But the split classfile design has obvious problems, such as
redundancy of representation (which can get out of sync. and must be verified),
and also odd reflection artifacts.

The latest iteration of L-World discards the split classfiles, and simply allows
both kinds of descriptors to apply (with either descriptor contract, as
appropriate) to a primitive class, and similarly for both kinds of mirrors.  
