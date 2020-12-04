# State of Valhalla

#### Section 4: Translation
#### Brian Goetz, Mar 2020

::: sidebar
Contents:

1. [The Road to Valhalla](01-background.html)
2. [Language Model](02-object-model.html)
3. [JVM Model](03-vm-model.html)
4. [Translation scheme](04-translation.html)
:::

This document describes how inline classes, and operations on inline types, are
translated from Java source code to Java classfiles.

## Class declarations

An inline class declaration can have superclasses (either `Object` or a suitable
abstract class) and superinterfaces:

```
inline class C
    extends A
    implements I { }
```

Upon compilation, this gives rise to two classfiles, `C$ref` and `C$val`,
corresponding to the source-level types `C.ref` and `C.val`, which look like:

```
[ PermittedSubtypes(C$val.class) ]
[ NestMembers(C$val.class) ]
abstract sealed class C$ref
    extends A implements I
    permits C$val {
}

[ NestHost(C$ref.class) ]
ACC_INLINE class C$val extends C$ref {
}
```

Further, this gives rise to a type `C`, which is an alias to one of the above
types (depending on whether `C` is `ref-default` or `val-default`).  If `C` is a
`ref-default` class, then we must translate the reference projection to a
classfile called `C`, not `C$ref`, because one motivation for reference-default
inline  classes is compatible migration from identity classes.  (Alternately,
we could always call one of the classes `C` and the other `C$ref` or `C$val`.)

The use of sealing here means that any value of class `C$ref` will either be a
reference to a value of `C$val`, or will be the null reference (effectively,
`C$ref` is the adjunction of `C$val` with the distinguished value `null`.)
Additionally, the two generated classes are _nestmates_, so that they can access
private members of each other.

#### Members

The members declared in `C` become members of both `C.ref` and `C.val`.  There
are several possible ways to translate the members of the class declaration into
members of the projection classfiles.  We choose an approach that leans into
what the VM already does, minimizing duplication and the need for translation
artifacts such as bridge methods, and minimizing future migration compatibility
issues.  This approach sorts state-related members -- fields and constructors --
into the value projection, and behavioral members -- methods -- into the
reference projection.  Fields are implicitly final, and constructors at the
source level are mapped to factory methods in the classfile.

```
inline class C {
    private int x;

    public C(int x) { this.x = x; }

    public int x() { return x; }
}
```

When we translate this, the members are sorted as follows, with various
adjustments in the translation of the method bodies to account for this
translation:

```
abstract sealed class C$ref permits C$val {
    public int x() { return ((C$val) this).x; }
}

ACC_INLINE class C$val extends C$ref {
    private final int x;

    public static void <new>(int x) {
        defaultvalue C$val
        iload_0
        withfield C$val.x:int
        areturn
    }
}
```

Static members are translated onto to the reference projection.  

#### Type uses and member access

In most cases, such as field descriptors and method descriptors, uses of `C.ref`
is translated as `LC$ref;`, uses of `C.val` is translated as `QC$val;`, and uses
of `C` are translated into one of these based on whether `C` is declared as a
ref-default or val-default inline class.

Access to all instance members of `C$val`, and method access on `C$ref` are
translated as usual (`getfield`, `invokevirtual`, etc); field access through
`C$ref` is translated by first casting to `C$val` (the two will have the same
accessibility), and accessing the field through the value projection.
Instantiation of `C$ref` is translated as instantiation of a `C$val` followed by
a cast to `C$ref`.  Accesses to static members through `C$val` are translated by
widening to `C$ref` and accessing the members there (this handles private static
fields too); the same is true for invocation of private methods (since they are
not inherited.)  These translation artifacts are transparent to the user; the
user sees both projections as having all the members declared in `C`.

This translation scheme has been chosen to achieve two goals: minimal
duplication of members in the classfiles (such as abstract or bridge methods),
and not conditioning translation decisions on `val-default` vs `ref-default`
(except for the meaning of the unadorned name) or other characteristics which
could create new binary compatibility constraints.

## Conversions and subtyping

The relationship between the value and reference projections differs
between the language model and the translation to the VM model.  In the
language, the value projection is not a subtype of the reference projection;
instead, the two are related by _inline narrowing and widening conversions_,
whereas in the VM, the two are related by actual subtyping.  

It may seem strange to have deliberately picked a different model in the
language as in the VM, but as we'll see, this choice leaves language constraints
in the language (where they belong), and lets the VM focus primarily on being
an efficient translation target.

The inline narrowing and widening conversions are semantically similar to the
more familiar unboxing and boxing conversions, but do not share many of the
negative performance costs associated with boxing (indirection, allocation,
accidental identity.)  When we convert a `C$val` to a `C$ref`, while this is
considered a conversion at the language level, no actual bytecode need be
emitted for this conversion, since the VM already sees `C$val` as a subtype of
`C$ref`.  In the other direction, the language compiler will emit a `checkcast`
bytecode when narrowing a `C$ref` to a `C$val`, but because the VM knows that
`C$ref` is sealed to permit only `C$val`, this can be lowered to a null check
(or completely optimized away.)  So what look like boxing and unboxing at the
language level turn into no-ops and near-no-ops in the VM.

Still, why did we not choose to simply expose the subtyping between `C$val` and
`C$ref` at the language level?  Because we wanted to preserve a path to unifying
primitives and inlines at the language level.  Currently, in the language, we
have `int` and `Integer` related through ad-hoc boxing and unboxing conversions.
As we outlined in [Language Model](02-object-model.html), we would like to
migrate the primitives to be ordinary inline classes, with the legacy box types
being the reference projection for those classes -- having two different kinds
of "boxes" (a legacy heavy box and a new lighter box, where the former is
related via boxing and the latter via subtyping) for primitives would be a
permanent wart, and having a new, second set of conversions (subtyping) might
incompatibly change existing overload selection decisions.  To remove this wart,
we preserve the semantics of the conversion between `int` and `Integer`, and
extend it to all inline widening/narrowing conversions, so that the legacy
boxing conversion is just an ordinary inline widening and narrowing.

Further, the boxing relationship plays a role in type inference and overload
selection, and by choosing inline widening and narrowing semantics that match
that of boxing,  these complex and pervasive language features can continue to
work as users expect them to when extended to inline classes, with little in the
way of special treatment for primitives.

It may appear that we have pulled a rhetorical trick, just renaming "boxing" to
"inline widening".  But the difference is: now we have a new translation target,
one which can routinely optimize layout, instantiation, and calling conventions
for inline classes and their reference projections.  We've just grafted familiar
language semantics onto that efficient translation target, preserving the
illusion that nothing has changed (except performance.)

## Primitives

As discussed in [Language Model](02-object-model.html), a key goal of this story
is to migrate primitives so that they are "mostly just" inline classes, while
retaining the performance characteristics of primitives.  The current divide
between _primitive and reference types_ migrates to being a divide between
_inline and identity classes_, where the rules in each domain (and the
relationship between them) are largely the same as they are on the corresponding
side of the divide today.  This migration will likely play out in stages.

The first step is for the language to treat the primitives as if they are the
value projection of some implicitly declared inline class (e.g.,
`java.lang.int`), and their wrapper classes as if they are the reference
projection of that inline class.  (Then `int.ref` becomes an alias for
`Integer`, and `Integer.val` becomes an alias for `int`.)  This allows the rules
for type inference, overload selection, type bound conformance, autoboxing, etc,
to treat primitives and inline uniformly, in a way that is entirely consistent
with how primitives are treated today.

Later on, it may be possible that `int` can actually be declared as a (special)
class in `java.lang`, with interfaces and methods, and finally, when the
generics story is complete, `int` can implement `Comparable<int>`.

The JVM will, of course, continue to have direct support for primitives (e.g.,
the `I` carrier, and the `i*` instructions).  The language compiler will have
substantial latitude to use the `I` carrier and instructions in generated code
where beneficial and where required for migration compatibility (such as in
field and method descriptors), and use `Qjava/lang/int;` elsewhere.

This scheme involves one behaviorally incompatible change: synchronization on an
instance of one of the primitive wrapper classes will throw `IMSE`.  While we do
not take this lightly, it is almost universally an error to do this, and in the
very few cases where this is not an outright error, simple workarounds exist.

## Reflection

In Q-World, reference projections were derived by the JVM mechanically from
inline class files, so the JVM also had to create a synthetic mirror for
the reference projection.  Now that the value and reference projections are
real classfiles, we can use real class mirrors to do most of the reflective
work.  

We may wish to extend the reflection API to provide more information about
inline classes (is this class an inline class, what is its reference projection,
etc).  We may also find that there are cases where we want to tweak the
reflective behavior of one or both of the mirrors to behave more like their
language counterpart -- or alternately, we may not, and simply let reflection
expose the translation model.  
