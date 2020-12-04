# Valhalla: Unified Generics

In the first part, we outlined a story for unifying primitives and classes,
while retaining (and expanding) the memory efficiency benefits of primitives.
This, on its own, is an enormous step; it reduces the gap between primitive and
"regular" class types, it gives us a single single top type for objects
(`Object`) and arrays (`Object[]`), and allows us to write new classes with
primitive-like runtime behavior and performance.

## Generics over primitive classes

But, there is still another gap to be addressed -- that generics cannot have
primitive types as their type parameters.  This is a problem even just with the
eight built-in primitives, and, when the domain of primitive types grows, will
become an even bigger problem.  

Because each primitive class has a reference projection (`P.ref`), which can be
thought of as a "better box", we could declare the problem solved by saying that
generics work over reference types, and "just" using `Foo<P.ref>`.  But this is
unsatisfying in several ways.

The most obvious way that this is unsatisfying is that it is ugly; using the
`.ref` projection explicitly is supposed to be an escape hatch for rare
situations such as actually not wanting flattening, requiring structural
circularity or nullability, or compatibly migrating existing value-based classes
to primitive classes.  But instantiating an `ArrayList` over some primitive
class is not going to be rare.  (If this were the only concern, we might seek a
syntactic fix (treating `Foo<P>` as if it were `Foo<P.ref>`), but that merely
moves the problem elsewhere.)

A more significant way that this is unsatisfying is that it puts users to a
choice that Valhalla was supposed to help them avoid: choosing between
abstraction and performance.  Primitive objects have attractive density and
flattening behavior that makes them more efficient than identity objects, but if
we are going to generify over their _references_, we are giving back the
hard-won performance benefits of eliminating indirections.  The long-term
solution for this to make generics _specializable_, so that an `ArrayList<int>`
is actually backed by a flattened `int[]`, getting the benefits of abstraction
and reuse while retaining the flatness and density benefits of primitive
classes.  This is, of course, a significant undertaking, and not one that we
will be able to deliver at the same time as primitive classes (unless we want to
delay delivering primitive classes until then.)  While it would be nice to have
it all, and now, erased-to-reference generics over a unified type system is
still a significant improvement over the status quo.

If specialized generics are on the horizon, we want to avoid creating the future
migration compatibility problem where today's erased-to-reference generics over
primitive classes want to become tomorrow's specialized generics.  And this is
the final, and perhaps most important, reason to not simply say "just use
`P.ref`" for erased generics today: it sows the seeds of tomorrow's
compatibility problem.  We want the generic code people write today to be
compatible with tomorrow's specialized generics, otherwise people will face
significant migration and interoperability problems in the future.

#### Charting our path

Viewed in this light, we can see that we have two interconnected goals with
regard to generics: an expressiveness goal that we want to deliver in the same
time frame as primitive classes (letting us express generics over all the types)
and a performance goal we want to solve eventually (specializing generic classes
over primitive type variables so we can get flattening all the way down.)

We can draw some inspiration from the approach we've taken for primitive classes
(and countless design decisions previously).  Declaring a class as a primitive
class does not necessarily mean that the runtime must always represent it
directly; it means that the semantics of a primitive class are _sufficiently
constrained_ so that we can have confidence that the runtime can _routinely_
flatten it where it deems it beneficial.  We leave the actual choice of
representation to the judgment of the runtime -- for example, if a primitive
class had 1000 fields, it would be quite sensible for the runtime to choose an
indirect representation.  (One could say that the reason Valhalla is needed _at
all_ is that the semantics of identity classes are not sufficiently constrained
for the runtime to routinely figure out when it can flatten or elide allocation;
it needs a little help from the programmer.)

The flip side of this approach -- constrain the semantics so that optimization
is possible, rather than raising direct control over the optimization the
programming model -- is that the representation choices are observable only
indirectly (through performance observations.)  This plays to the strength of
Java's approach, which is to leave optimization decisions to the JVM based on
what degrees of freedom the program can be observed to have given up.

We select this same approach with respect to generic specialization; rather than
allowing the programmer to _demand_ that a particular instantiation be
specialized -- which could create two different kinds of instantiations of
`Foo<X>` -- we relax the semantics of type variables so that they can take on
any type, including primitive value types (and `void`.)  But, as we did with
primitive classes, we leave the choice of whether to specialize or not to the
runtime.  Initially, the runtime will not specialize at all, but later the
runtime will have the option to specialize suitable generic instantiations
without the code explicitly opting in.  As with primitive classes, the semantics
are constrained so that the only observable difference is performance.

#### Universal type variables

Currently, a generic type variable can only be instantiated with a reference
type; we propose to replace these with _universal type variables_ that can range
over all types (and `void`.)  The benefit here is significant: just as we
finally unify primitives with objects, we can also unify the generic type system
to engage uniformly with all Java's types.

The design of primitive classes was influenced, in no small part, by the desire
to be able to include primitive classes in erased generics; that primitives
extend `Object`, that their arrays extend `Object[]`, and that they can be
operated on by the `a*` bytecodes,  are all a part of this story.  Erased
generics offer us a strong safety promise:

> If a program compiles with no unchecked or raw warnings, the synthetic casts
introduced by the compiler are guaranteed to succeed.

When a variable of type `T` stores a value that is not consistent with its
instantiation, this is called _heap pollution_.  The above safety guarantee
tells us that if we don't have any generics warnings, there will be no heap
pollution.

If we allow generics to be instantiated with primitive value types, a new form
of heap pollution becomes possible: that a `T`-valued variable instantiated with
a primitive value type will hold `null`.  We could call this _null pollution_,
but it is a special case of heap pollution.  In order for our safety guarantee
to hold, we have to ensure that nulls do not creep unexpectedly into `T`-valued
variables, or at least that we issues suitable warnings (analogous to raw type
or unchecked conversion warnings) when this might happen.

Possible sources of null pollution are:

 - Explicitly assigning `null` to a `T`-valued variable.
 - Uninitialized `T`-valued fields or array elements, since `null` is the always
   the default value for the erasure of `T`.
 - Capture conversion.  If we have a `List<? extends Animal>`, we can't put
   anything into this list, because we don't know whether it is a `List<Dog>` or
   `List<Cat>` -- except `null`.  Since `null` is in the intersection of all the
   possible instantiations, we can assign `null` to capture variables.
 - Implicit conversions.  If we have a method that returns `T`, and we return a
   `T.ref`, this is subject to an implicit conversion, but if the `T.ref` holds
   `null`, then this implicit conversion might fail with an NPE.

If we want to ensure this safety guarantee, we must restrict or at least issue
generics warnings for all the possible sources of null pollution.  And, if we
want to make "universal type variables" the default, this may result in new
warnings for existing code (much as introducing generics resulted in new
warnings, such as raw type warnings.)

#### A (soft) migration challenge

When we get one of these new warnings that tell us that a `T`-valued variable
holds null, we have to figure out how to remediate that.  One option is to
promote it to `T.ref`.  In many cases, this is enough, because there are other
means to ensure that the nulls do not escape; for example, the internal backing
array (which never escapes) inside an `ArrayList` can be typed as `T.ref[]`.  

In other cases, this may moves the problem elsewhere, to where callers and
subclasses might have to deal with it.  For example, consider the method
`Map::get`, which signals failure by returning null.  In this case, we would
want to adjust the return type to reflect this:

```
interface Map<K,V> {
    V.ref get(K k);
}
```

and clients would additionally need to be prepared to deal with the `null` when
dealing with maps whose values are primitives -- as they are today.  

While developers will not thank us for giving them new warnings, the alternative
is probably worse -- requiring generic classes to opt into instantiation with
primitives.  This would likely set back the timeframe where clients could use
generics with primitives by at least five years, since the opt-in feature would
only be available in new code, and most libraries want to compile to an older
version (such as Java 8) to maximize their audience.  (It took five to ten years
before libraries embraced generics, partially because they did not want to take
on a Java 5 dependency until well into the lifetime of Java 7.)

The opposite possibility -- where a class can opt out of primitive instantiation
-- is still a possibility.  Whether we support such an option or not will be
based on an assessment of the worst-case migration cost of moving to universal
type variables.

To make decisions here, we need to gather data on the cost of migrating
real-world classes to be primitive-safe, and the intrusiveness of `T.ref` types
on their APIs.

#### Type inference and conformance

For generic type bounds such as `Foo<T extends Bar>` where `Bar` is an interface
or suitable abstract class, we can treat any primitive class which extends `Bar`
(directly or indirectly) as conforming to the bound.

Currently, type inference is fairly aggressive about promoting primitive types
to their box types early in the constraint-gathering process; the analog of this
in the new world is promoting primitive types `P` to `P.ref` similarly early.
If left in place, this would cause `.ref` types to pop out of inference quite
frequently, undermining the goal that the `.ref` types are intended to be
infrequent and only needed in cases where a reference is actually needed.
