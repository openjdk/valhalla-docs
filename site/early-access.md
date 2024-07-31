# Early Access Release

The latest early access release of Valhalla features is published at
**[https://jdk.java.net/valhalla/](https://jdk.java.net/valhalla/)**.

Functionality, language syntax, and `class` file formats are subject to change.
Some supplementary tools and APIs do not fully support the new features.
The goal of this release is to solicit feedback and establish a milestone as we
work toward a preview release in Java SE.

**This milestone is focused on specification conformance, and excludes
some important performance optimizations.** It is not intended to be
used for performance benchmarking.


## Value Classes and Objects

The features implemented by this release are described in
[JEP 401: Value Classes and Objects (Preview)](https://openjdk.org/jeps/401).
(See the [specification changes](http://cr.openjdk.java.net/~dlsmith/jep401/latest)
for more precise details.)

When [preview features](https://openjdk.org/jeps/12) are turned on,
Java programs can declare `value` classes and records, which have only `final`
fields and lack object identity. At run time, the `==` operator compares
value class instances according to their field values, without regard
to when or where they were created. Other identity-sensitive operations
are adjusted to appropriately to work with objects that lack identity.

By enabling preview features, developers also get value-class versions of
`java.lang.Integer`, `java.time.LocalDate`, `java.util.Optional`, and a
handful of other classes in the standard libraries. Developers are encouraged
to test their code against these modified libraries, and to experiment with their
own value-candidate classes.

Please provide any feedback to
<span style="white-space:nowrap;">`valhalla-dev@openjdk.java.net`</span>.


## Notes

-   This release lacks some important performance optimizations, and
    should not be used for performance benchmarking.

-   The JEP includes some helpful guidance on the
    [migration of existing classes](https://openjdk.org/jeps/401#Migration-of-existing-classes).

-   Deserialization of a value class is only supported by the `ObjectInputStream`
    API if the class is a record class or is instantiated via `readResolve`.
    Improved support for deserialization will be coming later.
