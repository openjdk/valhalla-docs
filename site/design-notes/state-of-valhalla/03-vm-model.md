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
between a reference to a primitive object and a reference to an identity object.

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
   races, unless other measures are taken to prevent this.
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
variable of an interface type, a suitable abstract class types, or `Object` may be
a reference to _either_ an identity object or a primitive object.  The JVM treats
such extension as ordinary subtyping, and so references to primitive objects may be
widened to references to `Object` or suitable interface or abstract class type
without casting.  Primitive classes do not have constructor methods; instead,
source level constructors are translated to static factory methods with the same
argument list as the declared constructor, with method name `<new>`.  Unlike an
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

An abstract class is a _primitive superclass candidate_ if it has no fields, its
no-arg constructor is empty (marked `ACC_ABSTRACT`), it has no other `<init>`
methods, and its superclass is either `Object` or a primitive superclass
candidate.  (The static factory of the primitive class does not call the
superclass constructor.  This can be safe only if the constructor is empty
anyway.)

The properties of being a primitive superclass candidate are "contagious" up the
superclass chain.  That is, if some abstract class has an empty no-arg
constructor, its immediate superclass must _also_ have an empty no-arg
constructor.  It would seem to follow that even `Object` must have a empty
(`ACC_ABSTRACT`) constructor, but `Object` is special, because it alone (of all
classes) is allowed to have a truly empty concrete constructor method.  Thus,
`Object` is exempt from the "contagion" of the requirement of empty
constructors. Note, even if `Object` is contrived to have a non-empty constructor
it will not be called by primitive classes.

In its role as a sort of "honorary interface", `Object` commits to neither kind
of class; it implements neither of the marker interfaces `IdentityObject` nor
`PrimitiveObject`.  Yet the existing behavior of the expression `new Object()`
creates an identity object.  It seems that `Object` is wearing too many hats!
The JVM will have to give special treatment to the bytecode instruction `new
java/lang/Object`, which after link resolution will create an identity object of
a secret substitute class which has no fields (like `Object`) and implements
`IdentityObject`.  The verifier does not have to know about this, and the
subsequent `invokespecial` of `Object::<init>` will be the only initialization
the object ever experiences.  This secret class will be designed to allow this
treatment. [Does this mean we will hide the class from JVMTI pre-load events
like the ClassFileLoadHook? The class can be made non-modifiable to prevent
post-load `<init>` changes]

If an identity class declares a primitive superclass candidate as its super, it
must (as usual) call the superclass `<init>` method, but the JVM links this call
straight up to the no-arg constructor of `Object`, skipping the intervening
abstract (empty) `<init>` methods.

If a primitive superclass candidate were allowed to have either fields or
non-empty `<init>` methods, they would presumably be available in the usual way
to identity subclasses of the superclass.  But those extra `<init>` methods
would be unusable by primitive subclasses of the same superclass, and even if
the fields were somehow "pulled down" into the primitive class, they would not
properly initialized by it.  If these obstacles could be worked around somehow,
it would be possible to create primitive subclasses of `java.lang.Enum` and
other important classes.  We may revisit and relax the restrictions on primitive
superclasses when we have more experience.

#### Restrictions

Primitive classes come with some restrictions compared to their identity
counterparts.  Instance fields of a primitive class must be marked `ACC_FINAL`,
and must not be marked `ACC_VOLATILE`.

Primitive classes are implicitly adjusted by the JVM, as necessary, to implement
the interface `PrimitiveObject`.  They or their supers may explicitly declare
`PrimitiveObject` as a super, in which case the JVM does not insert the extra
super.  They may not directly or
indirectly implement the interface `IdentityObject`.

Also, abstract classes that fail to be primitive superclass candidates (perhaps
because they contain fields) are implicitly adjusted by the JVM, as necessary,
to implement the interface `IdentityObject`.  They may not directly or
indirectly implement the interface `PrimitiveObject`.

In this way, every concrete object implements exactly one of `IdentityObject` or
`PrimitiveObject`, and every abstract class implements at most one of the two.
Most interfaces, and `Object`, will implement neither.

JVM-inserted supers are *not* visible to reflection via `Class::getInterfaces`,
but they are respected, on a per-instance basis, by `checkcast`, `instanceof`,
`aastore`, `Class::isInstance`, and `Class::cast`.  They are also respected by
`Class::isAssignableFrom`.  Thus, to detect if the JVM has inserted the super
`IdentityObject` into some class `C`, first call `Class::isAssignableFrom` to
test each super of `C` for `IdentityObject`, and if it is not present, then test
`Class::isAssignableFrom` on `C` itself; if `C` mysteriously implements the
interface despite none of its supers doing so, the JVM has secretly made its
mark on `C`.  Keeping secrets from reflection is a risky business, but in this
case it seems safer to do so, lest a large fraction of existing Java classes
suddenly grow new items in their `Class::getInterfaces` arrays.
[ Hiding the injected interface from `Class::getInterfaces` seems to be one of
those "solves today's problems while creating tomorrow's" and wouldn't pass
muster if we were designing the feature without the weight of existing test
suites. Are there other concerns than the existing test suites?  Given the
other more significant changes in the ecosystem, I'm unclear why this one
is worth avoiding.] 

If a primitive class extends a class
other than `Object`, that class must be a primitive superclass candidate.  The
instance fields of a primitive class `Foo` may not, either directly or
indirectly, have a field of type `Foo` using a `Q` descriptor.  (Such an object
would have no definite size; each one would contain an infinite regression of
sub-objects of the same type.)

An `L` descriptor can be applied to a primitive class in any case; any infinite
regress is stopped immediately by a `null` reference.

A `static` field of type `Foo` using a `Q` descriptor in `Foo` is not
problematic, although it requires careful preparation, and (in general) a hidden
indirection to manage lazy initialization.  Such `static` fields of type `QFoo;`
must be initialized to the default value of `Foo`, but the JVM may need to
prepare such static fields before the default value is materialized, which may
possibly require lazy or indirect access to the field.  If so, this internal
logic can be hidden within the `getstatic` bytecode.

## Bytecodes and constants

The incorporation of primitive classes affects a number of bytecodes.  Some are
new, and others gain additional behavior or restrictions when applied to primitive
types.  The existing `a*` bytecodes are extended to uniformly support references
to both identity and primitive classes.

There are two new bytecode instructions used for constructing instances of
primitive classes.

 - `defaultvalue` is the analogue of `new` for primitive class instances; it leaves
   a reference to the default (all zero) instance of a primitive class on the
   stack.  (Fields and array elements of a `Q` type are also initialized to the
   default value for that type, as if by `defaultvalue`.)  The sole operand of
   this bytecode is a reference to a `CONSTANT_Class` item giving the name
   of the primitive class (not its `Q` descriptor).

 - `withfield` is the analogue of `putfield` for primitive class instances; it
   takes a reference to a primitive object and a new field value on the stack, and
   leaves  a reference on the stack to an instance of the primitive class whose
   fields are all the same as the original instance, except for the specified
   field.

The `withfield` instruction is restricted; it can only be executed within the
nest of the class
that declares the primitive class being modified.  This encapsulates creation of
novel values, so the class can ensure that any value escaping the implementation
is seen to adhere to its invariants.  The `new` and `putfield` bytecodes may
not be used with instances of primitive classes, and the `withfield` bytecode may
not be used with identity classes.

(In a future version of a VM, a class may be able to grant access to `withfield`
instructions to other clients, on an opt-in basis, if this seems useful.  In
fact, in a future JVM a class may also be able to revoke access to
`defaultvalue` instructions in other clients, on an opt-out basis, again if this
seems useful.)

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
Note that the `anewarray` instruction returns a plain (`L`) reference to an
array regardless of its component type.

The `aaload` instruction, as always, takes an array of a particular reference
element type, and returns an element of that type.  If the array element type is
a `Q` type, the value pushed on the stack by `aaload` is tracked by the verifier
as a `Q` type.  The `aastore` instruction, as always, relies fully on the
dynamic array store check; the verifier is indifferent to the precise component
type of the array input to `aastore`.

The `ldc` instruction, applied to a `CONSTANT_Class` item which contains a `Q`
descriptor, will return the secondary (`val`) mirror for the class.  The normal
syntax (of a plain class name in a `CONSTANT_Class` item) will return the
primary (`ref`) mirror; again, it is as if the normal syntax works like an `L`
descriptor.

Most uses of `CONSTANT_Class` continue to require plain class names, not array
descriptors or `Q` descriptors.  A `CONSTANT_Class` item that specifies `Q`
descriptor is valid only for the following uses:

 - as a loadable constant: an operand of `ldc` or a bootstrap method
 - as an operand of `checkcast` or `anewarray`
 - in a stack map table for the verifier (to distinguish `L` and `Q` types)

In particular, `CONSTANT_Fieldref`, `CONSTANT_Methodref`, and
`CONSTANT_InterfaceMethodref` items and `instanceof` instructions must not refer
to `CONSTANT_Class` items with `Q` descriptors.

## Verifier rules

The verifier keeps track of the "flavor" of each reference value (`L` vs `Q`) as
derived from the descriptor at its source.  Because the contract of `L`
descriptors is in every way more lenient (for value sets) than that of `Q`
descriptors, the verifier behaves as if `QFoo;` is a subtype of the plain
reference `Foo` (also known as `LFoo;`), for every `Foo`.

The verifier tracks `Q`-carried values, deriving them from from the following
sources:

 - any `Q` descriptors in the argument types of the current method
 - the `invoke` family (when the return value is a `Q` type)
 - `getfield`, `getstatic` (when the field is a `Q` type)
 - `defaultvalue`, `withfield` (always produces a `Q` type)
 - `aaload` (when the array component type is a `Q` type, not an `L` type)
 - `checkcast` (when the `CONSTANT_Class` operand is a `Q` descriptor)
 - `ldc` (when the constant is a `Q` type)

The verifier enforces the `Q` marking of values when it sends them to
the following consumers:

 - `areturn` (when the return value is a `Q` type)
 - the `invoke` family (any `Q` descriptors for outgoing arguments)
 - `putfield`, `putstatic`, `withfield` (when the field is a `Q` type)
 - `withfield` (the stacked object to update must be a `Q` type)

In some cases the JVM uses dynamic checks to ensure type safety, and the
verifier does not enforce marking of `Q` types for these consumers, but
rather accepts `Q` and `L` marked types indiscriminately:

 - the receiver of `getfield`, `putfield`, and the `invoke` family
   (these already include a mandatory `null` check)
 - `aastore` (this has always been purely dynamic)
 - `checkcast` (`Q` promotes to `L`, which is then cast)

There is no need for a verifier rule that demands a `checkcast` when promoting
from `QFoo;` to `LFoo;` or any of its supers, all of which are also plain `L`
descriptor types.  By its nature, an `L` descriptor can never constrain anything
but the class named by the descriptor.  There may be technical reasons to
voluntarily issue a `checkcast` against a super of `QFoo;` other than `Foo` or
`Object`, since that would avoid having the verifier load `Foo.class` in order
to ensure that the conversion is a widening conversion.  Note that the verifier
is not required to load a class when it is being implicitly converted to
`Object`, to an interface, or to a class of the same name but different
"flavor", such as `QFoo;` converting to plain `Foo` (`LFoo;`).

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

A flattened array (whose component is a `Q` descriptor), loads values of that
`Q`-carried type.  A referance value of any kind (whether `Q` or `L` or null or
array) may be presented to the `aastore` instruction to store into the flattened
array, but the store check will apply the same logic as `checkcast`, to ensure
that the incoming value can actually be stored as a flattened array element.
This behavior is essentially unchanged from before Valhalla.

A `Q` type may be viewed as a subtype of its corresponding `L` type, but only in
a limited sense, specifically in the verifier's specialized type system.  Array
covariance also "pretends" that the `Q` type is a subtype of the `L` type, so
that if two arrays have the same element class, but one is flattened, the
flattened one can be stored in a reference to the non-flattened type, but not
vice versa.  The `checkcast`, `instanceof`, and `aastore` logic, which applies
covariance rules to arrays, enforces this relation between flattened and
non-flattened arrays of the same class, and also (transitively) allows any
flattened array to be stored in a variable of type `Object[]`.

(At present, `int[]` is still not a subtype of `Object[]`, even though
`Point.val[]` is a subtype of `Object[]`, since it works with `aastore`.  At
some point, if the built-in primitives are brought into alignment with primitive
classes, the subtyping links to `int[]` and the other primitive arrays may be
set up.)

## Heap effects

A primitive object may be stored either nested inside another object or array,
or directly on the heap.  In the latter case, we say that the primitive object
has been _buffered_ on the heap.  This is sometimes necessary in order to treat
the object as a regular reference.  (For example, adding it to a `List<Object>`
requires buffering.)  The JVM handles buffering transparently, and tries avoid
buffering when possible, lifting them into registers or storing them in flat
variables.

A sudden buffering of many large primitive objects, transferring their contents
from the stack to the heap, could in principle cause an `OutOfMemoryError`.
This could happen, in particular, when a very large optimized stack frame
deoptimizes back to the interpreter, flushing temporary objects to the heap.
Such an error will probably appear "out of band" or causeless to the user.

Mutable state on the heap is subject to the rules of the Java Memory Model.
Relative to the JMM, a buffered primitive object must be treated comparably to
an identity object which has all final fields.  In both cases, some platforms
may require a memory barrier after all fields have been stored to heap, and
before the resulting pointer is published.

When an array element or mutable instance field is of a `Q` type, the effect of
loading or storing a `Q` value is tracked by the Java Memory Model as if the
primitive object were recursively broken down into the component subfields of it
and all of its embedded sub-objects, all the way down to plain pointers and
scalar (non-object) primitives like `int` and `long`.

In other words, a flattened value in the heap is "scalarized", decomposed into a
set of machine words.

#### Struct-tearing

It follows that a multi-threaded program can contain races on changes to these
individual pointers and scalars.  Thus, a thread can witness an "impossible"
combination of fields in a primitive object, given the right (or wrong!) race
condition.  We call this _struct-tearing_, a multi-word analogue of word-tearing
seen on some processors.  It is also analogous to the racing update of two
32-bit halves of a `long` or `double` value, which can also cause "impossible"
`long` or `double` values to appear in Java programs.  We also use the term
struct-tearing retroactively to refer to these effects on `long` and `double`.

Struct-tearing may not appear on some platforms.  On other platforms it may be
rare, although mischievous code might force it to happen on command.  In any
case, struct-tearing cannot be ruled out absolutely.  Java provides a workaround
to suppress struct-tearing of values stored in fields: Declare the field
`volatile`.  The JVM then excludes struct-tearing by any means it chooses:
Software transactions, hardware locking, hidden synchronization, extra
buffering, or some combination of the above.  This workaround does not apply to
values stored in array elements.

For 64-bit scalar values, struct tearing is inconvenient but has a limited
impact.  Since primitive classes will often carefully control their internal
invariants, struct-tearing can cause new kinds of failures, if it violates those
invariants.  The JVM is likely in the future to support some way to mark a
primitive class "non-tearable", instructing the JVM to ensure that
struct-tearing is impossible for all variables of that type, including array
elements.  The simplest way to do this is to always buffer and never flatten.
There may be other ways to get the effect, but there is likely always to be
a cost, so this "non-tearable" marking is not going to be the default.
The likely syntax in the class file is a marker interface implemented by
the class itself, `java.lang.NonTearable`.

#### Uninitialized variables

Heap variables also must be initialized to default values, even before they are
initialized by user code.  This is most visible with array elements, which are
always `null` or a primitive default value.  A potential problem with some
primitive classes is that there is no legitimate default value.  Often
programmers of classes want to perform special processing when every instance of
a class is created, and the `defaultvalue` bytecode subverts this, to some
extent.  (The programmer has to get used to these non-constructed default values
floating around.)  There are several proposals for what to do about this,
including "nothing", and "map some class default values to `null`".  The latter
option would make an uninitialized variable (of a primitive class that opts into
this treatment) look exactly the same as an uninitialized reference variable.

Another option is try to protect users from ever seeing uninitialized primitives
(of a class that opts for this processing) by issuing some sort of error when a
heap variable is loaded before it has been stored to.  In this design, the
`defaultvalue` instruction would become subject to an access check that
parallels the present access check to the `withfield` instruction.

It is possible that some sort of marker interface like `NonTearable` would
indicate classes that request such processing.

#### Weakness

Like one of today's `Integer` objects, a buffered primitive object may be used
as a key to a weak hash map.  (There are many such in the Java ecosystem.)  The
implementation of a weak hash map probably appeals to the `WeakReference` JDK
class or something similar.  In any case, applications make varying use of
`WeakReference` and related types, as well as JNI weak references.  It appears
that we cannot exclude buffered primitive objects from such uses, and so we
need to assign a meaning to a weak reference to a primitive object.

What does it mean to wait for a primitive object to become unreachable?  If the
primitive object is simply a bit-pattern of scalars, the unhappy answer is that
it is alway reachable in principle, since the bit-pattern could be read from
disk, loaded into a fresh copy of the primitive, and presented as a witness of
reachability.  After all, a reference is reachable if some other code can
produce a reference which `acmp` will compare as indistinguishable from the
original reference.  But `acmp` on a primitive pays attention only to object
content, not to object identity.

If a primitive object is composed of a combination of scalars and references
to identity objects, the problem is less hopeless, though still complicated.
A primitive object becomes unreachable (in the modified definition of the
previous paragraph) if and only if _at least one_ of its component identity
object references becomes unreachable.

It seems possible to implement this standard of reachability for primitive
objects using a combination of GC adjustments, JDK library changes to
`WeakReference`, and runtime code changes for JNI weak references. It also seems
terribly inconvenient to do this, and we are looking for use cases to help us
weigh the alternatives, such as refusing to build such `WeakReference`s (forcing
library authors to recode) or just holding on to primitives forever and ever.

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

[ Which mirror defines the fields for apis like `getFields()`?]

In particular, `Class::forName` and `Object::getClass` will always return a
`ref` projection.  If you want the mirror that "works like `int.class`", you
have to look somewhere special, as with `int` today, or ask the primary mirror
for its corresponding secondary mirror.  There will be an API point on `Class`
for obtaining the `val` mirror from the `ref` mirror, and vice versa.

### JVMTI & redefinition of mirrors
JVMTI, and the `java.lang.instrument` package provide powerful tools for
modifying classes.  The native `ClassFileLoadHook` allows modifying classes
before they are loaded and may be used to transform classes in surprising ways
such as adding methods or fields, and can be used in ways that javac would not,
violating invariants such as empty constructors.  Both tools can be used to
modify the bodies of methods after they have been loaded (assuming the classes
are not marked as `unmodifiable`).

* How do we treat the two mirrors generated from a single primitive classfile?
* Is the `ref` version marked as non-modifable (treating the `val` one as the `true`
class)?
* Does the CFLH get called once (for the classfile) or twice (for `val` & `ref`)?

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

[Not sure we're entirely clear of the odd reflection artifacts.  We may have just
moved them around a bit]

The latest iteration of L-World discards the split classfiles, and simply allows
both kinds of descriptors to apply (with either descriptor contract, as
appropriate) to a primitive class, and similarly for both kinds of mirrors.  
Members of the primitive class can be accessed via a receiver which uses either
the `Q` or `L` carrier.

## Static translation

The translation of Java classes which declare and use primitive classes requires
minor adjustments to the current translation strategy.

Primitive classes have their `ACC_PRIMITIVE` bit set, the `ACC_FINAL` bit on all
fields, and their constructors are translated to static `<new>` methods, but
otherwise are translated exactly as identity classes are.  Whether a primitive
class is reference-favoring or value-favoring does not affect the translation of
the class, only of the type uses of the unadorned class name.

[ I would still like to have the `PrimitiveObject` interface included by javac in
the primitive clases, especially if reflection hides injected interfaces.  Better
to be explicit to start so `getInterfaces` matches what people will expect from
something defined to have the `PrimitiveObject` interface]

Type uses of `P.ref` in method descriptors, field descriptors, and signature
attributes are translated as `LP;`; type uses of `P.val` are translated as
`QP;`.  (Uses of the unadorned `P` are aliases for one of these two
projections.)

Primitive widening conversion requires no explicit translation, since the JVM
treats `LP;` as a supertype of `QP;`.  Primitive narrowing conversion is
translated as a `checkcast` to the corresponding `Q` descriptor.

Instantiation of primitive classes (`new P(...)`) is translated as invocation of
the corresponding factory method with `invokestatic`.  In the body of the
factory method, rather than accepting an uninitialized `this` as a parameter,
`this` is initialized inside the body of the factory with `defaultvalue`,
assignments to fields of `this` are translated as modifying that value with
`withfield`, and the factory returns the completed `this`.

#### Differences between language and VM models

The relationship between the value and reference projections differs between the
language and VM models.  In the language, the value projection is not a subtype
of the reference projection; instead, the two are related by _primitive
narrowing and widening conversions_.  In the VM, the two are related by actual
subtyping.

It may seem strange to have deliberately picked a different model in the
language as in the VM, but as we'll see, this choice leaves language constraints
in the language (where they belong), and lets the VM focus primarily on being an
efficient translation target.

The primitive narrowing and widening conversions are semantically similar to the
more familiar unboxing and boxing conversions, but do not share many of the
negative performance costs associated with boxing (indirection, allocation,
accidental identity.)  When we convert a `P.val` to a `P.ref`, while this is
considered a conversion at the language level, no actual bytecode need be
emitted for this conversion, since in the VM `QP;` is a subtype of `LP;`.  In
the other direction, the static compiler emits a `checkcast` when narrowing a
`P.ref` to a `P.val`, but because the VM understands the relationship between
these types, this can be lowered to a null check (or completely optimized away.)
So what look like boxing and unboxing at the language level turn into no-ops and
near-no-ops in the bytecode.

Still, why did we not choose to simply expose the subtyping between `P.val` and
`P.ref` at the language level?  Because we wanted to preserve a path to unifying
legacy primitives and primitive classes at the language level.  Currently, in
the language, we have `int` and `Integer` related through ad-hoc boxing and
unboxing conversions.  We would like to migrate the legacy primitives to be
ordinary primitive classes, with the legacy box types being the reference
projection for those classes.  Having two different kinds of "boxes" (a legacy
heavy box and a new lighter box, where the former is related via boxing and the
latter via subtyping) for primitives would be a permanent wart, and having a
new, overlapping set of conversions (subtyping) might incompatibly change
existing overload selection decisions.  To remove this wart, we preserve the
semantics of the conversion between `int` and `Integer`, and extend it to all
primitive widening/narrowing conversions, so that the legacy boxing conversion
becomes the simpler primitive widening and narrowing.

Further, the boxing relationship plays a role in numerous language features,
such as type inference and overload selection.  By choosing primitive widening
and narrowing semantics to match boxing, these complex and pervasive language
features can continue to work as users expect them to when extended to primitive
classes, with little in the way of special treatment for primitives.

It may appear that we have pulled a rhetorical trick, just renaming "boxing" to
"primitive widening".  But the difference is: we now have a new translation
target, one which can routinely optimize layout, instantiation, and calling
conventions for primitive classes and their reference projections.  We've just
grafted familiar language semantics onto that efficient translation target,
preserving the illusion that nothing has changed (except performance.)

#### Legacy primitives

As discussed in [Language Model](02-object-model.html), a key goal of this story
is to migrate primitives so that they are "mostly just" primitive classes, while
retaining the performance characteristics of primitives.  

The existing wrapper classes (e.g., `Integer`) are redefined as primitive
classes.  We treat `int.ref` as an alias for `Integer`, and `Integer.val` as an
alias for `int`.  This means that the rules for type inference, overload
selection, type bound conformance, autoboxing, etc, can apply uniformly to the
legacy primitives as well as newly declared primitive classes.  

The JVM will, of course, continue to have direct support for primitives (e.g.,
the `I` carrier, and the `i*` instructions).  The language compiler will have
substantial latitude to translate uses of `int` to use the `I` carrier and
instructions in generated code where beneficial (and where required for
migration compatibility, such as in field and method descriptors).  The
descriptor `Qjava/lang/Integer;` will be used later when describing
specializations over primitives.  Conversions between `I` and
`Qjava/lang/Integer;` where needed in the translation will be inserted by the
compiler.

This scheme involves one behaviorally incompatible change: synchronization on an
instance of one of the primitive wrapper classes will throw `IMSE`.  While we do
not take this lightly, it is almost universally an error to do this, and in the
very few cases where this is not an outright error, simple workarounds exist.
