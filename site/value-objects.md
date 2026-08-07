# Value Objects

[*Value objects*](https://openjdk.org/jeps/401) are immutable objects that
lack identity. They are distinguished solely by the values of their fields, and
can be represented by Java Virtual Machines in ways that improve performance.

Value objects will be available in JDK 28 as a preview feature.
Early-access builds are published at **<https://jdk.java.net/28/>**.
Interested developers are encouraged to download and experiment with these builds!

## Getting Started

*Value objects* model immutable data. A value object is an instance of
a *value class*, declared with the `value` modifier. Developers can save memory
and improve performance by using value objects for immutable data.

To get started, you can review this
**[short introduction](https://inside.java/2025/10/27/try-jep-401-value-classes/)**
to value objects at *inside.java*.

For a more in-depth overview,
**[JEP 401: Value Objects (Preview)](https://openjdk.org/jeps/401)**
describes the new feature, and detailed language and VM
**[specification changes](https://download.java.net/java/early_access/jdk28/docs/specs/value-objects-jls.html)**
are available.

## Value Object Performance

JDK 28 focuses on two optimizations for value objects: *reference flattening*,
which reduces the memory footprint of value objects stored in fields and arrays,
and *reference scalarization*, which avoids memory allocation for value objects
in JIT-compiled code. Details of these optimizations are
[discussed in the JEP](https://openjdk.org/jeps/401#Run-time-optimizations-for-value-objects).
You can also learn more with a
[presentation by Frederic Parain](https://www.youtube.com/watch?v=NF4CpL_EWFI)
from the 2025 JVM Language Summit.

Interested developers should download the
[early-access build](https://jdk.java.net/28/)
and try it out on their performance-sensitive workloads. 
Profiling tools like
[JDK Mission Control](https://docs.oracle.com/en/java/java-components/jdk-mission-control/)
and the
[Java Microbenchmark Harness (JMH)](https://github.com/openjdk/jmh)
can help to track how performance changes with value objects.

Some important notes about value object performance:

-   Heap-flattened encodings are always read and written atomically, and need to
    account for `null` references. This means value objects with 64 bits or more
    of field data cannot normally be flattened.
    [Future work](https://openjdk.org/jeps/401#Future-Work) will explore
    various techniques to overcome these limitations.

-   Scalarization does not happen until C2 compilation. During the warmup phase,
    value objects are allocated on the heap, as usual. Reads from flattened
    fields and arrays may even produce *extra* allocations, but these can be
    expected to stop occurring after repeated execution.

-   Variables that are *polymorphic*—that is, that store objects of more than
    one concrete class type—cannot be flattened and are not typically
    scalarized. This includes variables in generic APIs. (Keep in mind, however,
    that the JIT compiler may be able to use sharper types than the types
    declared by the program.)
    [Future work](https://openjdk.org/jeps/401#Future-Work) will allow generic
    code to be optimized for each instantiation that uses a different value
    class type.

-   Older class files that use a migrated value class may not be fully
    optimized. For best performance, compile all classes that refer to a value
    class with `--enable-preview`.

## Compatibility

Attempts to use value classes may encounter some behavioral incompatibilities
and other limitations.

**Language semantics:** The JEP includes some helpful guidance on the
[migration of existing classes](https://openjdk.org/jeps/401#Migrating-to-value-classes).
Most classes that meet the requirements can be compatibly migrated without any
issues, but there are some behavioral changes to be aware of.

**Library support:** The JEP discusses
[limitations of some Java Platform APIs](https://openjdk.org/jeps/401#Value-classes-in-the-Java-Platform)
when interacting with value objects. Potential areas of concern include
serialization (`java.io.ObjectOutputStream` and `java.io.ObjectInputStream`),
deep reflection (`java.lang.reflect.Field.setAccessible`), and garbage
collection (`java.lang.ref` and `java.util.WeakHashMap`).

## Sending Feedback

Feedback at **<valhalla-dev@openjdk.org>** is welcome and encouraged!
(To send e-mail to this address you must first
[subscribe to the mailing list](http://mail.openjdk.org/mailman/listinfo/valhalla-dev).)

We are particularly interested in experiences using real-world applications and
workloads. Let us know how you're using the feature, and whether there are any
bugs or performance pitfalls we should be aware of.
