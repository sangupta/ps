# Testing for concurrency in Java

Testing for concurrency is not as straight forward as asserting for conditions
to be `true` or `false`. It requires a thorough understanding of concurrency as
well as the various possible bugs that mainfest in concurrent code.

TL;DR - read as much as you can on concurrency including existing tests for popular
Java concurrent code from JDK and other open-source libraries. Utilize static code
analyzers for detecting early bugs.

## Locating concurrent code

The first step towards testing for concurrency is to locate concurrent code. This
is the easiest step in all other steps. Any occurrence of the code below suggests
presence of concurrency:

```
extends Thread
implements Runnable // this may not for sure be concurrent

// keywords such as
synchronized
volatile

// usage of methods such as
.notify()
.notifyAll()
.wait()
.wait(...)
.interrupt()
.join()
.join(...)
.interrupted()
.sleep()
.yield()

// usage of classes/packages such as
java.util.concurrent
java.util.concurrent.Atomic
java.util.concurrent.locks

// usage of library classes like
com.google.common.util.concurrent.ServiceManager
```

## Mark all your code with concurrency annotations

The next step in testing for concurrency is to mark all your code with concurrency
related annotations as specified in **Java Concurrency in Practice** book. The
annotations available are:

* `NotThreadSafe` - The presence of this annotation indicates that the author believes
the class is not thread-safe. The absence of this annotation does not indicate that
the class is thread-safe, instead this annotation is for cases where a naïve assumption
could be easily made that the class is thread-safe. In general, it is a bad plan
to assume a class is thread safe without good reason.

* `ThreadSafe` - The presence of this annotation indicates that the author believes
the class to be thread-safe. As such, there should be no sequence of accessing the
public methods or fields that could put an instance of this class into an invalid
state, irrespective of any rearrangement of those operations by the Java Runtime
and without introducing any requirements for synchronization or coordination by
the caller/accessor.

* `GuardedBy` - The specified lock that guards the annotated field or method.

* `Immutable` - The presence of this annotation indicates that the author believes
the class to be immutable and hence inherently thread-safe. An immutable class is
one where the state of an instance cannot be **seen** to change.

## Using static code analyzers

There are many static code analyzers available that can help detect concurrency
bugs such as:

* [CheckThread](https://dzone.com/articles/checkthread-a-static-analysis-tool-for-java-concurrency-bugs)
* [FindBugs](http://findbugs.sourceforge.net/)
* [Coverity](https://www.coverity.com)
* [Jlint](http://jlint.sourceforge.net/)
* [JTest](https://www.parasoft.com/product/jtest/)
* [Another Paper](http://people.csail.mit.edu/amy/papers/deadlock-ecoop05.pdf)

If the code is properly annotated with the JCIP concurrency annotations then
there are high chances that the static code analyzers will help detect the basic
mistakes that occur with concurrency.

## Avoid known basic pitfalls

* Avoid using non-synchronized Java objects such as `java.text.SimpleDateFormat`,
`StringBuilder` without proper guards

## Writing JUnit tests

This is the last piece in testing concurrent code. The following articles explain
with code samples on how various concurrent code pieces have been tested. Another
nice place to read is to go thorugh the JUnit tests written for popular libraries
like Google Guava.

* [This article](https://codurance.com/2015/12/13/testing-multithreaded-code-in-java/)
explains with sample code on how to test an atomic big-counter class.

* [This article](https://zeroturnaround.com/rebellabs/concurrency-torture-testing-your-code-within-the-java-memory-model/)
samples code on how to test concurrent access to a BitSet.

* [This article](https://garygregory.wordpress.com/2011/09/09/multi-threaded-unit-testing/)
explains how to test a unique ID generator for concurreny.

* [JUnit test](https://github.com/google/guava/blob/master/guava-tests/test/com/google/common/util/concurrent/ServiceManagerTest.java)
for the Google Guava `ServiceManager` class

* [Apache Harmony](https://github.com/apache/harmony) was on open-source implementation
of JDK 5. It includes JUnit tests for classes like [AtomicInteger](https://github.com/apache/harmony/blob/java6/classlib/modules/concurrent/src/test/java/AtomicIntegerTest.java)
, [ConcurrentHashMap](https://github.com/apache/harmony/blob/java6/classlib/modules/concurrent/src/test/java/ConcurrentHashMapTest.java)
, [ReentrantReadWriteLock](https://github.com/apache/harmony/blob/java6/classlib/modules/concurrent/src/test/java/ReentrantReadWriteLockTest.java)
, [ExecutorsTest](https://github.com/apache/harmony/blob/java6/classlib/modules/concurrent/src/test/java/ExecutorsTest.java) and more

## Resources

* [ConcurrentUnit](https://github.com/jhalterman/concurrentunit) - an extension
for JUnit helping with testing concurrent code

* [Java Concurrency in Practice](http://jcip.net/) - IMHO the best book on Java
concurrency

* [Comparing concurrency static-code analyzers](http://robertfeldt.net/publications/grahn_2010_comparing_static_analysis_tools_for_concurrency_bugs.pdf)

* [FindBugs](http://findbugs.sourceforge.net/)
