# Project Valhalla

Project Valhalla plans to augment the Java object model with *value objects* and
*user-defined primitives*, combining the abstractions of object-oriented
programming with the performance characteristics of simple primitives.
These features will be complemented with changes to Java's generics to preserve
performance gains through generic APIs.

This Project is sponsored by the
[HotSpot Group](http://openjdk.java.net/groups/hotspot).

## JEPs

Valhalla project features will be added to Java over multiple releases.
This process is managed with JEPs, each of which facilitates the development and
integration into the JDK of a cohesive set of changes.

Preparatory changes:

-   [JEP 181: Nest-Based Access Control](https://openjdk.java.net/jeps/181)
    introduces a JVM mechanism for private members to be shared between
    different classes in the same package; `javac` uses this to simplify
    some interactions between nested and enclosing classes (delivered in 11)

-   [JEP 309: Dynamic Class-File Constants](https://openjdk.java.net/jeps/309)
    introduces the `CONSTANT_Dynamic` form of constant pool entry, enabling JVM
    constants to be computed dynamically by bootstrap code at resolution time
    (delivered in 11)

-   [JEP 371: Hidden Classes](https://openjdk.java.net/jeps/371)
    provides an API for the runtime creation of *hidden classes*, special
    classes that cannot be named directly but can implement supertypes or be
    accessed via reflection (delivered in 15)

-   [JEP 390: Warnings for Value-Based Classes](https://openjdk.java.net/jeps/390)
    implements compile-time and runtime warnings to discourage misuse of some
    future value classes in the JDK, including the primitive wrapper classes
    (delivered in 16)

-   [Better-defined JVM class file validation](https://openjdk.java.net/jeps/8267650)
    will update the JVM specification to clarify various aspects of class file
    validation, in anticipation of new Valhalla features (draft)

Value objects:

-   [Value Objects (Preview)](https://openjdk.java.net/jeps/8277163)
    will introduce *value classes* with *value object* instances, a special kind
    of object that has only `final` instance fields and lacks identity, thus
    enabling certain runtime optimizations (draft)

-   [JEP 401: Primitive Classes (Preview)](https://openjdk.java.net/jeps/401)
    will build on the value classes feature with *primitive classes*, enabling
    the declaration of user-defined primitive types with performance comparable
    to the basic primitives (candidate)

-   [JEP 402: Classes for the Basic Primitives (Preview)] will repurpose the
    primitive wrapper classes to act as declarations for the basic primitive
    types, unifying the treatment of these types with that of other types
    declared by primitive classes (candidate)

Enhanced generics:

-   [Universal Generics (Preview)](https://openjdk.java.net/jeps/8261529)
    will allow primitive types to be used directly as type arguments without the
    need to convert to reference types; new warnings will maintain the safety
    guarantees of generic code (draft)

-   Parametric JVM
    will support *specialization* of generic classes and methods for particular
    type argument layouts, improving the performance of generic code when used
    with value class types and primitive types (no draft yet)

## Background Documents

These documents present a more holistic view of the Valhalla project's goals and
design considerations.

The State of Valhalla (December 2021):

- [1. The Road to Valhalla](design-notes/state-of-valhalla/01-background)
- [2. The Language Model](design-notes/state-of-valhalla/02-object-model)
- [3. The JVM Model](design-notes/state-of-valhalla/03-vm-model)

Enhanced generics:

- [Background: How We Got the Generics We Have](design-notes/in-defense-of-erasure) (June 2020)
- [The Saga of the Parametric VM](design-notes/parametric-vm/parametric-vm) (April 2021)

## Implementation

Prototyping for the project takes place in a public OpenJDK repository, with
occasional early-access builds being published.

Interested developers are encouraged to experiment with these prototypes.

-   [Early Access Builds](https://jdk.java.net/valhalla/)

-   [GitHub repository](https://github.com/openjdk/valhalla), including the
    following branches:
    
    - `lworld`, the main line of Valhalla development
    - `master`, tracking mainline OpenJDK
    - `jep*`, staging area for the production-ready implementation of a JEP
    - various other prototypes of proposed features

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

We welcome input from interested Java developers. Keep in mind that most
theoretical ideas have been well explored over the last few years! The greatest
help can be provided by those who try out concrete prototypes and can share
their experiences with real-world code bases.

## Legacy Links

See the [legacy page](legacy) for links to earlier proposed JEPs, design
documents, and presentations.
