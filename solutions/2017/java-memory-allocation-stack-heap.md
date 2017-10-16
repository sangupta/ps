# Java Memory Management

```
This article is in response to request https://github.com/sangupta/ps/issues/6
```

This article tries to explain the concept of Java memory management and
the nuances between stack and heap allocation for objects. This does not
intend to get into the advance techniques of memory management. So let's
get started.

Java Virtual Machine (JVM) or for that matter other runtimes like Common
Language Runtime (CLR for .NET) allocates memory to attributes in one of
the two: either stack or heap. There are other designated areas like the
perm generation, string-pools etc. but let's not consider them for now.

## Stack space

Every Java program runs as a process and has multiple threads associated
with it. Even a simple `Hello World` program will spin up threads like the
garbage collector and all. These threads are JVM threads and should not
be confused with the program threads.

Each thread in the JVM gets a chunk of memory allocated to itself, which
is called the stack memory. This is used by the thread to hold a stack
of the function jumps of where the current instruction being executed is.
This memory also holds any variables that are allocated in the method
scope. The stack space is NOT allocated from the heap memory (explained
later) and thus the total memory consumed by a Java process would be the
sum of `stack + heap + JVM process`.

The default thread stack size varies with JVM, OS and environment variables. 
A typical value is 512k. It is generally larger for 64bit JVMs because 
references are 8 bytes rather than 4 bytes in size. This means that if your 
app uses 100 threads, 50MB will be used for thread stacks. In some environments 
the defaults stack may be as large as 2MB. With a large number of threads, 
this can consume a significant amount of memory which could otherwise be 
used by your application or OS. We can tweak the stack size allocated by 
using the `-Xss` JVM flag.

All primitive variables running in the method scope are assigned memory from
the stack space.

## Heap space

`Heap` or `Heap-space` in JVM is a common shared memory area which is shared
amongst all the threads of the virtual machine. This area is used to allocate
memory for all class instances, arrays and more. This is like the global scratch
space for the program.

It is created at the start of JVM and lives till the JVM itself terminates. 
Threads may start and die, but the heap itself is eternal till the life of the
JVM. Heap space may grow as a program executes, and may shrink when the JVm figures
that memory can be released to the operating-system due to memory-lean period of
execution.

All object instances whether allocated at class level or allocated at method
level, are assigned memory in the heap space. As all classes can only be used
as objects, they also occupy space in heap.

## Understanding allocation

To understand the allocation of memory for different kinds of variables, let's
consider the following code piece:

```java
public class SomeClass {

	public void someMethod() {

		// 4 bytes are allocated on stack for holding the primitive
		int x = 3;

	}

}
```

In the example above the variable `x` is assigned memory on the stack. The
primitive needs 4 bytes of allocation and these are provided from the stack
memory of the thread that will be executing the code.

Consider, the following code piece now:

```java
ublic class SomeClass {

	public void someMethod() {

		// 4 bytes are allocated on stack for holding the reference
		// or the so called pointer to the object that is being created
		// in heap
		AnotherObject anotherInMethod = new AnotherObject();

	}

}
```

In the instantiation above, first, a new `Object` of type `AnotherObject` is
created in the heap space. All class-level attributes of this object are allocated
space within the heap.

The pointer reference to this object for the current thread is however, stored
in the stack space. Thus 4 bytes for pointer reference (8 bytes on 64-bit JVM)
are allocated for a pointer, that points to the object in heap.

Now, consider the following code piece:

```java
public class SomeClass {

	// an object-level primitive attribute
	int y = 10;

	// an object-level non-primitive attribute
	AnotherObject anotherInClass = new AnotherObject();

}
```

In the example above, the attribute `anotherInClass` occupies the memory in heap
as before. Note that the pointer itself to this reference is also held in the 
heap space, as the instance of the `SomeClass` object lies in the heap space.

Note that the primitive attribute `y` is also allocated space at the heap level
because the instance of `SomeClass` is itself allocated in heap.

## Static variables and allocation

As we are already aware that all classes itself are `Object`s in JVM world. Not
just the class instances but the classes that are loaded in memory. Think of
`reflection` here.

```java
public class SomeClass {

	public void someMethod() {

		// see the class itself is an object and this instance 
		// is allocated in the heap
		Class<?> myClassObject = SomeClass.class;

	}
}
```

I hope by now you would have guessed, where, `static` attributes are allocated. Yes,
they are allocated in heap, as the `class` object itself lies in heap.

## String allocation

`String` objects have a different kind of allocation. `String` objects themselves are
not allocated in heap or stack space. They occupy a memory portion called as `String-pool`.
As `String` objects are immutable, the literal strings are allocated memory in this pool
space and all attributes whether class-level or method-level refer to the same `String`
in the `string-pool`.

Consider the following code piece and the memory allocations associated with it:


```java
public class SomeClass {

	// the reference pointer is allocated space in heap
	public String classLevelString = "Hello World";

	public void someMethod() {

		// the reference pointer is allocated space in stack
		String methodLevelString = "Hello world";

	}
}
```

Note in the example above, both the attributes occupy space either in stack or heap
depending on where they are initiailized. However, because both the variables point to
the same exact literal string, they point to the same reference in the pool space.

# TL;DR

The following example illustrates where space is being allocated:

```java
public class SomeClass {

	// allocated in heap
	public static int num1 = 5;

	// both reference and object allocated in heap
	public static CustomObject co1 = new CustomObject();

	// reference in heap, literal in pool
	public static String s1 = "Hello world";

	// allocated in heap
	pubic int num2 = 5;

	// both reference and object allocated in heap
	public CustomObject co2 = new CustomObject();

	// reference in heap, literal in pool
	public String s2 = "Hello world";


	public void someMethod() {

		// allocated in stack
		int num3 = 10;

		// reference in stack, object in heap
		CustomObject co3 = new CustomObject();

		// reference in stack, literal in pool
		String s3 = "Hello world";
	}

}
```

Hope the above example summarizes the allocations well.
