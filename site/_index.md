# Project Valhalla

Project Valhalla is augmenting the Java object model with *value objects*,
combining the abstractions of object-oriented programming with the performance
characteristics of simple primitives. Supplementary changes to Java's generics
will carry these performance gains into generic APIs.

This Project is sponsored by the
[HotSpot Group](http://openjdk.org/groups/hotspot).

## JEPs

Valhalla project features will be added to Java over multiple releases.
This process is [managed with JEPs](https://openjdk.org/jeps/1),
each of which facilitates the development and integration into the JDK of a
cohesive set of changes.

There are five distinct feature sets under development:

1.  *[Value Classes and Objects](https://openjdk.org/jeps/401)*, introducing
    objects that lack identity and thus can have optimized encodings

2.  *Null-Restricted and Nullable Types*, providing language support for
    null-aware types and runtime enforcement of null restrictions

3.  *[Null-Restricted Value Class Types](https://openjdk.org/jeps/8316779)*,
    improving the performance of fields and arrays with null-restricted
    value class types

4.  *[Enhanced Primitive Boxing](https://openjdk.org/jeps/402)*, allowing
    primitives to be treated more like objects

5.  *Parametric JVM*, preserving and optimizing generic class and method
    parameterizations at runtime

We've also worked on some supplementary tasks and features, including:

-   [JEP 181: Nest-Based Access Control](https://openjdk.org/jeps/181)
    (delivered in 11)

-   [JEP 309: Dynamic Class-File Constants](https://openjdk.org/jeps/309)
    (delivered in 11)

-   [JEP 334: JVM Constants API](https://openjdk.org/jeps/334)
    (delivered in 12)

-   [JEP 371: Hidden Classes](https://openjdk.org/jeps/371)
    (delivered in 15)

-   [JEP 390: Warnings for Value-Based Classes](https://openjdk.org/jeps/390)
    (delivered in 16)

## Background Documents & Presentations

These documents and presentations present a more holistic view of the Valhalla
project's goals and design considerations.

-   [Brian Goetz: Valhalla - Where Are We?](https://www.youtube.com/watch?v=IF9l8fYfSnI) (JVM Language Summit, July 2024)

-   [Dan Smith: Value Objects in Valhalla](https://www.youtube.com/watch?v=a3VRwz4zbdw) (JVM Language Summit, August 2023)

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

-   [Early Access Documentation](early-access)
-   [Early Access Builds](https://jdk.java.net/valhalla/)
-   [GitHub repository](https://github.com/openjdk/valhalla)

Branches in the repository include `lworld` (the main line of Valhalla
development), `master` (tracking mainline OpenJDK), and various others
prototyping proposed features.

## Community

-   [Members](http://openjdk.org/census#valhalla)

-   Mailing lists

    -   [valhalla-dev](http://mail.openjdk.org/mailman/listinfo/valhalla-dev),
        for technical discussion related to Project Valhalla
        ([archives](http://mail.openjdk.org/pipermail/valhalla-dev/))

    -   [valhalla-spec-experts](http://mail.openjdk.org/mailman/listinfo/valhalla-spec-experts),
        for moderated design discussion among expert group members only
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
help can be provided by those who try out concrete prototypes and can share
their experiences with real-world code bases.

## Legacy Links

See the [legacy page](legacy) for links to earlier proposed JEPs, design
documents, presentations, and prototypes.
