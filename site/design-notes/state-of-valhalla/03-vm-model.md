# State of Valhalla

#### Section 3: JVM Model
#### John Rose and Brian Goetz, December 2021

::: sidebar
Contents:

1. [The Road to Valhalla](01-background.html)
2. [Language Model](02-object-model.html)
3. [JVM Model](03-vm-model.html)
:::

This document describes the _Java Virtual Machine_ view of value classes.  Note
that this is not necessarily the same as the Java Language view; readers are
advised to exercise care in drawing conclusions about the Java Language, from
this document.

## Value classes and objects

Prior to Valhalla, all objects -- instances of classes and arrays -- had a
unique _object identity_.  Valhalla allows classes to choose, by marking a class
with the `ACC_VALUE` flag, whether their instances have identity ("identity
classes") or not ("value classes").  Instances of value classes are called
_value objects_.  Whether an obejct is an identity object or value object is
also reflected in the dynamic type system; identity objects are instances of
`java.lang.IdentityObject`, and value objects are instance of
`java.lang.ValueObject`.  

As with identity objects, value objects are referred to through _object
references_.  For some value classes -- those additionally marked with the
`ACC_PRIMITIVE` flag -- these can also be described directly, as _bare
values_.  

Because the JVM knows that value objects do not have identity, it can more
effectively optimize layout, instantiation, member access, and calling
convention for value objects, whether represented via reference or as bare
values.

#### Carriers and descriptors

JVM type descriptors (JVMS 4.3.2) use a leading letter to denote their _basic
type_.  There are currently eight basic types corresponding to the eight
primitive types (`I`, `J`, etc), and one basic type corresponding to object
references (`L`).  (The letter `V` appears in method descriptors but is not
considered a type.)  The character `[` in type descriptors introduces an array
type that is hardwired to an internally defined array class, but for our
purposes it is a variation of the `L` syntax, introducing a special type of
object reference (which always refers to an identity object.)

These nine basic types map to five _carrier types_ -- int, long, float, double,
and object reference -- which correspond to the different kinds of values that
can be stored in a stack or local variable slot.  (We erase `byte`, `short`,
`char`, and `boolean` to `int`, using the `I` carrier and `i*` bytecodes for
these.)  The primitive carriers store a value directly in the stack or local
variable slot (long and double values use two adjacent slots), and the object
reference (`L`) carrier stores an _object reference_, whether for an identity or
value object, in the corresponding slot.

To describe bare values of primitive types, we add a new basic type, `Q`.  
`Q` descriptors have the same syntactic structure as `L` descriptors (e.g.,
`Qjava/lang/int;`).  Both `Q`-carried and `L`-carried values can be operated on
by the same set of `a*` bytecodes.  The verifier ensures, however, that
`Q`-carried and `L`-carried values, even of the same class, are kept distinct.  

`Q` types can be referred to from `Constant_Class_info` by specifying a a field
descriptor for a `Q` type as its payload, rather than an internal binary name.

It is an error to refer to a class using a `Q` descriptor which does not have
the `ACC_VALUE` and `ACC_PRIMITIVE` flags set; only primitive classes support
bare values.  Any class may be referred to using an `L` descriptor.  (In
particular, there is no legal use for `Qjava/lang/Object;`.)

While Valhalla has had `L` and `Q` descriptors since the beginning, their
interpretations has gone through substantial evolution.  Initially the
distinction was flat vs pointer; then it was nullable vs not; then it was
"exotic" vs "traditional", and finally it has come to land at value vs
reference. This final landing may seem superficially similar to where we started
(flat vs pointer), but there is a significant difference: flat vs pointer
prescribes a particular _representation_, whereas value vs reference describes a
semantic distinction (from which we can derive the desired representational
properties.)  

Bare values have several major differences from references to value objects:

 - References to value objects, like all object references, can be null, whereas
   bare values cannot.  This allows a `Q` type to "work like an `int`",
   without troubling the JVM to represent a `null` value in a 32-bit integer.  

 - Loads and stores of references are atomic with respect to each other, whereas
   loads and stores of sufficiently large bare values may tear under race, as
   is is this case for `long` and `double`.  

 - The JVM is required to load classes named in `Q` descriptors much earlier
   than those named by `L` descriptors.

 - Data structures _may not_ be circular if linked via `Q` descriptors: a class
   `C` cannot reference `QC;` in its layout, either directly or indirectly.
   References must be used to break circularities.

 - If the class operand of the `checkcast` bytecode is a reference type, then
   the class is only loaded if the operand is non-null; if it is a `Q` type,
   then the class is eagerly loaded in any case.

 - `L` descriptor names participate in class loader constraints; `Q` descriptors
   names do not.  (Because of eager loading, the constraints can be checked
   using resolved classes when methods are prepared.)

The expectation is that bare values will routinely be flattened in object and
array layouts, and in calling conventions between methods.  Perhaps
surprisingly, _references to_ value objects can also be flattened under some
conditions as well, though their stronger semantic guarantees (nullity,
non-tearability) will have some cost.

#### Eager loading

In most cases, the JVM is lazy about loading classes referred to by other
classes, such as classes named in method and field descriptors or in
`Constant_Class` constants.  However, the JVM ensures that a classes superclass,
and its implemented interfaces, are loaded eagerly during the loading of a
class.

Valhalla adds two new triggers for eager class loading.  Any class referred to
via `Q` descriptor in the descriptor of a method or field declaration is eagerly
loaded in a similar manner to supertypes.  Additionally, any classes identified
in the `Preload` attribute on a class are preloaded in the same manner.  

```
Preload {
    u2 attribute_name_index;
    u4 attribute_length;
    u2 number_of_types;
    u1 preload_types[number_of_types];
}
```

These additional preload triggers are present so that by the time we have to
make layout decisions for references to value objects, or for bare values, we
have complete information about these classes.  (As a result, applications are
at greater risk of _bootstrap circularities_ because of increased preloading.)

#### Nullity and default values

Object references may be null, which means `L`-described variables must be able
to represent null.  The JVM has complete latitude over how to represent null
references.  For identity classes, where pointers are effectively a forced move
for representing references, the null pointer is the obvious representation.  
For value classes that are going to be flattened, an alternate representation
must be chosen.  Where the JVM can identify an unused bit or bit pattern (such
as slack bits in pointers or booleans), it can use these to encode null; in the
worst case, it can inject additional bits into the layout.

Heap variables -- fields and array elements -- must be initialized to default
values before they are initialized or read by user code.  For all reference
types, the default value is `null`; the built-in primitives each have their own
form of "zero" which acts as a default value.  for bare values, the default
value is the one for which each field is initialized to its default value.  

#### Supertypes

Value classes may implement interfaces, and may extend certain restricted
abstract classes as well as the special class `Object`.  This means that a
variable of an interface type, a suitable abstract class types, or `Object` may
be a reference to _either_ an identity object or a value object.  Value classes
do not have constructor methods; instead, source level constructors are
translated to static factory methods with the same argument list as the declared
constructor, with method name `<new>`. Unlike an `<init>` method, which operates
on a partially-built receiver instance and must return `void`, a static factory
method must return the `Q` descriptor type of the class being constructed.  (A
static factory method named `<new>` is also allowed to return `Object` or
perhaps one of the supertypes of that class, as an `L` descriptor; this is
necessary for hidden value classes.)

An abstract class is a _value-superclass candidate_ if it has no fields, its
no-arg constructor has no body (marked `ACC_ABSTRACT`), it has no other
constructors, and its superclass is either `Object` or a value-superclass
candidate. Value-superclass candidate are identified by the `ACC_VALUE_SUPER`
flag. (The static factory of a value class does not call the superclass
constructor; this can be safe only if the constructor is empty anyway.)

The requirements of being a value-superclass candidate are "contagious" up the
superclass chain.  That is, if some abstract class has an empty no-arg
constructor, its immediate superclass must _also_ have an empty no-arg
constructor.  It would seem to follow that even `Object` must have a empty
(`ACC_ABSTRACT`) constructor, but `Object` is special, because it alone is
allowed to have a truly empty concrete constructor method.  (Thus, `Object` is
exempt from the "contagion" of the requirement of empty constructors.)

Like interfaces, `Object` commits to neither kind of class; it implements
neither of the marker interfaces `IdentityObject` nor `ValueObject`.  Yet the
existing behavior of the expression `new Object()` creates an identity object.
The JVM will give special treatment to the bytecode instruction `new
java/lang/Object`, which after link resolution will create an identity object of
a secret substitute class which has no fields (like `Object`) and implements
`IdentityObject`.  (The verifier does not have to know about this, and the
subsequent `invokespecial` of `Object::<init>` will be the only initialization
the object ever experiences. This secret class will be designed to allow this
treatment.)

If an identity class declares a value-superclass candidate as its super, it must
(as usual) call the superclass `<init>` method, but the JVM links this call
straight up to the no-arg constructor of `Object`, skipping the intervening
abstract (empty) `<init>` methods.

#### Restrictions

Value classes come with some restrictions compared to their identity
counterparts.  Instance fields of a value class must be marked `ACC_FINAL`, and
must not be marked `ACC_VOLATILE`.

Value classes are implicitly adjusted by the JVM, as necessary, to implement the
interface `ValueObject`.  They or their supers may explicitly declare
`ValueObject` as a super, in which case the JVM does not insert the extra super.
They may not directly or indirectly implement the interface `IdentityObject`.

Also, abstract classes that fail to be value-superclass candidates (perhaps
because they contain fields) are implicitly adjusted by the JVM, as necessary,
to implement the interface `IdentityObject`.  They may not directly or
indirectly implement the interface `ValueObject`.

In this way, every concrete object implements exactly one of `IdentityObject` or
`ValueObject`, and every abstract class implements at most one of the two.
Most interfaces, and `Object`, will implement neither.

JVM-inserted supers are visible to reflection via `Class::getInterfaces` and
`Class::isAssignableFrom`, and respected by `checkcast`, `instanceof`,
`aastore`, `Class::isInstance`, and `Class::cast`.  

If a value class extends a class other than `Object`, that class must be a
value-superclass candidate.  The instance fields of a value class `Foo` may not,
either directly or indirectly, have a field of type `Foo` using a `Q`
descriptor.  (Such an object would have no definite size; each one would contain
an infinite regression of sub-objects of the same type.)

An `L` descriptor can be applied to a value class in any case; any infinite
regress is stopped immediately by a `null` reference.

A `static` field of type `Foo` using a `Q` descriptor in `Foo` is not
problematic, although it requires careful preparation, and (in general) a hidden
indirection to manage lazy initialization.  Such `static` fields of type `QFoo;`
must be initialized to the default value of `Foo`, but the JVM may need to
prepare such static fields before the default value is materialized, which may
possibly require lazy or indirect access to the field.  If so, this internal
logic can be hidden within the `getstatic` bytecode.

#### Static translation

The translation of Java classes which declare and use value classes requires
minor adjustments to the current translation strategy.  

Value classes that are not primitive classes require no change in translation;
these are translated with `L` descriptors.  Similarly, the reference companion
type of a primitive class is translated with `L` descriptors, and primitive
value types are translated with `Q` descriptors.  Widening and narrowing
conversions between `Q` and `L` types are performed with `checkcast`.  

Instantiation of value classes (`new V(...)`) is translated as invocation of the
corresponding factory method with `invokestatic`.  In the body of the factory
method, rather than accepting an uninitialized `this` as a parameter, `this` is
initialized inside the body of the factory with `aconst_init`, assignments to
fields of `this` are translated as modifying that value with `withfield`, and
the factory returns the completed `this`.

## Bytecodes and constants

The incorporation of value classes, and of `Q` descriptors, affects a number of
bytecodes.  Some are new, and others gain additional behavior or restrictions
when applied to primitive types.  The existing `a*` bytecodes are extended to
uniformly support references to both identity and value classes.

There are two new bytecode instructions used for constructing instances of
value classes.

 - `aconst_init` is the analogue of `new` for value objects; it leaves a
   reference to the intial value for a value class on the stack.  This initial
   value is guaranteed to not be equal to `null`.  The sole operand of this
   bytecode is a reference to a `CONSTANT_Class` item giving the internal binary
   name of the value class (not its `Q` descriptor).

 - `withfield` is the analogue of `putfield` for value objects; it takes a
   reference to a value object and a new field value on the stack, and leaves  a
   reference on the stack to an instance of the value class whose fields are all
   the same as the original instance, except for the specified field.

Both the `aconst_init` and `withfield` instruction are restricted and can only
be executed within the nest of the class that declares the value class being
initialized or modified.  This encapsulates creation of novel values, so the
class can ensure that any value escaping the implementation is seen to adhere to
its invariants.  The `new` and `putfield` bytecodes may not be used with
instances of primitive classes, and the `withfield` bytecode may not be used
with identity classes.  (In a future version of a VM, we may allow a class to
grant access to these restricted instructions to other clients, on an opt-in
basis, if this seems useful.)

The `anewarray` instruction is similarly extended to apply to a `Q` descriptor;
this determines whether the resulting array is flattened (`Q`), in which case it
is an array of bare values, or not (`L`).  The `anewarray` instruction
returns a plain (`L`) reference to an array regardless of its component type.

The `aastore` instruction performs a null check on the element value before
storing a value into an array of bare values.  The `aaload` instruction, as
always, takes an array of a particular reference element type, and returns an
element of that type.  If the array element type is a `Q` type, the value pushed
on the stack by `aaload` is tracked by the verifier as a `Q` type.  The
`aastore` instruction, as always, relies fully on the dynamic array store check;
the verifier is indifferent to the precise component type of the array input to
`aastore`.

The `acmp*` instructions perform a more sophisticated comparison between their
operands.  Two object references are consider equal if they are both `null`,
both references to the same identity object, or both references to primitive
objects of the same type and all of their fields are "equivalent".  Equivalence
means recursively applying a suitable equality comparison to the fields, which
uses ordinary equality for primitives (except for `float` and `double` fields,
which are compared with the semantics of `Float::equals` and `Double::equals`)
and `acmp` for references.  

The `checkcast` instruction is extended to apply to a `Q` descriptor, with the
existing `checkcast` interpreted as following the rules for `L` descriptors.  A
`checkcast` with a `Q` descriptor differs from the traditional semantics, in
that the class constant is resolved even if the object reference is null.

The `ldc` instruction, applied to a `CONSTANT_Class` item which contains a `Q`
descriptor, will return the secondary (bare) mirror for the class.  The normal
syntax (of a plain class name in a `CONSTANT_Class` item) will return the
primary (ref) mirror; again, it is as if the normal syntax works like an `L`
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

#### Arrays

An array type is a reference type (using the `L` contract) generated from a
descriptor beginning with `[`.  The addition of `Q` descriptors simply implies
that an array element type can be either `L` or `Q`, corresponding to a choice
between an array of references, or an array of bare values.

An array of bare values loads values of the corresponding `Q`-carried type. A
referance value of any kind (whether `Q` or `L` or null or array) may be
presented to the `aastore` instruction to store into the flattened array, but
the store check will apply the same logic as `checkcast` to ensure that the
incoming value can actually be stored as a bare value. 

Arrays of reference types are covariant; if `C` is a subtype of `D`, then `[LC;`
is a subtype of `[LD;`.  For primitive classes, we extend this covariance such
that an array of bare values is seen as a subtype of an array of references
of the corresponding type: `[QX;` is a subtype of `LX;` (and, since reference
arrays are covariant, `[QX;` is a transitively subtype of `[LObject;`.)  The
same consideration is given to the eight primitive types and their wrappers;
`[I` is a subtype of `[LInteger;`.  

#### Structure tearing

With the exception of `long` and `double`, loads and stores of references and
built-in primitives to and from the heap are atomic with respect to each other.
This provides protection against _out of thin air_ reads; even in the presence
of data races, a load of a variable will yield a value that _some_ thread has
put there (perhaps the default value), even if it is not the _most recent_ value
written to that variable.  (This, in turn, provides the underpinnings of
_initialization safety_ (JCiP 16.3), which allows property constructed immutable
objects to be shared published across threads, even via a data races.  (Without
this guarantee, it would be possible to observe the value of a `String` change
unexpectedly.)  

The larger-than-64-bit built-in primitives (`long` and `double`) are permitted
to _tear_ in a controlled way under race; absent proper coordination, a reader
may see the high-order 32 bits of one write, and the low-order 32 bits of
another write.  Any _happens-before_ ordering between writes and reads, such as
making the variable `volatile`, is sufficient to inhibit tearing.  Allowing
tearing for numerics was a pragmatic tradeoff given the hardware of the day
(today's JVM's typically don't tear even when they are allowed to, because
64-bit atomicity is cheap enough.)  For the built-in 64-bit primitives, tearing
is inconvenient but in practicality has a limited impact; numeric-intensive code
tends to not make extensive use of contended sharing.

While reads and writes of references are still atomic, meaning that we will not
see tearing for `L`-carried values, the same forces push us to allowing tearing
on larger bare values.  Thus, a thread can witness an "impossible"
combination of fields in a bare value, given the right (or wrong!) race
condition.  Since primitive classes may want to carefully control their internal
invariants, structure tearing can cause new kinds of failures if it violates
those invariants; this is often a cue to use value classes rather than primitive
classes.

Structure tearing may not appear on some platforms.  On other platforms it may
be rare, although mischievous code might force it to happen on command.  In any
case, tearing cannot be ruled out absolutely.  Java provides several means of
ensuring tear-freedom; declare the field `volatile`, or use a reference rather
than a bare value.  (The former is only available for fields; array elements
must choose the latter.)  The JVM then excludes tearing by any means it chooses:
Software transactions, hardware locking, hidden synchronization, extra
buffering, or some combination of the above.  

#### Legacy primitives

As discussed in [Language Model](02-object-model.html), a key goal is to migrate
the built-in primitives so that they are "mostly just" primitive classes, while
retaining the performance characteristics of the built-in primitives.  The
existing wrapper classes (e.g., `Integer`) are redefined as value classes. We
treat `int.ref` as an alias for `Integer`.  

The JVM will, of course, continue to have direct support for primitives (e.g.,
the `I` carrier, and the `i*` instructions).  The language compiler will have
substantial latitude to translate uses of `int` to use the `I` carrier and
instructions in generated code where beneficial (and where required for
migration compatibility, such as in field and method descriptors).  The
descriptor `Qjava/lang/Integer;` may be used later when describing
specializations over primitives.  Conversions between `I` and
`Qjava/lang/Integer;` where needed in the translation will be inserted by the
language compiler.

This scheme involves one behaviorally incompatible change: synchronization on an
instance of one of the primitive wrapper classes will throw `IMSE`.  While we do
not take this lightly, it is almost universally an error to do this, and in the
very few cases where this is not an outright error, simple workarounds exist.

#### Verification

The verifier keeps track of the "flavor" of each reference value (`L` vs `Q`) as
derived from the descriptor at its source.  `L` and `Q` references may be
interconverted by the use of the `checkcast` instruction; a non-null `LX;` can
be converted to a `QX;`, and a `QX;` can always be converted to an `LX;` (or one
of its supertypes.)

The verifier tracks `Q`-carried values, deriving them from from the following
sources:

 - any `Q` descriptors in the argument types of the current method
 - the `invoke` family (when the return value is a `Q` type)
 - `getfield`, `getstatic` (when the field is a `Q` type)
 - `aconst_init`, `withfield` (always produces a `Q` type)
 - `aaload` (when the array component type is a `Q` type)
 - `checkcast` (when the `CONSTANT_Class` operand is a `Q` descriptor)
 - `ldc` (when the constant is a `Q` type)

The verifier enforces the `Q` marking of values when it sends them to
the following consumers:

 - `areturn` (when the return value is a `Q` type)
 - the `invoke` family (any `Q` descriptors for outgoing arguments)
 - `putfield`, `putstatic`, `withfield` (when the field is a `Q` type)
 - `withfield` (the stacked object to update must be a `Q` type)

In some cases the JVM uses dynamic checks to ensure type safety, and the
verifier does not enforce marking of `Q` types for these consumers, but rather
accepts `Q` and `L` marked types indiscriminately:

 - the receiver of `getfield`, `putfield`, and the `invoke` family
   (these already include a mandatory `null` check)
 - `aastore` (this has always been purely dynamic)
 - `checkcast` (`Q` promotes to `L`, which is then cast)

In the verifier, the `null` type is a subtype of every plain (`L` descriptor)
type, but is not a subtype of any `Q` type.  (Note that as a result of these
rules, every `Q`-carried value is sourced from either a previous `Q`-carried
value, or else is the result of a `checkcast`.  These rules are sufficient to
ensure that stray `null` value can be excluded eagerly in the interpreter,
before they would pass into fields that are flattened or method arguments that
are scalarized, and even before an optimizer begins to scalarized calling
sequences.)

In the verifier, a type `QFoo;` has no subtypes or supertypes at all.  

#### Reflection

When a class is reflected, its field and method types directly represent the
corresponding descriptors by using a distinct mirror (of type `Class`) for each
distinct descriptor.  (There are even mirrors for `int` and `void`!)  And since
a primitive class `Foo` can appear with either kind of descriptor (`L` or `Q`),
it follows there must be two mirrors associated with any primitive class `Foo`.
The source types are denoted `Foo.ref` and `Foo`, which are translated as
`LFoo;` and `QFoo;`, and likely mirrored as `Foo.ref.class` and `Foo.class`. The
JVM generates both mirrors automatically when loading a primitive class.

In all paths through the reflective APIs which classically (before Vahalla)
yield class mirrors, the JVM will continue to return `ref` mirrors.  This is
more compatible than returning the "more accurate" value.  The situation is
similar, in some ways, to that of `Integer.class` and `int.class`, if one can
imagine that `Integer` works like `int.ref`, and that `I` is really (somehow) a
`Q`-like descriptor.  

In particular, `Class::forName` and `Object::getClass` will always return a
`ref` projection.  If you want the mirror that "works like `int.class`", you
have to look somewhere special, as with `int` today, or ask the primary mirror
for its corresponding secondary mirror.  There will be an API point on `Class`
for obtaining the `val` mirror from the `ref` mirror, and vice versa.

#### Reachability

Like one of today's `Integer` objects, a value object may be used as a key to a
weak hash map.  (There are many such in the Java ecosystem.)  The implementation
of a weak hash map probably appeals to the `WeakReference` JDK class or
something similar.  In any case, applications make varying use of
`WeakReference` and related types, as well as JNI weak references.  It appears
that we cannot exclude value objects from such uses, and so we need to assign a
meaning to a weak reference to a value object.

What does it mean to wait for a value object to become unreachable?  If the
primitive object is simply a bit-pattern of scalars, the unhappy answer is that
it is alway reachable in principle, since the bit-pattern could be read from
disk, loaded into a fresh copy of the value object, and presented as a witness
of reachability.  After all, a reference is reachable if some other code can
produce a reference which `acmp` will compare as indistinguishable from the
original reference.  But `acmp` on a value object pays attention only to object
state, not identity.

If a value object is composed of a combination of scalars and references to
identity objects, the problem is less hopeless, though still complicated. A
value object becomes unreachable (in the modified definition of the previous
paragraph) if and only if _at least one_ of its component identity object
references becomes unreachable.

It seems possible to implement this standard of reachability for value objects
using a combination of GC adjustments, JDK library changes to `WeakReference`,
and runtime code changes for JNI weak references. It also seems terribly
inconvenient to do this, and we are looking for use cases to help us weigh the
alternatives, such as refusing to build such `WeakReference`s (forcing library
authors to recode) or just holding on to value objects forever.

## Flattening and representation

It was clearly a goal of Valhalla to enable the flattening of bare values.  
Given the constraints on primitive classes, flattening bare values in the heap,
and in calling conventions, is straightforward; we can replace a bare value with
its component fields whenever it is convenient to do so, and reassemble them
into a value object where needed.  

Perhaps surprisingly, it is also possible to flatten some object references.  
Historically, object references were always represented as pointers.  Because
all objects had identity, there was really no other practical option; objects
with identity live in exactly one place, and a pointer is the ideal way to
record this. Similarly, when the referent is polymorphic (such as a reference to
an interface type) and all the possibilities are identity objects, a pointer is
again the natural representation.

When the type of the referent is more constrained, and the referent is known to
have no identity, alternate representations become possible. An object without
identity is just its (immutable) state (including its typestate), and two
objects with the same type and state are indistinguishable.  This means that it
becomes possible to flatten references into their referred-to state.  

There are a number of accidental reasons why we might not be able to flatten a
reference.  We might not have complete information about the range of types of
the referent (it could be a reference to an unsealed interface type).  And even
if it is a reference to a type that turns out to be monomorphic, we may not know
this at the time we are called upon to choose a representation.  

Because the layout of references must frequently be chosen early (such as when
choosing the layout for a class containing the reference, which is done at class
preparation time), optimizations involving reference layout often require _early
loading_ of their referent type.  However, the JVM specification strongly
encourages us to defer the loading of classes described with `L` descriptors
until the latest possible time (when an instance is instantiated, or a static
member is accessed.)  Accordingly, to enable flattened representations of
references, we needed to arrange for earlier loading of the referent type.

Another consequence of flattening the layout of references is that an alternate
strategy for representing nullity is needed.  References are intrinsically
nullable (a reference needs a way to express "refers to nothing", to avoid
infinite regress when constructing objects), and therefore we need to ensure
that there is a way to represent null references.  If we are representing null
as a pointer, as we do for identity objects, this is easy: use a sentinel value
(such as the null pointer) to encode the null reference.  But if we are
flattening the referent into the reference encoding, we'll need at least one bit
pattern to represent null.  (In some cases we can find an empty bit pattern in
the representation, in which case we can do so without additional footprint
cost; otherwise we'll need to adjoin additional bits to the representation.)

#### Flattennable contexts

References may appear in fields, method parameter and returns, and local
variables.  Each of these contexts has different constraints on the timing of
layout.  The layout of fields is done at the time the containing class is
prepared and is generally fixed thereafter.  The layout of method parameters and
return values is done at the time the invocation instruction is linked and is
similarly fixed thereafter, though it is allowable for a method to support more
than one calling convention.

Final fields containing value objects, under both `L` and `Q` descriptors, can
be routinely flattened.  Mutable fields containing value objects under `Q`
descriptors can be routinely flattened; mutable fields containing value objects
under `L` descriptors must provide atomic loads and stores, which requires
either atomic instructions or an indirection.  

When an array element or mutable instance field is of a `Q` type, the effect of
loading or storing a `Q` value is tracked by the Java Memory Model as if the
primitive object were recursively broken down into the component subfields of it
and all of its embedded sub-objects, all the way down to plain pointers and
scalar (non-object) primitives like `int` and `long`.  (In other words, a
flattened value in the heap is "scalarized", decomposed into a set of machine
words.)

Value objects as method parameters and returns, under both `L` and `Q`
descriptors, can use flattened calling conventions for out-of-line calls.  
For inlined calls, JITs have virtually unlimited latitude to flatten (scalarize)
value objects under both `L` and `Q` descriptors.  

In all cases, the VM has the latitude to not flatten at its discretion.  For
value objects with many fields, the VM may decide to fall back to the
traditional indirected representation.