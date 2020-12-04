## Specialized generics: classfile and translation

This document outlines an approach for representing optional generic type
information in the classfile, so that the VM can use it to perform code path
and layout specialization.  

## Overview

This approach is a refinement of the template classfile approach, but one which
leans more heavily on _optionality_ of type information, and which leaves much
more of the details to the language and translation scheme.

What they have in common is that they centralize variant information in the
constant pool, with the expectation of splitting the constant pool for each
variant of the class.  Accordingly, the constant pool can have _holes_ that
represent placeholders to be filled in later, and other constants can depend on
constants with holes, so that a hole that contains the value of `T` can be used
as an input to a constant pool entry for the species for `Foo<T>`.  

Where they differ is that the template classfile approach attempted to flow
transparent type information all the way into descriptors, representing erasure
as a specific type parameter; the current approach flows opaque type metadata
through optional side channels, leaving descriptors untouched, and represents
erasure as the lack of sharper type information.  The effect is that the JVM is
mostly in the business of plumbing, rather than interpretation, of type data  --
a much more comfortable role for the JVM.

#### Specializable entities

We say some classes or methods are _parametric_; this means they have a set of
parameters (which are generally types), whose arity and kinds are known to the
language and translation scheme, but not necessarily  to the JVM.  For example,
generic classes like `List<T>` are parametric, taking one type parameter.

We capture parametricity in the classfile with the `Parametric` attribute
on a `ClassFile` or `method_info`:

```
Parametric {
    u2 name_index;
    u4 length;
    u2 species_bootstrap;
    u2 parameter_constant;
}
```

The `Parametric` attribute says that the classfile entity it applies to is
parametric.  The `species_bootstrap` is used in resolving species of this
entity, and is an index into the `BootstrapMethods` attribute; the
`parameter_constant` is the index of the `Constant_Parameter` corresponding to
this entity.  Note that the `Parametric` attribute says nothing about the
parameters!  The JVM isn't interested in knowing how many or what kinds they
must be, or what constitutes a valid parameterization -- the bootstrap handles
that.  The JVM just knows that each valid parameterization corresponds to a
_species_ of the parametric entity.

Species can be created reflectively, and they can be stored in constant pool
entries using `condy`.  To resolve a species for entity X, the species
parameters are passed to a lookup method, which ensures that X is a parametric
entity, and invokes X's species bootstrap with the parameters.  The species
bootstrap validates the parameters, possibly adapting or normalizing them, and
if they are valid, invokes a privileged lookup to find if the desired species is
already cached, or create the desired species if it is not.  Species are
interned, so they can be compared with `==`.

A species for a parameteric class is of type `ClassSpecies`, which likely will
extend `java.lang.Class`, and a species for a parametric method is of type
`MethodSpecies`.  Both provide accessors for its associated parametric entity
and its species parameters.

Species can be used in a number of ways: as type tokens to describe a
specialization, reflectively to look up and invoke specialized members, and by
the JVM as a key to manage classfile specialization.

#### Specialized fields

Ultimately, specialized type data needs to flow into field declarations, so  we
can optimize the layout of specialized types.  Because specialized type
information is optional, specialized field declarations will have _two_ types; a
default (erased, fallback) type specified by the field descriptor, and a
specialized type to be used if type metadata is present.  The specialized type
is indicated with an attribute on the `field_info`:

```
Specialized {
    u2 name_index;
    u4 length;
    u2 specialized_type;
}
```

The `specialized_type` field refers to a `Constant_Class` in the constant pool.
If this constant is non-null, this type is used in preference to the resolved
descriptor type.  This type must be a subtype of the resolved descriptor type.

#### Holes in the constant pool

Each parametric entity has a special constant in the constant pool, which
represents a hole that is filled with a species of that entity.  

```
Constant_Parameter_info {
    u2 entity;
}
```

The entity is the constant pool index of the `this_class` entry (for the class
parameter) or a `Constant_MethodRef_info` (for a method parameter.)  For any
parametric entity, the hole will resolve to a species of that entity.  

The only thing the VM knows how to do with species is to compare them (are they
the same species) and check that a species that fills a hole is a species for
the right entity (and that entity is parametric.)  Everything is else about
flowing type information from species falls to the translation strategy, which
will likely use `condy` to extract information from the species.

Take the class:

```
class Foo<T> {
    T t;
}
```

We might compile this to:

```
#1 C_Class("Foo")      // this_class
#2 C_Parameter(#1)     // class species
#3 condy[JavaGenerics.GetTypeParam, #2, 0]:Class     // T
@Parametric(parameter_constant=#2,
   species_bootstrap=[JavaGenerics.ResolveSpecies, "(C)"])
class Foo {
    @Specialized(specialized_type=#3) Object t;
}
```

Since `Foo` is a parametric class, it gets a `Parametric` attribute, and a
corresponding `Constant_Parameter` constant to hold the species with which it is
instantiated.  The species is opaque to the VM, but it was manufactured by the
Java language runtime, so we can take it apart with the language runtime.  We
extract the `T` parameter -- which is a live type -- into CP entry 3.  We use
this as the specialized type of the `t` field.

The `Constant_Parameter` is a weird kind of constant, since it is a _per-species
constant_.  For each unique species of `Foo`, the JVM will split the constant
pool (but can still share the rest of the class file), so different species of
`Foo` can have different specialized field types, and from the perspective of a
given species, the species parameters are still constants.  In the Template
Classfiles model, there were explicit segments that were clear specialization
boundaries; the same concept applies here, except that they are implicit based
on dependency on parameters.  It is an error for a constant to depend (directly
or indirectly) on the parameter for multiple methods, or for a constant that
depends on a method parameter to be accessed from outside that method.

#### Enhanced types

So, how do the holes get filled?  We introduce one more constant pool form,
which takes another constant (`C_Class`, `C_MethodRef`) and enhances it with
species metadata:

```
Constant_Linkage_info {
    u2 underlying;
    u2 species;
}
```

Resolving a `C_Linkage` involves first resolving `underlying` and `species`.
If `species` resolves to `null`, then the linkage resolves to `underlying`,
otherwise it resolves to a special constant where `underlying` is decorated
with `species`.  (It is an error if `underlying` does not resolve to a
parametric entity, or if `species` is not a species of that entity.)  A
`C_Linkage` can be used in any of the places where a `C_Class` or `C_MethodRef`
can be used.  

The `new` bytecode, if presented with a decorated `C_Class`, instantiates an
instance of the corresponding species.  This is how information flows into the
holes.  The first time a new species is instantiated, the constant pool for its
underlying entity is split, and the `C_Parameter` entry in that split constant
pool is filled in with the species.  In our `Foo` example above, the specialized
type of the field depends on the `C_Parameter` for the class, so by specializing
`Foo` to a certain species, its field is automatically specialized to the first
type parameter of that species.

The type descriptors used for linkage of field access and method invocation are
"dead" strings that are statically computed by the language compiler; the type
metadata (if present) are live `Class` and `Species` objects which may well be
the result of dynamic resolution (take the class for `T`, derive the species
for `T[]` from that, and derive the species for `List<T>` from that.)  

#### Type metadata is optional

Everything about this scheme is optional.  If `C_Class` or `C_MethodRef`
constants are decorated with species, certain bytecodes and classfile metadata
may be refined with the species information, and if no decoration is provided,
the `C_Parameter` is filled with `null` and everything falls back gracefully to
the descriptors in the classfile.   So if we execute `new` with an undecorated
class, we'll get an instance with no species, whose `t` field is of type
`Object`.

The type checking bytecodes (`checkcast` and `instanceof`) have a `C_Class`
operand,  and also operate on a dynamic instance of some class.  The `C_Class`
operand may, or may not, have species metadata; similarly, the instance may, or
may not, be an instance of some species.  If both the operand and the instance
have species metadata, a sharp comparison is done (involving both class and
species);  if one or both are missing species metadata, then a dull (erased)
comparison is done, involving only the class.  A checkcast to `Foo<T>` is really
checking if the target is either a `Foo` specialized to `T`, or an erased `Foo`
-- but never `Foo` specialized to some other type `U`.

#### Generic methods

We've seen one way that a constant pool hole gets filled -- instantiate a
species of a parametric class.  The other way is to invoke a generic method.
Generic methods are parametric entities too; they are represented by species
that are constructed in the same way as class species -- using a bootstrap
provided by the language runtime that generated the class.  

A generic method in Java:

```
static<T> Foo<T> makeFoo() { return new Foo<T>(); }
```

has, like `Foo`, a single type parameter.  To perform a specialized invocation
with `T=String`, the client resolves the species constant for `Foo::makeFoo`
with `T=String` (with a `condy` that calls into reflection), uses a `C_Linkage`
that decorates the `Constant_MethodRef_info` with the method species as the
operand of the `invokestatic` instruction.  Just as with class species,
invoking a specialized method with a new species splits the constant pool, and
fills the corresponding hole with the species.  The type information from the
species can then flow into the decorated `C_Class` constant that is the operand
of the `new`.

```
#1 C_MethodRef("X", "makeFoo", "()Foo")
#2 C_Parameter(#1)
#3 condy[JavaGenerics.GetTypeParam, #2, 0]:Class     // T
#4 C_Class("Foo")
#5 condy[Species.makeSpecies, #4, #3]:ClassSpecies   // Foo<T>
class X {
    @Parametric(parameter_constant=#2,
                species_bootstrap=[JavaGenerics.ResolveSpecies, "(C)"])
    static Foo makeFoo() {
        new #5
        dup
        invokespecial Foo:<init>()
        areturn
    }
}
```

Here, we have a "hole" for the method species for `makeFoo` (CP#2), we extract
the type `T` from it (CP#3), we construct the species `Foo<T>` from that (CP#5),
and use that as the operand of the `new` bytecode.  If the hole is filled with a
species, then the method will return a specialized instance of `Foo`; otherwise
the hole will be filled with `null`, which will propagate through the runtime to
result in a `null`-decorated (equivalent to undecorated) class constant for
`Foo`.

The role of the JVM here is plumbing; the client constructs an opaque species,
and uses that to decorate a method call, which signals the intent to pass the
type information through the call chain, to where it is ultimately consumed  by
bytecodes such as `new` or class metadata such as specialized fields.  Our main
means of passing type information around are through the receiver (which may
already be a specialized instance), by decorating a `new` bytecode, or by
decorating an `invoke*` bytecode.  

#### Species resolution and binary compatibility

When we declare a parametric entity in some source language, there are likely to
be constraints (arity, kind, bounds) on its parameters.  We don't want the VM
interpreting those constraints; we want the language runtime to handle this.
This is why the resolution of a species is a two step process, where the client
nominates arguments, a language- (or entity-) specific bootstrap validates and
possibly normalizes them, and then asks reflection to make or look up the
appropriate species mirror.  The bootstrap can check the arity and kinds of the
arguments, check bounds, etc.  (The arguments don't even have to be classes;
they can be any reasonable constant, such as strings or integers, as long as by
the time the type information flows into specialized fields, it has been mapped
to a live type.)  

Further, the bootstrap is free to normalize the arguments as it sees fit.  If,
for example, Language X decides that `A` and `B` are equivalent types, it is
free to normalize species instantiations involving `B` to the one involving `A`.
If a language wants to support an explicit versioning scheme for
parameterization, a version token can be included in the species arguments which
can help to interpret what species is being asked for.  And if a language
supports certain binary compatible evolutions (i.e., it is OK to add parameters
at the end of the argument list), it can detect "old" parameter lists and map
them to the corresponding "new" one.

#### Classes and species

Every object has a class; some objects additionally have a species (if their
`new` bytecode was decorated with linkage information that resolved to a
species).  Classes have a class mirror (`java.lang.Class`); species have a
species mirror (which may be a subtype of `Class`.)  For any species `S` of
class `C`, `S <: C`.  The class is the natural type of wildcard types, raw
types, and generics that are erased at compile time, and a supertype of the
natural type of species of the class.

## Translation

Every language has its own translation scheme from source files to class files,
such as mangling method names according to some encoding.  As long as the same
translation strategy is used at both the client and declaration, we don't notice
-- We only notice when something goes wrong.

> linkage nearly always involves a conspiracy between a translation scheme and
> itself.

While compiling Java source to classfiles, Java erases type descriptors -- this
is part of its translation scheme, as is mangling the names of some artifacts
(such as  inner classes).  This is fine for Java, but we don't want to force
this translation scheme on other languages (unless they choose to explicitly opt
into it for purposes of interoperability.)  We want a mechanism that works well
with a broad range of translation schemes; the more we can move out of the JVM
and into the translation scheme, the more flexibility we have, and the easier it
is for JVM languages -- Java and others -- to evolve.

As we move forward, we are envisioning a translation scheme that involves:

 - Generic classes and methods gain a `Parametric` attribute;
 - Fields that are variant in their layout (those whose types are given by type
   variables, or by species of inline classes) gain a `Specialized` attribute;
 - Method and field descriptors continue to be erased;
 - Instruction sites for `new`, `checkcast`, `instanceof`, and `invoke*` may
   gain linkage metadata;
 - Supertypes may gain linkage metadata.

Additionally, some of the translation decisions can be moved from compile time
to linkage time, through the use of language-runtime bootstrap methods invoked
via `condy`.  

#### Example: partial erasure

Suppose we have

```
inline class Bar<T,U> {
    T t;
    U u;
}

class Foo<T> {
    Bar<T, int> b;
}
```

What is the specialized type metadata for `b` in an erased (no-metadata) `Foo`?

```
#1 = Class[Foo]
#2 = C_Parameter[#1]
#3 = Class[Bar]
#4 = Class[int]
#5 = condy[JavaGenerics.GetTypeParam, #2, 0]:Class     // T
#6 = condy[Species.makeSpecies, #3, #5, #4]:ClassSpecies   // Bar<T,int>

class Foo {
    @Specialized(#6)
    Bar b;
}
```

So what happens here, when `Foo` is erased?  CP#5 will contain `null`, since
there is no `T` available; this lack of type information can flow into
`Bar<T,int>` ("`Bar` with nothing for `T`, and `int` for `U`"), and can flow
further into the specialization for `Bar`, in which case `u` will have a type
specialization but not `t`.  (This is just one possible treatment by the
translation scheme; we could choose to fully erase partially erased types, as we
did in M3, but this seems better.  Since the translation scheme provides the
bootstraps, it is free to decide if and how to propagate the lack of metadata.)

#### Example: nested members

In Java, we can nest classes in classes and classes in methods (and maybe
someday, methods in methods).  Any of these can have type parameters.  

```
class Outer<T> {
    class Inner<U> {
        <V> method() {
            class Local<W> { }

            ... new Local<int>(...) ...
        }
    }
}
```

Unless one of the classes or methods in that chain is static, nested entities
have access to the type parameters of all the enclosing entities.  In fact,
these are implicit type parameters; we said `new Local<int>`, but what we really
meant was `new Local<T, U, V, int>()`.  (Actually, its worse than this, because
these names can be shadowed, but they are still there.)  Nesting is a feature of
the Java language, as are the rules for which enclosing type parameters are
accessible, and we'd like to keep it that way.

Having a common abstraction for "specialized class" and "specialized method"
helps us a lot here, because then we can have a uniform chain of "enclosing
parametric entity" all the way up, without having to distinguish between
enclosing classes and methods (this was particularly annoying in earlier
explorations.)  Another challenge in previous explorations had to do with
disambiguating type parameters under separate compilation anomalies.  Suppose
`Inner` was originally not generic, and a client was compiled at that time, but
later `Inner` acquired type parameters -- this should be binary compatible.  If
we used position only to assign parameters to various enclosing contexts, we'd
have a mismatch; we want to track each enclosing context separately.

Having a concrete _witness_ to an enclosing specialization -- the class or
method species -- gives us a clean way to represent this chain of nesting.  In
addition to the declared type parameter `W` of `Local`, because it is nested, it
acquires a synthetic species parameter, which should be a method species for
`method()`.  In turn, `method()` has a synthetic species parameter for
`Inner<U>`,  which has a synthetic species parameter for `Outer<T>`.  The JVM
doesn't care; the language generates the proper `condy` invocations to unpack
the enclosing type parameters.

In the case where `Inner` acquired an enclosing parameter, we might encounter
clients who provide either no enclosing species for `method()`, or an enclosing
species for `Outer<T>` instead of `Inner<U>`.  If we include enough metadata
about the expected form of the descriptor in the classfile (which can be
provided as static bootstrap arguments, or extracted from the `Signature`
attribute, or some other means) the language runtime can infer that `U` is
erased (but could still use the specialization of `T` if available), by having
the species-resolution bootstrap be tolerant of these sorts of mismatches.

We can translate the above as:

```
#1 = C_Class["Outer$Inner"]
#2 = C_Parameter[#1]
#3 = C_MethodRef[#1, "method", "()V"]
#4 = C_Parameter[#3]                // Method species for method()
#5 = C_Class["Outer$Inner$1Local"]
#6 = condy[[Species.makeSpecies, #5, #4, int.class] // Local<int>
#7 = C_Linkage[#5, #8]
@Parametric(parameter_constant=#2,
            species_bootstrap=[JavaGenerics.ResolveSpecies, "(SC)"])
class Outer$Inner {
    @Parametric()
    void method() {
        new #7
    }
}
```

In this example, the species parameters for the instantiation of `Local` are the
species for the enclosing method, plus Local's type parameter `W`; in turn, the
parameters for the species for the enclosing method include the species for
`Outer<T>.Inner<U>`, plus the method type parameter `V`.  If  we need access to
`T`, `U`, or `V`, we can extract them from the method species (which might
involve multiple rounds of extraction.)

## Generic inline classes

Specializable inline classes present a fresh challenge, in that we specialize
_fields_, but we move entire inline objects with `aload` or `getfield`.  When an
inline class contains specializable fields, this has consequences for the
semantics of the data-movement instructions `aload`, `astore`, `putfield`,
`getfield`, `aaload`, and `aastore` when they are moving inline objects.

Consider an inline class `V<T>`.  Specialized fields of type `V` will have a
descriptor of `QV;`, along with species metadata; that species metadata can
affect the layout of the fields of `V` itself.  However, we also need a
translation target for `V<?>`, for which the logical translation is `QV;` with
no metadata (effectively, an erased `V`).  If we have:

```
inline Class Two<T> {
    T t;
    T u;
}

class Foo {
    Two<int> ti = ...
    Two<?> tq = ti
}
```

then they layout of `ti` is two `int`s, and the layout of `bq` is two `Object`s.
But we are allowed to assign a `Box<int>` to a `Box<?>`, which means that for
variant (generic) inline classes, this assignment must be done fieldwise, with
adaptations between the fields if necessary.  (This sort of "deep fieldwise
operation" is not unlike how equality (`ACMP`) is defined on inlines.)

## Invoking generic instance methods

Suppose we have:

```
class Foo {
    <T> m(T t) { }
}

class Bar extends Foo {
    @Override
    <T> m(T t) { }
}
```

and a client invokes `foo.<int>m()`.  So the client will find the method species
for `Foo::m` with type parameter `T`, and decorates the `C_MethodRef` for
`Foo::m` with the appropriate species for the `invokevirtual` call.  Selection
chooses `Bar::m`.

At this point we have a problem; we have a method species for `Foo::m`, but  we
need one for `Bar::m`.  


## Reflection APIs

## Compatibility

[ descriptors stay the same as before -- yay compatibility ]
[ maybe some new casts and typecheck points, so maybe more CCEs ]
