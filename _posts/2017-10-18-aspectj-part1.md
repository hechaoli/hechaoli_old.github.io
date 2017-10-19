---
layout: post
title: AOP and AspectJ Part 1
tags: [Java, AOP, AspectJ]
---

# Aspect Oriented Programming (AOP)

Apect Oriented programming (AOP) is used to solve cross-cutting concerns. So
let's first take a look at what is a cross-cutting concern. Actually there are
many terminologies in AOP and we will go through them one by one.

## Cross Cutting Concern
There are two types of concern: core concern and cross-cutting concern.
**Core concern** is the primary functionality of the system. In contrast,
**cross-cutting concern** represents the secondary requirement and it is
applicable throughout the application. The following figure can help understand
the concept of cross-cutting concern.

![Cross-cutting concern](/img/cross-cutting-concern.jpg)

The arrows represent core concern while the planes represent cross-cutting
concerns. We can see that cross-cutting concerns provide ancillary
functionalities, such as security logging and transaction management, to all the
core concerns as they are common requirements of different components.

Once we understand cross-cutting concern, the concept of AOP is clear. AOP just
allows us to modularize crosscutting concerns. For example, instead of managing
transaction inside each function on our own, we can have an aspect that does that
for us. The following code shows how transaction is managed in a Spring AOP:

```java
@Transactional
public class DefaultFooService implements FooService {
    void insertFoo(Foo foo) {
        em.persist(foo);
    }
    void updateFoo(Foo foo) {
        em.merge(foo);
    }
    void removeFoo(String fooName) {
        Foo foo = em.find(Foo.class, fooName);
        em.remove(foo);
    }
}
```

After annotated with `@Transactional`, all the methods in the class becomes
transactional automatically. Without AOP, what you have to do is:

```java
public class DefaultFooService implements FooService {
    void insertFoo(Foo foo) {
        em.getTransaction().begin();
        em.persist(foo);
        em.getTransaction().commit();
    }
    void insertFoo(Foo foo) {
        em.getTransaction().begin();
        em.persist(foo);
        em.getTransaction().commit();
    }
    void removeFoo(String fooName) {
        em.getTransaction().begin();
        Foo foo = em.find(Foo.class, fooName);
        em.remove(foo);
        em.getTransaction().commit();
    }
}
```

The same code of transaction management is in every method and cannot be reused
because it contains code at both the beginning and end of the method instead of
a whole piece of reusable code. What AOP does is that it provides a way to make
our cross-cutting concern a whole piece of code so that you can reuse it easily.

## Other Terminologies

### Pointcut
The definition of a pointcut from the AspectJ
[doc](http://www.eclipse.org/aspectj/doc/next/progguide/semantics-pointcuts.html):

>A pointcut is a program element that picks out join points and exposes data
>from the execution context of those join points. Pointcuts are used primarily
>by advice. They can be composed with boolean operators to build up other
>pointcuts.

Example:
![AspectJ Pointcut Example](/img/aspectj-pointcut-example.png)

Basically pointcut is just a pattern that tells the program which are the
methods you want to have the aspect.

### Advice
The definition of a pointcut from the AspectJ
[doc](http://www.eclipse.org/aspectj/doc/released/progguide/language-anatomy.html#advice):

>A piece of advice brings together a pointcut and a body of code to define
>aspect implementation that runs at join points picked out by the pointcut.

Example:

```java
@Before("filteredTraceMethodsInDemoPackage()")
public void beforeTraceMethods(JoinPoint joinPoint) {
    // trace logic ..
}
```
An advice can be executed before, after, after returning, after throwing or
around the join point. It is just a piece of code that has the aspect logic. 

### Weaving
How AOP works is, it injects the aspect logic to your own code. And this
injection is called weaving. In AspectJ, there are three types of weaving:
compile-time weaving, post-compile weaving, and load-time weaving. (Isn't this
similar to three retention types we discussed in [previous
post](https://hechaoli.github.io/2017-10-12-java-annotation/)?)

Checkout this project for usage of all three types of weaving:
[aspectj-maven-example](https://github.com/Barlog-M/aspectj-maven-example).

#### Compile-Time Weaving
This is the simplest approach of weaving. The AspectJ compiler will compile the
source code and produce a woven class with the aspect code injected.

Using the example project above, let's see what happened to the class file after
weaving.

Before compiling:

```java
public class Foo {
    public void foo() {
        System.out.println("foo");
    }
}
```
After compile time weaving, we get Foo.class file. If we decompile the file, we
get:

```java
import org.aspectj.lang.JoinPoint;
import org.aspectj.runtime.reflect.Factory;

public class Foo {
    public Foo() {
    }

    public void foo() {
        JoinPoint var1 = Factory.makeJP(ajc$tjp_0, this, this);
        FooAspect.aspectOf().before(var1);
        System.out.println("foo");
    }

    static {
        ajc$preClinit();
    }
}
```

We see that the aspect code is injected to our own code. And this woven class
will be loaded to JVM as Foo class.

#### Post-Compile Weaving
Post-compile weaving (also sometimes called binary weaving) is used to weave
existing class files and JAR files. As with compile-time weaving, the aspects
used for weaving may be in source or binary form, and may themselves be woven by
aspects.

#### Load-Time Weaving
Load-time weaving is simply binary weaving deferred until the point that a class
loader loads a class file and defines the class to the JVM.

To support this, one or more "weaving class loaders" are required. These are
either provided explicitly by the run-time environment or enabled using a
"weaving agent".

# Reference
[https://blog.espenberntsen.net/2010/03/20/aspectj-cheat-sheet/]()
[http://www.baeldung.com/aspectj]()
