# Project Valhalla

Project Valhalla is augmenting the Java object model with *value objects*,
combining the abstractions of object-oriented programming with the performance
characteristics of simple primitives.

This Project is sponsored by the
[HotSpot Group](http://openjdk.org/groups/hotspot).

## What's New?

**October 2025:** Check out the latest
[early-access build](https://jdk.java.net/valhalla/)
implementing [value classes and objects](value-objects)!
We encourage Java developers to try it out on their real-world applications and
report their experiences.

To get started, you can review this
[short introduction](https://inside.java/2025/10/27/try-jep-401-value-classes/)
at *inside.java*.
Learn more with a
[presentation by Frederic Parain](https://www.youtube.com/watch?v=NF4CpL_EWFI)
about the JVM optimizations applied to value objects.

## New Features

The anticipated Valhalla language and performance features will not be delivered
all at once, but through a steady stream of enhancements across multiple JDK
releases.

There are five distinct feature sets under development:

1.  *[Value Classes and Objects](value-objects)*, introducing objects that lack
    identity and thus can have optimized encodings

2.  *Null Checking* at both compile time and run time to manage the flow of
    null references through programs

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

Development takes place in a public OpenJDK repository, with occasional
early-access builds being published.

Interested developers are encouraged to experiment with these early releases.

-   [Early-Access Builds](https://jdk.java.net/valhalla/)
-   [GitHub repository](https://github.com/openjdk/valhalla)

Branches in the repository include `lworld` (the main line of Valhalla
development), `master` (tracking main-line OpenJDK), and various others
prototyping proposed features.

## Community

-   [Members](http://openjdk.org/census#valhalla)

-   Mailing lists

    -   [valhalla-dev](http://mail.openjdk.org/mailman/listinfo/valhalla-dev),
        for technical discussion related to Project Valhalla
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
[JEP 401: Value Classes and    Value Classes        Submitted
Objects (Preview)][390]

[Strict Field Initialization   Supplementary        Draft (to be
in the JVM][8350458]                                submitted)

[Null-Restricted and Nullable  Null Checking        Draft
Types (Preview)][8303099]

[Null-Restricted Value Class   Null Checking        Draft
Types (Preview)][8316779]

[JEP 402: Enhanced Primitive   Unifying Primitives  Draft
Boxing (Preview)][402]

[Warnings for Value-Based      Value Classes        Delivered in 16
Classes][390]

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
[8303099]: https://openjdk.org/jeps/8303099
[8316779]: https://openjdk.org/jeps/8316779
[8350458]: https://openjdk.org/jeps/8350458


## Legacy Links

See the [legacy page](legacy) for links to earlier proposed JEPs, design
documents, presentations, and prototypes.