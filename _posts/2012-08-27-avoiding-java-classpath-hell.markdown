---
layout: body
title: "Avoiding Java Classpath Hell"
comments: true
---

##{{ page.title }}
<span class="blog_date">{{ page.date | date:'%B %d, %Y' }}</span>

_Of all Java build problems, those surrounding classpaths and class loaders can be the most vexing to understand._

If you've been writing Java code for more than 1 day, you've probably imported a class from an external package that you didn't write. The moment you start writing Java code which relies on external libraries, you open yourself up to a world of pain and confusion. Java loads referenced classes dynamically at runtime. This loading will occur when the JVM is initialized and sometimes continually as an application runs. Because Java loads classes in a dynamic and configurable way, it can be very difficult to debug or even comprehend errors that come up around classpaths. **You don't need to become an expert on dynamic class resolution to solve most issues, you just need to understand two things:**

### 1. Understand How `java` Finds Classes
The JVM always loads classes through a [`ClassLoader`](http://docs.oracle.com/javase/6/docs/api/java/lang/ClassLoader.html). A `ClassLoader` implementation fulfills a simple contract: you give it a fully qualified class name (package and class) and it finds and loads the relevant byte code. The JVM actually includes three ClassLoaders by default, one to bootstrap and load core java classes, one to load extensions (classes you have in the /ext directory of your JRE), and another to load classes in the conventional Java classpath. Understanding the inns and outs of ClassLoaders is complicated and not necessary for solving most basic issues around class inclusion. It is useful to know that they exist.

The only class loader you need to care about in many cases is the System ClassLoader. That loader's default behavior is to load classes based on locations specified in the `-classpath` (short `-cp`) argument to `java` or in the `CLASSPATH` environment variable on UNIX systems. Either of these expects a semi-colon separated list of directories or jar files. Here is a simple example:

__`$ java -cp "compiled_classes/;lib/foo1.2.jar;lib/bar3.4.jar" edu.berkeley.Foo`__

There are two gotchas around classpath strings. First, directories will be searched for `.class` files, which will be included, but *directories will not be searched for jar files* -- jar files must be named directly. Second, *If multiple definitions for a class are found, the first one present in the classpath is loaded and no error is thrown*.

This already gives us some things to avoid:

__`$ java -cp "folder-containing-jars/"`__  _NO! Will not search folders for jars._

__`$ java -cp "coolLib1.2.jar;coolLib1.3.jar"`__ _NO! coolLib1.2 silently shades coolLib1.3_

### 2. Understand Common Class Loading Errors
The JVM tries to be helpful by throwing exceptions or errors when something in the classpath just isn't right. There are four common error messages which indicate issues with class loading. It is important to understand and differentiate what these errors mean and how to go about fixing them.
### `NoClassDefFoundError`
This means that your code correctly compiled against a class `Foo` but at runtime, that class `Foo` can't be located by any of the active ClassLoaders. The most common cause of this error is that the class files (e.g. `Foo.class`) or the jar file containing the class (e.g. `foolib1.2.jar`) simply aren't in the classpath. This error will occur on JVM initialization and is not recoverable
###`ClassNotFoundException`
This also means that no ClassLoader can find the byte code for some particular named class, but this exception is thrown when that class is instantiated through reflection (e.g. `class.forName("bar.Foo")`, `classLoader.loadClass("bar.Foo")`) rather than being referenced from compiled code. This differs from `ClassNotFoundError` in a few ways. First, it can be thrown at any time during an execution. If you don't believe me, put the line `class.forName("blahblahblah");` in your code and see. This is also an `Exception` rather than an unchecked `Error` because it is recoverable. Since it can only arise as a result of user code, that code can simply catch the exception and move on. If you get this exception,
it means that you tried to instantiate a non-existent class, or you instantiated
an existing class but forgot to include it in the classpath.
###`NoSuchMethodException`
This is the method equivalent of `ClassNotFoundException`. It is called when you use reflection(e.g. `getMethod("length", String.class)`) and the method doesn't actually exist.
###`NoSuchMethodError`
This is the worst of all of the possible errors you can get. If you get this error, take a deep breath and get ready to face one of several unpleasant realities. NoSuchMethodError means that you compiled against version A of the class and you are running against version B, where version B is not compatible with version A (it lacks a method contained by A). This could be the result of accidentally including a newer version of a class's jar file when running than when compiling. More often, it is caused by having two classes that depend on divergent versions of a third class (I see this a lot with commonly used libraries, like [Jetty](http://jetty.codehaus.org/jetty/)). If that's your problem, the best solution it to find versions of your dependencies that agree on their version the third party library. It is also possible to use multiple ClassLoaders to fix this in a much more complicated way, but that should be avoided at all costs (it is both complex and brittle).

## Avoiding the Hell
The best way to avoid classpath nightmares is to understand how class loading works in the JVM and exactly what Java is complaining about when classes cannot be loaded. The underlying issue is almost always a discrepancy between the environment in which your code was compiled and that in which it is run. With a solid appreciation for what these errors mean, it becomes easy to debug any class loading issue without getting extremely frustrated.
