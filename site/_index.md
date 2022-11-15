# Project Valhalla

Project Valhalla is augmenting the Java object model with *value objects* and
*user-defined primitives*, combining the abstractions of object-oriented
programming with the performance characteristics of simple primitives.
These features will be complemented with changes to Java's generics to preserve
performance gains through generic APIs.

This Project is sponsored by the
[HotSpot Group](http://openjdk.java.net/groups/hotspot).

## JEPs

Valhalla project features will be added to Java over multiple releases.
This process is [managed with JEPs](https://openjdk.java.net/jeps/1),
each of which facilitates the development and integration into the JDK of a
cohesive set of changes.

There are four distinct feature sets under development:

1.  [Value Objects](https://openjdk.org/jeps/8277163), introducing class
    instances that lack identity

2.  [User-defined Primitives](https://openjdk.org/jeps/401), supporting
    null-free flattened storage

3.  [Classes for the Basic Primitives](https://openjdk.org/jeps/402), bringing
    value class features to the existing primitives

4.  [Enhanced Generics](https://openjdk.org/jeps/8261529), providing optimal
    performance for value classes used with generic APIs

We've also worked on some supplementary tasks and features, including:

-   [JEP 181: Nest-Based Access Control](https://openjdk.java.net/jeps/181)
    (delivered in 11)

-   [JEP 309: Dynamic Class-File Constants](https://openjdk.java.net/jeps/309)
    (delivered in 11)

-   [JEP 334: JVM Constants API](https://openjdk.org/jeps/334)
    (delivered in 12)

-   [JEP 371: Hidden Classes](https://openjdk.java.net/jeps/371)
    (delivered in 15)

-   [JEP 390: Warnings for Value-Based Classes](https://openjdk.java.net/jeps/390)
    (delivered in 16)

-   [Better-defined JVM class file validation](https://openjdk.java.net/jeps/8267650)
    (draft)

## Background Documents

These documents present a more holistic view of the Valhalla project's goals and
design considerations.

-   The State of Valhalla (December 2021)
    - [1. The Road to Valhalla](design-notes/state-of-valhalla/01-background)
    - [2. The Language Model](design-notes/state-of-valhalla/02-object-model)
    - [3. The JVM Model](design-notes/state-of-valhalla/03-vm-model)

-   Enhanced generics
    - [Background: How We Got the Generics We Have](design-notes/in-defense-of-erasure) (June 2020)
    - [The Saga of the Parametric VM](design-notes/parametric-vm/parametric-vm) (April 2021)

## Implementation

Development takes place in a public OpenJDK repository, with occasional
early-access builds being published.

Interested developers are encouraged to experiment with these early releases.

-   [Early Access Documentation](early-access) detailing the latest release
-   [Early Access Builds](https://jdk.java.net/valhalla/)
-   [GitHub repository](https://github.com/openjdk/valhalla)

Branches in the repository include `lworld` (the main line of Valhalla
development), `master` (tracking mainline OpenJDK), `jep*` (staging areas for
the production-ready implementations of JEPs), and various others prototyping
proposed features.

## Community

-   [Members](http://openjdk.java.net/census#valhalla)

-   Mailing lists

    -   [valhalla-dev](http://mail.openjdk.java.net/mailman/listinfo/valhalla-dev),
        for technical discussion related to Project Valhalla
        ([archives](http://mail.openjdk.java.net/pipermail/valhalla-dev/))

    -   [valhalla-spec-experts](http://mail.openjdk.java.net/mailman/listinfo/valhalla-spec-experts),
        for moderated design discussion among expert group members only
        ([archives](http://mail.openjdk.java.net/pipermail/valhalla-spec-experts/))

    -   [valhalla-spec-observers](http://mail.openjdk.java.net/mailman/listinfo/valhalla-spec-observers),
        for those who wish to monitor discussions in the *valhalla-spec-experts*
        list; public replies are allowed, but not forwarded to the experts list
        ([archives](http://mail.openjdk.java.net/pipermail/valhalla-spec-observers/))

    -   [valhalla-spec-comments](http://mail.openjdk.java.net/mailman/listinfo/valhalla-spec-comments),
        for sending specification-related comments, suggestions, and other
        feedback to the expert group
        ([archives](http://mail.openjdk.java.net/pipermail/valhalla-spec-comments/))

-   [Documentation repository](https://github.com/openjdk/valhalla-docs)
    (allows updating this page!)

We welcome input from interested Java developers. Keep in mind that most
theoretical ideas have been well explored over the last few years! The greatest
help can be provided by those who try out concrete prototypes and can share
their experiences with real-world code bases.

## Legacy Links

See the [legacy page](legacy) for links to earlier proposed JEPs, design
documents, presentations, and prototypes.
