---
layout: post
title: Java Annotation
tags: [Java, annotation]
---

# Basics
As the name implies, annotation is used to provide additional information for the elements (classes, methods,
etc) you
want to annotate. It is similar to meta data which shows properties of an element. The most common annotation should be
`@Override`, which indicates that you intend to override a method in the superclass. Some library defines its own
annotations to let you define properties. For example, in JPA, the annotation `@Entity` is used to indicate that a class
is a database model.

Following is a list of predefined Java annotations:

* **@Deprecated** [@Deprecated](https://docs.oracle.com/javase/8/docs/api/java/lang/Deprecated.html) annotation
  indicates that the marked element is deprecated and should no longer be used.
* **@Override** [@Override](https://confluence.eng.vmware.com/display/HEC/2017/10/11/Java+Annotation) annotation
  informs the compiler that the element is meant to override an element declared in a superclass.
* **@SuppressWarnings** [@SuppressWarnings](https://confluence.eng.vmware.com/display/HEC/2017/10/11/Java+Annotation)
  annotation tells the compiler to suppress specific warnings that it would otherwise generate.
* **@SafeVarargs** [@SafeVarargs](https://docs.oracle.com/javase/8/docs/api/java/lang/SafeVarargs.html) annotation,
  when applied to a method or constructor, asserts that the code does not perform potentially unsafe operations on its
  `varargs` parameter.
* **@FunctionalInterface**
  [@FunctionalInterface](https://docs.oracle.com/javase/8/docs/api/java/lang/FunctionalInterface.html)
annotation,introduced in Java SE 8, indicates that the type declaration is intended to be a functional interface, as
defined by the Java Language Specification.

Remember, annotation is "static" - it is just static information. It is used to instruct other "dynamic" module to
perform operations on the annotated elements. For example, if you have a method like this:

```java
public class Student {
    public String getStudent(@NonNull String id) {
        return "student";
    }
}
```
Even though you annotate the parameter id to be non-null, if you call the method `getStudent(null)`, nothing will
happen - No `NullPointerException` will be thrown. If you intend to throw a NPE, then you have to create a pluggable
module that helps you check the input parameter.

# Where Annotation Can Be Used

## Declarations
Annotation can be used on declarations of classes, fields and methods. For example:

```java
@Entity // Annotation on class
public class Student {
    @Id // Annotation on field
    private String id;

    private String firstName;

    private String lastName;

    @Column("full_name") // Annotation on method
    public String getFullName() {
        return firstName + " " + lastName;
    }
}
```
In a Spring controller, you can use annotation on a method parameters. This is also kind of declaration. For example:

```java
@RequestMapping(value="/student/{id}", method=RequestMethod.GET)
public Student getStudent(@PathVariable String id) { // Annotation on method parameter
    return new Student(id);
}
```

## Type Annotation (Since Java 8)
Since Java 8, annotations can be used anywhere you use a type. Type annotations were created to support improved
analysis of Java programs way of ensuring stronger type checking. Java 8 itself does not provide such a type checking
framework. But you can write your own or use an existing one. For example, see [Checker
Framework](https://checkerframework.org/) developed by University of Washington.

Examples of type annotations:

* Class instance creation expression:  
  `new @Interned MyObject();`
* Type cast:  
  `myString = (@NonNull String) str;`
* implements clause:  
  `class UnmodifiableList<T> implements
          @Readonly List<@Readonly T> {}`
* Thrown exception declaration:  
  `void monitorTemperature() throws
          @Critical TemperatureException {}`

# Create Your Own Annotation

## Definition
Similar to the annotations that 3rd party libraries provide such as `@Entity`, you can always create your own
annotations. The declaration of an annotation is similar to that of an interface. Take the pre-defined annotation
`@SuppressWarnings` for example:

```java
@Target({TYPE, FIELD, METHOD, PARAMETER, CONSTRUCTOR, LOCAL_VARIABLE})
@Retention(RetentionPolicy.SOURCE)
public @interface SuppressWarnings {
    String[] value();
}
```
We can see that the annotation type definition looks similar to an `interface` definition where the keyword interface is
preceded by the at sign '`@`'.  The body of the annotation definition contains annotation type element declarations,
which look a lot like methods. Note that they can define optional default values. The default value must be a constant
and can never be a `null` value.

Although annotation elements look like methods, they cannot have arguments, cannot define thrown exceptions and cannot
be generic because they are just more like "static" fields instead of "dynamic" methods. Additionally, the element
types are limited to:

* primitive types like `int`, `long`, `double` or `boolean`
* `String` class
* `Class` class with optional bounds
* `enum` types
* annotation types
* An array containing one of the above types

The similarity between interface and annotation is not a coincidence. In fact, annotations are visible to the JVM as
plain interfaces extending [Annotation](http://docs.oracle.com/javase/8/docs/api/java/lang/annotation/Annotation.html)
interface and the annotation elements are visible as abstract methods. It is also possible to create static fields,
static classes and enums inside an annotation. It is, however, impossible to create a
new annotation type by extending existing annotation type.

## Meta-annotations
Annotations that apply to other annotations are called meta-annotations. There are several meta-annotation types
defined in `java.lang.annotation`. When you create your own annotations, you can use the meta-annotations to instruct
how your annotation can be used.

### @Retention
[@Retention](https://docs.oracle.com/javase/8/docs/api/java/lang/annotation/Retention.html) annotation specifies how
the marked annotation is stored:

* `RetentionPolicy.SOURCE` – The marked annotation is retained only in the source level and is ignored by the compiler.
  Thus it is never stored in class files. This can be used when you use a annotation to replace comments.
* `RetentionPolicy.CLASS` – The marked annotation is retained by the compiler at compile time, but is ignored by the
  Java Virtual Machine (JVM). Thus it is in class file but not retained by JVM at runtime. I don't know what is the use
  case for this type. It might be useful if you want to do byte code-level post-processing. This is the default policy.
* `RetentionPolicy.RUNTIME` – The marked annotation is retained by the JVM so it can be used by the runtime
  environment. This can be used when you use reflection.

Example:  
Let's define an annotation annotated with `RetentionPolicy.SOURCE`

```java
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;

@Retention(RetentionPolicy.SOURCE)
public @interface Author {
    String name();
    String date();
}
```

Then we create a class annotated with `@Author`

```java
@Author(name = "Hechao Li", date = "10/12/2017")
public class Student {
    private String id;
    private String name;
}
```
Now we compile `Author.java` and `Student.java` files and get class files.

```bash
$ javac Author.java Student.java
```
Then we decompile Student.class file we get. You can use any decompiler. Here I will just open the .class file in
Intellij and it will decompile for me. The result is as follows:

```java
//
// Source code recreated from a .class file by IntelliJ IDEA
// (powered by Fernflower decompiler)
//

package com.experiment.rentention;

public class Student {
    private String id;
    private String name;

    public Student() {
    }
}
```
We can see that the annotation is not there in the decompiled file. Because the `@Author` annotation is annotated with
`RetentionPolicy.SOURCE`. You can try the other two RetentionPolicy and see the decompiled results. The annotation
should exist in the decompiled file with the other two values.

### @Documented
[@Documented](https://docs.oracle.com/javase/8/docs/api/java/lang/annotation/Documented.html) annotation indicates that
whenever the specified annotation is used those elements should be documented using the Javadoc tool. (By default,
annotations are not included in Javadoc.) For more information, see the [Javadoc
tools](https://docs.oracle.com/javase/8/docs/technotes/guides/javadoc/index.html) page.

Example:  
This time we create two annotations - `@DocumentedAuthor` and `@UnDocumentedAuthor`

```java
import java.lang.annotation.Documented;

@Documented
public @interface DocumentedAuthor {
    String name();
    String date();
}
```
```java
public @interface UnDocumentedAuthor {
    String name();
    String date();
}
```
And we add two methods to our Student class and annotate them with different annotations:

```java
public class Student {
    private String id;
    private String name;

    @UnDocumentedAuthor(name = "Hechao Li", date = "10/12/2017")
    public String getId() {
        return id;
    }

    @DocumentedAuthor(name = "Hechao Li", date = "10/12/2017")
    public String getName() {
        return name;
    }
}
```

Next we create java doc for Student class:

```bash
$ javadoc Student.java
```
Open the generated html file. We can see that only `getName()` method, which is annotated with `@DocumentedAuthor`, has
the annotation in the generated Java doc:

![Documented Java Doc](/img/documented-java-doc.png)

### @Target
[@Target](https://docs.oracle.com/javase/8/docs/api/java/lang/annotation/Target.html) annotation marks
another annotation to restrict what kind of Java elements the annotation can be applied to. A target annotation
specifies one or more of the following element types as its value:

* `ElementType.ANNOTATION_TYPE`  can be applied to an annotation type (creates meta-annotation)
* `ElementType.CONSTRUCTOR`  can be applied to a constructor
* `ElementType.FIELD`  can be applied to a field (includes enum constants)
* `ElementType.LOCAL_VARIABLE`  can be applied to a local variable
* `ElementType.METHOD` can be applied to a method
* `ElementType.PACKAGE`  can be applied to a package (placed in package-info.java file)
* `ElementType.PARAMETER`  can be applied to a method parameter
* `ElementType.TYPE`  can be applied to a type (class, interface, enum or annotation)
* `ElementType.TYPE_PARAMETER`  can be applied to a type parameter (Since Java 8)
* `ElementType.TYPE_USE`  can be applied to a use of type (Since Java 8)

Example:  
Let's define an annotation annotated with ElementType.METHOD and try to apply it on a class:

```java
import java.lang.annotation.ElementType;
import java.lang.annotation.Target;

@Target(ElementType.METHOD)
public @interface MethodAuthor {
    String name();
    String date();
}
```
```java
@MethodAuthor(name = "Hechao Li", date = "10/12/2017")
public class Student {
    private String id;
    private String name;
}
```
When we compile the code, we will get a compilation error:

```bash
$ javac *.java
Student.java:3: error: annotation type not applicable to this kind of declaration
@MethodAuthor(name = "Hechao Li", date = "10/12/2017")
^
1 error
```

### @Inherited
[@Inherited](https://docs.oracle.com/javase/8/docs/api/java/lang/annotation/Inherited.html) annotation indicates that
the annotation type can be inherited from the super class. (This is not true by default.) When the user queries the
annotation type and the class has no annotation for this type, the class' superclass is queried for the annotation
type. This annotation applies only to class declarations.

Example:  
In this example, we have an annotation annotated with @Inherited. We are going to apply this annotation to a super
class and check if this annotation is also applied to its subclass. Note, because we are going to use reflection in
this example, the annotation also has to be annotated with `@Rentation(RetentionPolicy.RUNTIME)`.

```java
import java.lang.annotation.Inherited;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;

@Inherited
@Retention(RetentionPolicy.RUNTIME)
public @interface InheritedAuthor {
    String name();
    String date();
}
```
```java
public class Main {
    @InheritedAuthor(name = "Hechao Li", date = "10/12/2017")
    public class Person {
        protected String name;
    }

    public class Student extends Person {
        private String id;
    }

    public static void main(String[] args) {
        System.out.println(Student.class.getAnnotation(InheritedAuthor.class));
    }
}
```
Execute this program and you should get result:

```bash
$ javac *.java
$ java Main
@com.experiment.inherited.InheritedAuthor(name=Hechao Li, date=10/12/2017)
```
But if you remove the `@Inherited` annotation from `@InheritedAuthor`, you should get result null.

### @Repeatable
[@Repeatable](https://docs.oracle.com/javase/8/docs/api/java/lang/annotation/Repeatable.html) annotation, introduced in
Java SE 8, indicates that the marked annotation can be applied more than once to the same declaration or type use. By
default, the annotation can be applied only once.

Example:

```java
import java.lang.annotation.Repeatable;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;

public class Main {
    @Repeatable(Authors.class)
    public @interface Author {
        String name();
        String date();
    }

    @Retention(RetentionPolicy.RUNTIME)
    public @interface Authors {
        Author[] value();
    }

    @Author(name = "Hechao Li", date = "10/12/2017")
    @Author(name = "Lan Liu", date = "10/12/2016")
    public class Student {

    }
}
```
If you remove the `@Repeatable` annotation from Author, then the code cannot compile. Note that you must also declare
the containing annotation type Authors, which has a value element with an array type and the array type is the Author.
My understanding is that compiler will convert the repeatable annotations to the following code, which is supported
even before Java 8.

```java
@Authors({
    @Author(name = "Hechao Li", date = "10/12/2017"),
    @Author(name = "Lan Liu", date = "10/12/2016")
})
public class Student {
}
```
For more information, see [https://docs.oracle.com/javase/tutorial/java/annotations/repeating.html]().

# References
* [https://docs.oracle.com/javase/tutorial/java/annotations/index.html]()
* [https://softwarecave.org/2014/05/02/custom-annotations-in-java/]()
* [https://www.developer.com/java/other/article.php/3556176/An-Introduction-to-Java-Annotations.htm]()
