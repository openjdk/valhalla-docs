# Value Classes and Objects

[Value Classes and Objects](https://openjdk.org/jeps/401) are a transformative
new Java feature, currently being developed for inclusion in a future JDK
release. *Value objects* are instances of *value classes*, which have only final
fields and lack object identity.

The latest early-access build is published at
**<https://jdk.java.net/valhalla/>**.
Interested developers are encouraged to download and experiment with it!

## Introducing Value Objects

When the `--enable-preview` command-line flag is used, Java programs are allowed
to declare `value` classes and records, and to create value object instances.
At run time, the `==` operator compares value objects according to their field
values, without regard to when or where they were created. Other
identity-sensitive operations have been adjusted to appropriately work with
objects that lack identity.

With preview features enabled, certain standard library classes are made value
classes, including `java.lang.Integer`, `java.time.LocalDate`, and
`java.util.Optional`.

To get started, you can review this
**[short introduction](https://inside.java/2025/10/27/try-jep-401-value-classes/)**
at *inside.java*, which illustrates some of the new behavior.

For a more in-depth overview,
**[JEP 401: Value Classes and Objects (Preview)](https://openjdk.org/jeps/401)**
describes the new feature, and detailed language and VM
**[specification changes](https://cr.openjdk.org/~dlsmith/jep401/latest)**
are available.

## Value Object Performance

Developers can save memory and improve performance by using value objects for
immutable data. Because programs cannot tell the difference between two value
objects with identical field values (not even with `==`), the Java Virtual
Machine is able to change how a value object is laid out in memory without
affecting the program.

In this release, we've focused on two optimizations: *heap flattening*, which
reduces the memory footprint of value objects stored in fields and arrays, and
*scalarization*, which avoids memory allocation for value objects in
JIT-compiled code. Details of these optimizations are
[discussed in the JEP](https://openjdk.org/jeps/401#Run-time-optimizations-for-value-objects).
You can also learn more with a
[presentation by Frederic Parain](https://www.youtube.com/watch?v=NF4CpL_EWFI)
from the 2025 JVM Language Summit.

Interested developers should download the
[early-access build](https://jdk.java.net/valhalla/)
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
[limitations of some Java Platform APIs](https://openjdk.org/jeps/401#Value-classes-and-the-Java-Platform)
when interacting with value objects. Potential areas of concern include
serialization (`java.io.ObjectOutputStream` and `java.io.ObjectInputStream`),
deep reflection (`java.lang.reflect.Field.setAccessible`), and garbage
collection (`java.lang.ref` and `java.util.WeakHashMap`).

**Compact Object Headers:** In the EA build, CDS archives that support the
Compact Object Headers feature while using the `--enable-preview` flag are not
included by default in the EA build. If Compact Object Headers are being used,
developers can generate an appropriate CDS archive to optimize start-up
performance as follows:

```
java -Xshare:dump --enable-preview -XX:+UseCompactObjectHeaders
```

Or, with compressed oops disabled:

```
java -Xshare:dump --enable-preview -XX:+UseCompactObjectHeaders \
        -XX:-UseCompressedOops
```

These commands will add a new file (`classes_coh_valhalla.jsa` or
`classes_nocoops_coh_valhalla.jsa`) to the directory where other CDS
archives are stored (e.g., `<jdk>/lib/server`).


## Sending Feedback

Feedback at **<valhalla-dev@openjdk.org>** is welcome and encouraged!
(To send e-mail to this address you must first
[subscribe to the mailing list](http://mail.openjdk.org/mailman/listinfo/valhalla-dev).)

We are particularly interested in experiences using real-world applications and
workloads. The early-access build is beta software, and it's sure to have some
bugs and surprising performance pitfalls. Your feedback is a valuable tool to
help us identify these issues.
