---
layout: post
title: AOP and AspectJ Part 2
subtitle: AspectJ In Practice
tags: [Java, AOP, AspectJ]
---

In [previous post](/2017/10/18/aspectj-part1), we discussed the concept of AOP
and explained the terminologies used in AspectJ. Now we can use some examples to
show how AspectJ can be used in a project.

# Overview
In this example, we will build a trace log aspect. The idea is, for any method
that meets certain requirement (the pointcut we defined), we print a log before
and after the method. For instance, we define an annotation `@TraceLog` and a
pointcut that matches any method annotated with it. Then for the following
method: 

```java
@TraceLog
int sum(int a, int b) {
    int s = a + b;
    LOGGER.info("The sum is {}", s);
    return s;
}
```

After executing `sum(41, 42)`, we should see the following logs:

```
Calling sum(41,42)
The sum is 83
Method call sum(41,42) returns 83
```
The first and the last logs are printed by our apsect. Next we will see how we
can achieve this goal.

# Define the annotation
First we define the annotation we use on the method we want to trace. For how to
define a custom annotation, see [Java Annotation](/2017/10/12/java-annotation).

```java
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.METHOD)
public @interface TraceLog {
}
```

# Define the pointcut
Next We define a pointcut that matches methods that are annotated with
`@TraceLog`. Remember that we have to use the full name, including the full
package, of the annotation. It might be more flexible if we can use
`TraceLog.class.getName()`. However, annotation value must be a constant.

```java
public class TraceLogPointcut {

    @Pointcut("execution(@com.hechaol.example.aspectj.annotation.TraceLog * *(..))")
    public void traceLog() {
    }
}
```
This pointcut matches methods annotated with `@TraceLog` and having any return
type, any name as well as any parameters.

# Define the aspect
Now we define the aspect. Because we want to print logs both before and after
the method execution, we need to define an around aspect. We can also define two
aspects - one before and one after. But using one around should be simpler.

```java
@Aspect
public class TraceLogAspect {

    @Around("com.hechaol.example.aspectj.pointcut.TraceLogPointcut.traceLog()")
    public Object aroundMethodCall(ProceedingJoinPoint joinPoint)
        throws Throwable {
        Method method =
            ((MethodSignature) joinPoint.getSignature()).getMethod();
        String methodName = method.getName();
        String parameters = Arrays.toString(joinPoint.getArgs());
        System.out.println("Calling " + methodName + "(" + parameters + ")");
        Object returnValue;
        try {
            returnValue = joinPoint.proceed();
        } catch (Throwable throwable) {
            System.out.println("Method call "
                + methodName + "(" + parameters + ") throws: " + throwable);
            throw throwable;
        }
        System.out.println("Method call "
            + methodName + "(" + parameters + ") returns: " + returnValue);
        return returnValue;
    }
}
```

The aspect just hooks the method and print logs before and after execution.

# Define the business logic
Finally, we can define our business logic. Here we write a simple calculator.

```java
public class Calculator {

    @TraceLog
    public static int add(int a, int b) {
        return a + b;
    }

    @TraceLog
    public static int sub(int a, int b) {
        return a - b;
    }

    @TraceLog
    public static int multiply(int a, int b) {
        return a * b;
    }

    @TraceLog
    public static int divide(int a, int b) {
        return a / b;
    }
}
```

# Weave It
Remember we need to weave the aspect code to our business code? In this example
because we have the source code, we can just use compile time weaving. We need
to add a weaving plugin to maven.

```xml
<plugin>
    <groupId>org.codehaus.mojo</groupId>
    <artifactId>aspectj-maven-plugin</artifactId>
    <version>1.7</version>
    <configuration>
        <complianceLevel>${java.version}</complianceLevel>
        <source>${maven.compiler.source}</source>
        <target>${maven.compiler.target}</target>
        <showWeaveInfo>true</showWeaveInfo>
    </configuration>
    <executions>
        <execution>
            <goals>
                <goal>compile</goal>
                <goal>test-compile</goal>
            </goals>
        </execution>
    </executions>
    <dependencies>
        <dependency>
            <groupId>org.aspectj</groupId>
            <artifactId>aspectjtools</artifactId>
            <version>${aspectj.version}</version>
        </dependency>
    </dependencies>
</plugin>
```

# Run It
We can write a main class to run the code and see what is printed. For each log
we print, we can add a prefix to distinguish it from the log the aspect prints.

```java
public class Main {
    public static void main(String[] args) {
        int sum = Calculator.add(41, 42);
        System.out.println("====== The sum is: " + sum);

        System.out.println();

        int quotient = Calculator.divide(42, 0);
        System.out.println("====== The quotient is: " + quotient);
    }
}
```
After running, we should see the result as follows:

```
Calling add([41, 42])
Method call add([41, 42]) returns: 83
====== The sum is: 83

Calling divide([42, 0])
Method call divide([42, 0]) throws: java.lang.ArithmeticException: / by zero
====== Got exception: java.lang.ArithmeticException: / by zero
```

# Conclusion

In this post, we use an trace log example to show how AspectJ can be used in
a project. The entire project of this example is on
[github](https://github.com/hechaoli/aspectj-trace-log).
