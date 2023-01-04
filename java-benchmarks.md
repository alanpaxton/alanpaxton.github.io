# RocksDB Java API Performance Improvements

Evolved Binary has been working on several aspects of how the Java API to RocksDB can be improved. Two aspects of this which are of particular importance are performance and the developer experience.

* We have built some synthetic benchmark code to determine which are the most efficient methods of transferring data between Java and C++.
* We have used the results of the synthetic benchmarking to guide plans for rationalising the API interfaces.
* We have made some opportunistic performance optimizations/fixes within the Java API which have already yielded noticable improvements.

## Synthetic JNI API Performance Benchmarks
The synthetic benchmark repository contains tests designed to isolate the Java to/from C++ interaction of a canonical data intensive Key/Value Store implemented in C++ with a Java (JNI) API layered on top.

JNI provides several mechanisms for allowing transfer of data between Java buffers and C++ buffers. These mechanisms are not trivial, because they require the JNI system to ensure that Java memory under the control of the JVM is not moved or garbage collected whilst it is being accessed outside the direct control of the JVM.

We set out to determine which of multiple options for transfer of data from `C++` to `Java` and vice-versa were the most efficient. We used the [Java Microbenchmark Harness](https://github.com/openjdk/jmh) to set up repeatable benchmarks to measure all the options.

We explore these and some other potential mechanisms in the detailed results (in our [Synthetic JNI performance repository](https://github.com/evolvedbinary/jni-benchmarks/blob/main/DataBenchmarks.md))

We summarise this work here:

### The Model

* In `C++` we represent the on-disk data as an in-memory map of `(key, value)`
  pairs.
* For a fetch query, we expect the result to be a Java object with access to the
  contents of the _value_. This may be a standard Java object which does the job
  of data access (a `byte[]` or a `ByteBuffer`) or an object of our own devising
  which holds references to the value in some form (a `FastBuffer` pointing to
  `com.sun.unsafe.Unsafe` unsafe memory, for instance).

### Data Types

There are several potential data types for holding data for transfer, and they
are unsurprisingly quite connected underneath.

#### Byte Array

The simplest data container is a _raw_ array of bytes.

```java
byte[]
```

There are 3 different mechanisms for transferring data between a `byte[]` and
C++

* At the C++ side, the method
  [`JNIEnv.GetArrayCritical()`](https://docs.oracle.com/en/java/javase/13/docs/specs/jni/functions.html#getprimitivearraycritical)
  allows access to a C++ pointer to the underlying array.
* The `JNIEnv` methods `GetByteArrayElements()` and `ReleaseByteArrayElements()`
  fetch references/copies to and from the contents of a byte array, with less
  concern for critical sections than the _critical_ methods, though they are
  consequently more likely/certain to result in (extra) copies.
* The `JNIEnv` methods `GetByteArrayRegion()` and `SetByteArrayRegion()`
  transfer raw C++ buffer data to and from the contents of a byte array. These
  must ultimately do some data pinning for the duration of copies; the
  mechanisms may be similar or different to the _critical_ operations, and
  therefore performance may differ.

#### Byte Buffer

This container abstracts the contents of a collection of bytes, and was in fact
introduced to support a range of higher-performance I/O operations in some
circumstances.

```java
ByteBuffer
```

There are 2 types of byte buffers in Java, _indirect_ and _direct_. Indirect
byte buffers are the standard, and the memory they use is on-heap as with all
usual Java objects. In contrast, direct byte buffers are used to wrap off-heap
memory which is accessible to direct network I/O. Either type of `ByteBuffer`
can be allocated at the Java side, using the `allocate()` and `allocateDirect()`
methods respectively.

Direct byte buffers can be created in C++ using the JNI method
[`JNIEnv.NewDirectByteBuffer()`](https://docs.oracle.com/en/java/javase/13/docs/specs/jni/functions.html#newdirectbytebuffer)
to wrap some native (C++) memory.

Direct byte buffers can be accessed in C++ using the
[`JNIEnv.GetDirectBufferAddress()`](https://docs.oracle.com/en/java/javase/13/docs/specs/jni/functions.html#GetDirectBufferAddress)
and measured using
[`JNIEnv.GetDirectBufferCapacity()`](https://docs.oracle.com/en/java/javase/13/docs/specs/jni/functions.html#GetDirectBufferCapacity)

#### Unsafe Memory

```java
com.sun.unsafe.Unsafe.allocateMemory()
```

The call returns a handle which is (of course) just a pointer to raw memory, and
can be used as such on the C++ side. We could turn it into a byte buffer on the
C++ side by calling `JNIEnv.NewDirectByteBuffer()`, or simple use it as a native
C++ buffer at the expected address, assuming we record or remember how much
space was allocated.

Our `FastBuffer` class provides access to unsafe memory from the Java side.


#### Allocation

For these benchmarks, allocation has been excluded from the benchmark costs by
pre-allocating a quantity of buffers of the appropriate kind as part of the test
setup. Each run of the benchmark acquires an existing buffer from a pre-allocated
FIFO list, and returns it afterwards. A small test
confirmed that the request and return cycle is of insignificant cost compared to
the benchmark API call.

### GetJNIBenchmark

These benchmarks are distilled to be a measure of

- Carry key across the JNI boundary frm Java
- Look key up in C++
- Get the resulting value into the supplied buffer
- Carry the result back across the JNI boundary

Benchmarks ran for a duration of order 6 hours on an otherwise unloaded VM,
  the error bars are small and we can have strong confidence in the values
  derived and plotted.

Comparing all the benchmarks as the data size tends large, the conclusions we
can draw are:

- Indirect byte buffers add cost; they are effectively an overhead on plain
  `byte[]` and the JNI-side only allows them to be accessed via their
  encapsulated `byte[]`.
- `SetRegion` and `GetCritical` mechanisms for copying data into a `byte[]` are
  of very comparable performance; presumably the behaviour behind the scenes of
  `SetRegion` is very similar to that of declaring a critical region, doing a
  `memcpy()` and releasing the critical region.
- `GetElements` methods for transferring data from C++ to Java are consistently
  less efficient than `SetRegion` and `GetCritical`.
- Getting into a raw memory buffer, passed as an address (the `handle` of an
  `Unsafe` or of a netty `ByteBuf`) is of similar cost to the more efficient
  `byte[]` operations.
- Getting into a direct `nio.ByteBuffer` is of similar cost again; while the
  ByteBuffer is passed over JNI as an ordinary Java object, JNI has a specific
  method for getting hold of the address of the direct buffer, and this done the
  `get()` cost is effectively that of the `memcpy()`.

![Raw JNI Get](./analysis/get_benchmarks/fig_1024_1_none_nopoolbig.png).

At small(er) data sizes, we can see whether other factors are important.

- Indirect byte buffers are the most significant overhead here. Again, we can
  conclude that this is due to pure overhead compared to `byte[]` operations.
- At the lowest data sizes, netty `ByteBuf`s and unsafe memory are marginally
  more efficient than `byte[]`s or (slightly less efficient) direct
  `nio.Bytebuffer`s. This may perhaps be explained by even the small cost of
  entering the JNI model in some fashion on the C++ side simply to acquire a
  direct buffer address. The margins (nanoseconds) here are extremely small.

![Raw JNI Get](./analysis/get_benchmarks/fig_1024_1_none_nopoolsmall.png).

#### Post processing the results

Our benchmark model for post-processing is to transfer the results into a
`byte[]`. Where the result is already a `byte[]` this may seem like an unfair
extra cost, but the aim is to model the least cost processing step for any kind
of result.

- Copying into a `byte[]` using the bulk methods supported by `byte[]`,
  `nio.ByteBuffer` are comparable.
- Accessing the contents of an `Unsafe` buffer using the supplied unsafe methods
  is inefficient. The access is effectively byte by byte, or at best word by
  word, in Java.
- Accessing the contents of a netty `ByteBuf` is similarly inefficient; again
  the access is presumably byte by byte, or at best word by word, using normal
  Java mechanisms.

![Copy out JNI Get - TODO - replace the plots](./analysis/get_benchmarks/fig_1024_1_copyout_nopoolbig.png).

### PutJNIBenchmark

We benchmarked `Put` methods in a similar synthetic fashion. Similar conclusions apply; using `GetElements` is the least-performance way of implementing transfers to/from Java objects in C++/JNI.

## Lessons from Synthetic API

Performance analysis shows that for `get()`, fetching into allocated `byte[]` is
equally as efficient as any other mechanism. Copying out or otherwise using the
result is straightforward and efficient. Using `byte[]` avoids the manual memory
management required with direct `nio.ByteBuffer`s, which extra work does not
appear to provide any gain. A C++ implementation using the `GetRegion` JNI
method is probably to be preferred to using `GetCritical` because while their
performance is equal, `GetRegion` is a higher-level/simpler abstraction.

Vitally, whatever JNI transfer mechanism is chosen, the buffer allocation
mechanism and pattern is crucial to achieving good performance. We experimented
with making use of netty's pooled allocator part of the benchmark, and the
difference of `getIntoPooledNettyByteBuf`, using the allocator, compared to
`getIntoNettyByteBuf` using the same pre-allocate on setup as every other
benchmark, is significant.

Equally importantly, transfer of data to or from buffers should where possible
be done in bulk, using array copy or buffer copy mechanisms. Thought should
perhaps be given to supporting common transformations in the underlying C++
layer.

## Recommendations

Of course there is some noise within the results. but we can agree:

 * Don't make copies you don't need to make
 * Don't allocate/deallocate when you can avoid it
 
Translating this into designing an efficient API, we want to:

 * Support API methods that return results in buffers supplied by the client.
 * Support `byte[]`-based APIs as the simplest way of getting data into a usable configuration for a broad range of Java use.
 * Support direct `ByteBuffer`s as these can reduce copies when used as part of a chain of `ByteBuffer`-based operations. This sort of sohpisticated streaming model is most likely to be used by clients where performance is important, and so we decide to support it.
 * Support indirect `ByteBuffer`s for a combination of reasons:
   * API consistency between direct and indirect buffers
   * Simplicity of implementation, as we can wrap `byte[]`-oriented methods
 * Continue to support methods which allocate return buffers per-call, as these are the easiest to use on initial encounter with the RocksDB API.

High performance Java interaction with RocksDB ultimately requires architectural decisions by the client 
 * Use more complex API methods where performance matters
 * Don't allocate/deallocate where you don't need to
   * recycle your own buffers where this makes sense
   * or make sure that you are supplying the ultimate destination buffer (your cache, or a target network buffer) as input to RockSDB `get()` and `put()` calls

## Optimizations

### Saving between 0 and 1 Copies

Having analysed JNI performance as described, we reviewed the core of RocksJNI for opportunities to improve the performance. We noticed one thing in particular; some of the `get()` methods of the Java API had not been updated to take advantage of the new [`PinnableSlice`](http://rocksdb.org/blog/2017/08/24/pinnableslice.html) methods.

Fixing this turned out to be a straightforward change, which has now been incorporated in the codebase [Improve Java API `get()` performance by reducing copies](https://github.com/facebook/rocksdb/pull/10970)

#### Performance Results