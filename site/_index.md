# Project Valhalla

Project Valhalla is augmenting the Java object model with *value objects*,
combining the abstractions of object-oriented programming with the performance
characteristics of simple primitives.

This Project is sponsored by the
[HotSpot Group](http://openjdk.org/groups/hotspot).

## What's New?

**August 2026:**
: [JEP 401: Value Objects (Preview)][401] and
    [JEP 539: Strict Field Initialization in the JVM (Preview)][539]
    are now integrated and will be included in JDK 28!
    Try out value objects today with an [early-access JDK 28](https://jdk.java.net/28/)
    build.

    For background, you can [read the JEP][401] or review this
    [short introduction](https://inside.java/2025/10/27/try-jep-401-value-classes/).
    For some broader context, see this JavaOne talk about programming with
    [immutable data in Java](https://www.youtube.com/watch?v=BdLND9D81lI).

## New Features

The anticipated Valhalla language and performance features will not be delivered
all at once, but through a steady stream of enhancements across multiple JDK
releases.

There are five distinct feature sets under development:

1.  *[Value Objects](value-objects)*, introducing objects that lack
    identity and thus can have optimized encodings

2.  *Null-Restricted Storage* of value objects to improve memory layouts

3.  *Array Enhancements* to support properties like immutability and safe
    initialization, and to facilitate interop between primitive and reference
    arrays

4.  *Unifying Primitives and Classes* by improving the use of boxing for method
    invocations and generics, and by expanding features like conversions and
    operators to work with value classes

5.  *Parametric JVM*, specializing and optimizing generic class and method
    parameterizations at run time

## Background Documents & Presentations

These documents and presentations provide a more holistic view of the Valhalla
project's goals and design considerations.

-   [Dan Smith: Better Tools for Immutable Data](https://www.youtube.com/watch?v=BdLND9D81lI)
    (JavaOne 2026)

-   [Frederic Parain: Value Classes & Heap Flattening](https://www.youtube.com/watch?v=NF4CpL_EWFI)
    (JVM Language Summit 2025)

-   [Brian Goetz: Growing the Java Language](https://www.youtube.com/watch?v=Gz7Or9C0TpM)
    (JVM Language Summit 2025)

-   [Dan Smith: A New Model for Java Object Initialization](https://www.youtube.com/watch?v=XtvR4kqK8lo)
    (JavaOne 2025)

-   [Brian Goetz: Valhalla—Where Are We?](https://www.youtube.com/watch?v=IF9l8fYfSnI)
    (JVM Language Summit 2024)

-   [Dan Smith: Value Objects in Valhalla](https://www.youtube.com/watch?v=a3VRwz4zbdw)
    (JVM Language Summit 2023)

-   The State of Valhalla (December 2021)
    - [1. The Road to Valhalla](design-notes/state-of-valhalla/01-background)
    - [2. The Language Model](design-notes/state-of-valhalla/02-object-model)
    - [3. The JVM Model](design-notes/state-of-valhalla/03-vm-model)

-   Parametric JVM
    - [Background: How We Got the Generics We Have](design-notes/in-defense-of-erasure) (June 2020)
    - [The Saga of the Parametric VM](design-notes/parametric-vm/parametric-vm) (April 2021)

## Implementation

[JEP 401: Value Objects (Preview)][401] and
[JEP 539: Strict Field Initialization in the JVM (Preview)][539]
are integrated into the JDK will be included in JDK 28.

Interested developers are encouraged to experiment with early-access builds at
<https://jdk.java.net/28/>.

Other features are prototyped in various branches of the
[Valhalla GitHub repository](https://github.com/openjdk/valhalla).

## Community

-   [Members](http://openjdk.org/census#valhalla)

-   Mailing lists

    -   [valhalla-dev](http://mail.openjdk.org/mailman/listinfo/valhalla-dev),
        for technical discussion and user experiences related to Project Valhalla
        ([archives](http://mail.openjdk.org/pipermail/valhalla-dev/))

    -   [valhalla-spec-experts](http://mail.openjdk.org/mailman/listinfo/valhalla-spec-experts),
        for moderated design discussion among invited experts
        ([archives](http://mail.openjdk.org/pipermail/valhalla-spec-experts/))

    -   [valhalla-spec-observers](http://mail.openjdk.org/mailman/listinfo/valhalla-spec-observers),
        for those who wish to monitor discussions in the *valhalla-spec-experts*
        list; public replies are allowed, but not forwarded to the experts list
        ([archives](http://mail.openjdk.org/pipermail/valhalla-spec-observers/))

    -   [valhalla-spec-comments](http://mail.openjdk.org/mailman/listinfo/valhalla-spec-comments),
        for sending specification-related comments, suggestions, and other
        feedback to the expert group
        ([archives](http://mail.openjdk.org/pipermail/valhalla-spec-comments/))

-   [Documentation repository](https://github.com/openjdk/valhalla-docs)
    (allows updating this page!)

We welcome input from interested Java developers. Keep in mind that most
theoretical ideas have been well explored over the last few years! The greatest
help can be provided by those who try out published implementations and can
share their experiences with real-world code bases.

## Project JEPs

Each enhancement to the Java platform is described and tracked with a
[JDK Enhancement Proposal (JEP)](https://openjdk.org/jeps/1). The following
JEPs are under active development or have been delivered.

--------------------------------------------------------------------
JEP                            Feature              Status
-----------------------------  -------------------  ----------------
[JEP 401: Value Objects        Value Objects        Integrated
(Preview)][401]

[JEP 539: Strict Field         Supplementary        Integrated
Initialization in the JVM
(Preview)][539]

[Null-Restricted Value Class   Null-Restricted      Draft
Types (Preview)][8316779]      Storage

[JEP 402: Enhanced Primitive   Unifying Primitives  Draft
Boxing (Preview)][402]

[JEP 390: Warnings for         Value Objects        Delivered in 16
Value-Based Classes][390]

[JEP 371: Hidden               Supplementary        Delivered in 15
Classes][371]

[JEP 334: JVM Constants        Supplementary        Delivered in 12
API][334]

[JEP 309: Dynamic Class-File   Supplementary        Delivered in 11
Constants][309]

[JEP 181: Nest-Based Access    Supplementary        Delivered in 11
Control][181]
--------------------------------------------------------------------


[181]: https://openjdk.org/jeps/181
[218]: https://openjdk.org/jeps/218
[303]: https://openjdk.org/jeps/303
[309]: https://openjdk.org/jeps/309
[334]: https://openjdk.org/jeps/334
[371]: https://openjdk.org/jeps/371
[390]: https://openjdk.org/jeps/390
[401]: https://openjdk.org/jeps/401
[402]: https://openjdk.org/jeps/402
[539]: https://openjdk.org/jeps/539
[8316779]: https://openjdk.org/jeps/8316779


## Legacy Links

See the [legacy page](legacy) for links to earlier proposed JEPs, design
documents, presentations, and prototypes.