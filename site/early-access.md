# Early Access Release

The latest early access release of Valhalla features, codenamed *LW4*, will be
published at [jdk.java.net](https://jdk.java.net/valhalla/).

Functionality, language syntax, and `class` file formats are subject to change.
Many supplementary tools and APIs do not fully support the new features.
The goal of this release is to solicit feedback and establish a milestone as we
work toward a preview release in Java SE.

## LW4 Functionality

This release is primarily focused on implementing the
[Value Objects JEP](https://openjdk.org/jeps/8277163).
(See the [specification changes](http://cr.openjdk.java.net/~dlsmith/jep8277163/)
for more precise details.)

Java programs can declare concrete `value` classes, and can also use the
`identity` and `value` keywords in other class and interface declarations.
Various constraints are enforced on the declaration and use of classes with
these modifiers, as described by the JEP.
At run time, value objects have the specified identity-free behavior (can be
equal to separately-created instances, will throw on synchronization attempts),
and typically have inline, allocation-free encodings when stored in local
variables or passed between methods.

Interested users are encouraged to explore the performance and migration impact
of value objects on their applications, and to provide feedback at
`valhalla-dev@openjdk.java.net`.

Support for flattened fields and arrays is still experimental, and must be
activated using the `javac` flag `-XDenablePrimitiveClasses` and the `java` flag
`-XX:+EnablePrimitiveClasses`.
This unlocks features that generally align with
[JEP 401](https://openjdk.org/jeps/401) (as of November 2022):
classes may be declared `primitive`, uses of the primitive class name refer to a
*primitive value type*, and `.ref` can be appended to a primitive class name to
get its corresponding reference type.
Primitive value types are null-free; arrays of these types can be allocated and
treated as instances of supertype arrays (such as `Object[]`).
At run time, primitive-typed fields and arrays typically store their primitive
values' fields directly, and may be non-atomically modified.

Notes:

-   While these features are destined to be preview features, they currently
    work out of the box, without the `--enable-preview` flag.

-   Only the x64 Linux, x64 macOS, and x64 Windows platforms are currently
    supported.

-   The JVM's existing escape analysis and code inlining optimizations are quite
    effective at eliminating the footprint of locally-contained identity
    objects. Performance improvements for value objects are achieved in cases
    where these optimizations would fail.

-   Refactoring an `identity` class to be a `value` class will have various
    compatibility implications, as
    [outlined](https://openjdk.org/jeps/8277163#Migration-of-existing-classes)
    in the JEP.

-   `jshell` works with value classes but does not support the primitive class
    flags.

-   `javap` recognizes all the new class file features.

-   ...
