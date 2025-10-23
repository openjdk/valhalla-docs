# Early Access Release

The latest early access release of Valhalla features is published at
**[https://jdk.java.net/valhalla/](https://jdk.java.net/valhalla/)**.

This release is a modified build of [JDK 26](https://jdk.java.net/26/)
that fully supports
[JEP 401: Value Classes and Objects (Preview)](https://openjdk.org/jeps/401).
We encourage users to experiment with value objects and explore their
performance implications. We welcome user feedback at
<span style="white-space:nowrap;">`valhalla-dev@openjdk.org`</span>
as we work toward a preview release in an upcoming JDK.

Functionality, language syntax, and `class` file formats are subject to change.
Some supplementary tools and APIs do not fully support the new features.
The goal of each EA release is to solicit feedback and establish a milestone
along the path to an official preview feature.


## Value Classes and Objects

See the [Value Classes and Objects JEP](https://openjdk.org/jeps/401)
for an in-depth overview of the new features. Detailed
[language and VM changes](https://cr.openjdk.org/~dlsmith/jep401/latest)
are available as well.

When the `--enable-preview` command-line flag is used, Java programs are allowed
to use `value` classes and records, which have only `final` fields and lack
object identity. At run time, the `==` operator compares value class instances
according to their field values, without regard to when or where they were
created. Other identity-sensitive operations have been adjusted to appropriately
work with objects that lack identity.

Developers can save memory and improve performance by using value objects for
immutable data. Because programs cannot tell the difference between two value
objects with identical field values (not even with `==`), the Java Virtual
Machine is able to change how a value object is laid out in memory without
affecting the program; for example, its fields could be stored on the stack
rather than the heap.

By enabling preview features, developers also get value-class versions of
`java.lang.Integer`, `java.time.LocalDate`, `java.util.Optional`, and a
handful of other classes in the standard libraries. Developers are encouraged
to test their code against these modified libraries, along with declaring their
own value classes.

Please provide any feedback to
<span style="white-space:nowrap;">`valhalla-dev@openjdk.org`</span>.


## Notes

-   The JEP includes some helpful guidance on the
    [migration of existing classes](https://openjdk.org/jeps/401#Migrating-to-value-classes).

-   The JEP also discusses
    [limitations of some Java Platform APIs](https://openjdk.org/jeps/401#Value-classes-and-the-Java-Platform)
    when interacting with value objects. Potential areas of concern include
    serialization (`java.io.ObjectOutputStream` and `java.io.ObjectInputStream`),
    deep reflection (`java.lang.reflect.Field.setAccessible`),
    and garbage collection (`java.lang.ref` and `java.util.WeakHashMap`).

-   CDS archives that support the Compact Object Headers feature while using the
    `--enable-preview` flag are not included by default in the EA build. If
    Compact Object Headers are being used, developers can generate an
    appropriate CDS archive to optimize start-up performance as follows:

    ```
    java -Xshare:dump --enable-preview -XX:+UseCompactObjectHeaders
    ```

    Or, with compressed oops disabled:

    ```
    java -Xshare:dump --enable-preview -XX:+UseCompactObjectHeaders -XX:-UseCompressedOops
    ```

    These commands will add a new file (`classes_coh_valhalla.jsa` or
    `classes_nocoops_coh_valhalla.jsa`) to the directory where other CDS
    archives are stored (e.g., `<jdk>/lib/server`).

<!--
-   On macOS, after expanding the `tar.gz` archive, you may need to remove the
    quarantine attribute from the bits before commands can be executed. Use
    Terminal to execute the following command:

    ```
    xattr -d com.apple.quarantine ./jdk-26.jdk
    ```
-->