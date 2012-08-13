---
layout: body
title: "Deep Dive: Java's Lock-Free Concurrency"
comments: true
---

##{{ page.title }}
<span class="blog_date">{{ page.date | date:'%B %d, %Y' }}</span>
Java is increasingly the language of choice for systems programming, 
especially in open source
projects like Hadoop. While the JVM provides useful, time-saving
abstractions, those same simplifications often hide performance implications.
Often people realize this too late, once a system is running (slowly). 
To keep myself honest, I try to dig through JVM code to see exactly what is 
going on in various performance-sensitive constructs, such as concurrency
mechanisms.

This weekend I looked at Java's `AtomicInteger` class, which supports 
lock-free concurrent operation by leveraging underlying atomic instructions 
in modern CPU's. This post gives a deep dive into how these instructions percolate through the JVM into actual assembly code. It digs through three levels
of JVM source code, then <a href="#conclusion">concludes</a>.

### java.util.concurrent.atomic.AtomicInteger
Java's [`AtomicInteger`](http://docs.oracle.com/javase/7/docs/api/java/util/concurrent/atomic/AtomicInteger.html) class
represents an integer value that one can increment, decrement, and 
compare-and-swap safely from multiple threads without explicit
synchronization. It is a developer facing utility class contained in the 
`java.util.concurrent.atomic` package. 
Internally, `AtomicInteger` implements all atomic operations in terms of 
its own `compareAndSet` function. 
[Compare-and-swap](http://en.wikipedia.org/wiki/Compare-and-swap) (CAS) is a fundamental building block in many concurrent systems. In this case,
`compareAndSet` is pretty boring:

* * *

{% highlight java %}
    /**
     * Atomically sets the value to the given updated value
     * if the current value {@code ==} the expected value.
     */
    public final boolean compareAndSet(int expect, int update) {
        return unsafe.compareAndSwapInt(this, valueOffset, expect, update);
    }
{% endhighlight %} 

* * *

### sun.misc.Unsafe
`AtomicInteger`'s `compareAndSet` delegates to `compareAndSwapInt` in the famed 
[`sun.misc.Unsafe`](http://www.docjar.com/html/api/sun/misc/Unsafe.java.html)
class. `Unsafe` is an internal utility class used by several Java libraries.
It is chock full of low-level, abstraction-breaking goodness. 
For instance, `Unsafe` lets you access and modify memory at arbitrary locations.
Because it requires information about exact memory addresses, 
most code in `Unsafe` is native, meaning it is written directly 
in c++ rather than Java. `AtomicInteger` must pass a memory offset 
(`valueOffset` above) to identify which memory region the CAS should be executed
against. This offset is determined elsewhere in the code, using another utility function in `Unsafe`. Becasue `compareAndSwapInt` is native, we only 
see a function signature in the class itself:

* * * 

{% highlight java %}
    /**
     * Atomically update Java variable to <tt>x</tt> if it is currently
     * holding <tt>expected</tt>.
     * @return <tt>true</tt> if successful
     */
    public final native boolean compareAndSwapInt(Object o, long offset,
                                                  int expected,
                                                  int x);
{% endhighlight %}

* * *

<p style="margin-top:10px;">
Digging up the implementation requires downloading JDK source code, 
which is available through the
<a href="http://download.java.net/openjdk/jdk7/">
OpenJDK source release page</a>.
The files considered from this point forward are found in /openjdk/hotspot/,
and the relevant bits are included below.
</p>

### /src/share/vm/prims/unsafe.cpp
The `Unsafe_CompareAndSwapInt` function is the bridge between the JVM's 
object-oriented semantics and the hardware's memory-oriented instructions. 
It takes the relative memory offset passed in from 
`AtomicInteger` and calculates the absolute memory address of the integer
being modified. That address is what it needs to pass to the underlying
hardware instruction. Performing the instruction call correctly depends on the
underlying hardware. Thus, this calls the hardware-dependent `cmpxchg` 
function, which is inserted at compile time.

* * *

{% highlight c++ %}
UNSAFE_ENTRY(jboolean, Unsafe_CompareAndSwapInt(
  JNIEnv *env, jobject unsafe, jobject obj, jlong offset, jint e, jint x))
  UnsafeWrapper("Unsafe_CompareAndSwapInt");
  oop p = JNIHandles::resolve(obj);
  jint* addr = (jint *) index_oop_from_field_offset_long(p, offset);
  return (jint)(Atomic::cmpxchg(x, addr, e)) == e;
UNSAFE_END
{% endhighlight %}

* * *

### /src/os_cpu/linux_x86/vm/atomic_linux_x86.inline.hpp 
OpenJDK includes several architecture-dependent `cmpxchg` implementations. 
I run on x86, so my JVM is compiled against the function below, which
is contained in the `atomic_linux_x86.inline` header file.
This directly calls the 
[`cmpxchgq` assembly instruction](http://faydoc.tripod.com/cpu/cmpxchg.htm), 
a 64-bit compare-and-swap operation.
The `LOCK_IF_MP` macro first determines whether the lock prefix needs to be 
attached to the instruction. This prefix ensures that the CAS will be atomic
even in multi-processor architectures. That is, extra care needs to be taken
despite the fact that `cmpxchgq` is "atomic" on a single CPU.

* * *

{% highlight c++ %}
// Adding a lock prefix to an instruction on MP machine
#define LOCK_IF_MP(mp) "cmp $0, " #mp "; je 1f; lock; 1: "
...
inline jlong Atomic::cmpxchg(jlong exchange_value, volatile jlong* dest, 
  jlong compare_value) {
  bool mp = os::is_MP();
  __asm__ __volatile__ (LOCK_IF_MP(%4) "cmpxchgq %1,(%3)"
                        : "=a" (exchange_value)
                        : "r" (exchange_value), "a" (compare_value)
                        : "r" (dest), "r" (mp), "cc", "memory");
  return exchange_value;

{% endhighlight %}

* * *

<a name="conclusion" style="text-decoration:none">
<h3>Learning from our deep dive - why use AtomicInteger?</h3>
</a>
Any time an `AtomicInteger` method is called, the `cmpxchgq` assembly 
instruction is
ultimately triggered. From the JVM's perspective, this is 
considered a nonblocking or "lock-free" method. That is, unlike a 
`synchronized` statement, which triggers a mutex acquire, it will never suspend
the calling thread or cause a context-switch. In addition to never blocking,
atomic counters entirely avoid the OS interaction required by kernel-based 
mutex implementations. There is no need to switch to kernel space when
performing a concurrent operation, since no kernel facilities are used.
The performance benefits of atomic counters stem from avoiding these two 
sources of overhead.

It should be noted, however, that _we aren't really avoiding
the issue of locking_, just pushing it down into the hardware. If two threads
in different cores are trying to update an `AtomicInteger` simultaneously, 
one of them will assert the bus lock first and the other will stall until 
the first one finishes. This is a type of mini-(b)locking that can, indeed 
happen, and may decrease performance if the shared memory address is highly 
contended. The good news is that this locking is happening at 
_extremely fine granularity_ -- that of a single instruction -- so the
probability of contention is much lower. Under very high
degrees of contention, it remains possible that `AtomicInteger` could 
underperform an analogous `synchronized` implementation, but my hunch is 
that this rarely happens. As a general rule, it is prudent to design around
hardware-backed counters if they are sufficiently expressive for your
application.

Thanks to [@apanda](https://twitter.com/apanda) and [@tlipcon](https://twitter.com/tlipcon) for help with this.
