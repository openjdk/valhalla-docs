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
of value objects on their applications, and to provide feedback to
<span style="white-space:nowrap;">`valhalla-dev@openjdk.java.net`</span>.

Support for flattened fields and arrays is still experimental, and must be
activated using the command-line options documented below.
The unlocked behavior is generally consistent with
[JEP 401](https://openjdk.org/jeps/401) (as of November 2022):
Classes may be declared `primitive`, uses of the primitive class name refer to a
*primitive value type*, and `.ref` can be appended to a primitive class name to
get its corresponding reference type.
Primitive value types are null-free; covariant arrays of these types can be
created (e.g., a `Point[]` can be treated as an `Object[]`).
At run time, primitive-typed fields and arrays typically store their primitive
values' fields directly; reads and writes are not guaranteed to be atomic.



## Command-Line Options

Special options supported by `javac`:

-   `-XDenablePrimitiveClasses` to unlock experimental support for Primitive
    Classes

Special options supported by `java`:

-   `-XX:-EnableValhalla` to deactivate support for Value Objects and revert to
    standard JVM behavior

-   `-XX:+EnablePrimitiveClasses` to unlock experimental support for Primitive
    Classes

-   `-XX:InlineFieldMaxFlatSize=n`, when primitive class support is activated,
    set the max number of bytes, `n`, stored in a flattened field (negative
    means unbounded; default is 128)

-   `-XX:FlatArrayElementMaxSize=n`, when primitive class support is activated,
    set the max number of bytes, `n`, stored in a flattened array component
    (negative means unbounded; default is unbounded)

-   `-XX:FlatArrayElementMaxOops=n`, when primitive class support is activated,
    set the max number of references, `n`, permitted in a single flattened array
    component (negative means unbounded; default is 4)



## Performance Guidance

As
[described by the JEP](https://openjdk.org/jeps/8277163#HotSpot-implementation),
HotSpot optimizes the stack encodings of value objects within C2-generated code,
avoiding heap allocation.
Most performance benefits will be observed in code that would traditionally
require allocation of a large number heap objects, to be referenced by local
variables and passed between methods.

Normal value objects will *not* be "flattened" into field and array storage.
That behavior is part of the experimental Primitive Classes feature.

Note that HotSpot's existing escape analysis and code inlining optimizations are
quite effective at eliminating heap allocations for locally-contained identity
objects. Performance improvements for value objects will primarily be observed
in cases where these identity object optimizations would fail.



## Notes

-   While these features are destined to be preview features, they currently
    work out of the box, without the
    <span style="white-space:nowrap;">`--enable-preview`</span> flag.

-   Refactoring an `identity` class to be a `value` class will have various
    compatibility implications, as
    [outlined in the JEP](https://openjdk.org/jeps/8277163#Migration-of-existing-classes).

-   A value class may be declared as a record (`value record Foo(int x, int y)`)
    and may be serializable (`value class Bar implements Serializable`).
    These features are fully functional.

-   Methods to dynamically test whether an object is an identity object or a
    value object can be found in the `java.util.Objects` class.

-   The `java.lang.ref` API rejects attempts to create references to value
    objects. 

-   Core reflection has added support for the `identity` and `value` modifiers.
    A `MethodHandle` for value class instance creation can be produced using the
    `Lookup.findStatic` method (with method name `"<vnew>"`).

-   Value class files generated by `javac` are meant only for use with this
    early access release, and will probably cause errors if loaded by other JVMs
    or processed by third-party tools.

-   When experimental primitive class support is active, primitive value types
    are expressed in class files with special `Q` descriptors, and in core
    reflection using special class objects, distinct from `ClassName.class`.

-   HotSpot support for CDS, JVMTI, and the serviceability agent has not been
    fully tested and unexpected behavior may occur.

-   `jshell` works with value classes but does not support the experimental
    primitive class features.

-   `javap` recognizes all the new class file features.
