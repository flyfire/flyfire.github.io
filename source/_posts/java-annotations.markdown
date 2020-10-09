---
layout: post
title: "Java Annotations"
date: 2014-09-12 12:52
comments: true
categories:
- dev
- java
tags:
- java
---
<center><p><img src="/images/java-annotations.png" width="210" height="135" alt="java"></p></center>

Ref:[Java Annotations tutorial](http://docs.oracle.com/javase/tutorial/java/annotations/),[Wiki](https://en.wikipedia.org/wiki/Java_annotation),[How do Annotations Work In Java?](http://java.dzone.com/articles/how-annotations-work-java),

Annotation is special kind of Java construct used to decorate a class, method, field, parameter, variable, constructor, or package. It’s the vehicle chosen by JSR-175 to provide metadata.Java annotations are typically used for the following purposes:

+ Compiler instructions
+ Build-time instructions
+ Runtime instructions

## Basics
Annotations have a number of uses, among them:

+ Information for the compiler — Annotations can be used by the compiler to detect errors or suppress warnings.
+ Compile-time and deployment-time processing — Software tools can process annotation information to generate code, XML files, and so forth.
+ Runtime processing — Some annotations are available to be examined at runtime.

The format of an Annotation:In its simplest form, an annotation looks like ``@Entity``,the annotation can include elements, which can be named or unnamed,if there is just one element named value, then the name can be omitted,if the annotation has no elements, then the parentheses can be omitted, as shown in the previous ``@Override`` example,it is also possible to use multiple annotations on the same declaration.If the annotations have the same type, then this is called a repeating annotation,repeating annotations are supported as of the Java SE 8 release.

The predefined annotation types defined in java.lang are ``@Deprecated``, ``@Override``, and ``@SuppressWarnings``.Note that the Javadoc tag starts with a lowercase d (``deprecated``)and the annotation starts with an uppercase D(``Deprecated``).

Every compiler warning belongs to a category. The Java Language Specification lists two categories: ``deprecation`` and ``unchecked``. The unchecked warning can occur when interfacing with legacy code written before the advent of generics. To suppress multiple categories of warnings, use the following syntax:``@SuppressWarnings({"unchecked", "deprecation"})``.

Annotations can be applied to declarations: declarations of classes, fields, methods, and other program elements. When used on a declaration, each annotation often appears, by convention, on its own line.As of the Java SE 8 release, annotations can also be applied to the use of types.

## Declaring an Annotation Type

Declaring an annotation type, syntax for doing this is:

```java
@interface ClassPreamble {
   String author();
   String date();
   int currentRevision() default 1;
   String lastModified() default "N/A";
   String lastModifiedBy() default "N/A";
   // Note use of array
   String[] reviewers();
}
```

The annotation type definition looks similar to an interface definition where the keyword interface is preceded by the at sign (@) (@ = AT, as in annotation type). Annotation types are a form of interface.The body of the previous annotation definition contains annotation type element declarations, which look a lot like methods. Note that they can define optional default values.Annotations only support primitives, string and enumerations. All attributes of annotations are defined as methods and default values can also be provided.

```java
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
@interface Todo {
    public enum Priority {LOW, MEDIUM, HIGH}
    public enum Status {STARTED, NOT_STARTED}
    String author() default "Yash";
    Priority priority() default Priority.LOW;
    Status status() default Status.NOT_STARTED;
}
```

**Note**: To make the information in @ClassPreamble appear in Javadoc-generated documentation, you must annotate the @ClassPreamble definition with the @Documented annotation:

```java
// import this to use @Documented
import java.lang.annotation.*;

@Documented
@interface ClassPreamble {

   // Annotation element definitions
   
}
```

You can also define that your annotation is a qualifier for the ``@Inject`` annotation.

```java
@javax.inject.Qualifier
@Documented
@Target({ElementType.FIELD, ElementType.PARAMETER})
@Retention(RetentionPolicy.RUNTIME)
public @interface Checker {
} 
```

<!-- more -->

## Predefined Annotation Types

### Annotation Types Used by the Java Language

Annotation types used by the Java Language include ``@Deprecated``, ``@Override``, ``@SuppressWarnings``, ``@SafeVarargs``, ``@FunctionalInterface``.

A set of annotation types are predefined in the Java SE API. Some annotation types are used by the Java compiler, and some apply to other annotations.The predefined annotation types defined in ``java.lang`` are ``@Deprecated``, ``@Override``, and ``@SuppressWarnings``.``@SuppressWarnings`` annotation tells the compiler to suppress specific warnings that it would otherwise generate.``@SafeVarargs`` annotation, when applied to a method or constructor, asserts that the code does not perform potentially unsafe operations on its varargs parameter. When this annotation type is used, unchecked warnings relating to varargs usage are suppressed.``@FunctionalInterface`` annotation, introduced in Java SE 8, indicates that the type declaration is intended to be a functional interface, as defined by the Java Language Specification.

### Annotations That Apply to Other Annotations

Annotations that apply to other Annotations include ``@Retention``, ``@Documented``, ``@Target``, ``@Inherited``, ``@Repeatable``.

``@Retention``annotation specifies how the marked annotation is stored:

+ ``RetentionPolicy.SOURCE`` – The marked annotation is retained only in the source level and is ignored by the compiler.
+ ``RetentionPolicy.CLASS`` – The marked annotation is retained by the compiler at compile time, but is ignored by the Java Virtual Machine (JVM).
+ ``RetentionPolicy.RUNTIME`` – The marked annotation is retained by the JVM so it can be used by the runtime environment.

``@Documented`` annotation indicates that whenever the specified annotation is used those elements should be documented using the Javadoc tool. (By default, annotations are not included in Javadoc.) 

``@Target`` annotation marks another annotation to restrict what kind of Java elements the annotation can be applied to. A target annotation specifies one of the following element types as its value:

+ ``ElementType.ANNOTATION_TYPE`` can be applied to an annotation type.
+ ``ElementType.CONSTRUCTOR`` can be applied to a constructor.
+ ``ElementType.FIELD`` can be applied to a field or property.
+ ``ElementType.LOCAL_VARIABLE`` can be applied to a local variable.
+ ``ElementType.METHOD`` can be applied to a method-level annotation.
+ ``ElementType.PACKAGE`` can be applied to a package declaration.
+ ``ElementType.PARAMETER`` can be applied to the parameters of a method.
+ ``ElementType.TYPE`` can be applied to any element of a class.

``@Inherited`` annotation indicates that the annotation type can be inherited from the super class. (This is not true by default.) When the user queries the annotation type and the class has no annotation for this type, the class' superclass is queried for the annotation type. This annotation applies only to class declarations.

``@Repeatable`` annotation, introduced in Java SE 8, indicates that the marked annotation can be applied more than once to the same declaration or type use.

## Type Annotations and Pluggable Type Systems

Before the Java SE 8 release, annotations could only be applied to declarations. As of the Java SE 8 release, annotations can also be applied to any type use. This means that annotations can be used anywhere you use a type. 

## Repeating Annotations

There are some situations where you want to apply the same annotation to a declaration or type use. As of the Java SE 8 release, repeating annotations enable you to do this.

For compatibility reasons, repeating annotations are stored in a ``container`` annotation that is automatically generated by the Java compiler. In order for the compiler to do this, two declarations are required in your code.

+ Declare a Repeatable Annotation Type

The annotation type must be marked with the ``@Repeatable`` meta-annotation. The following example defines a custom ``@Schedule`` repeatable annotation type:

```java
import java.lang.annotation.Repeatable;

@Repeatable(Schedules.class)
public @interface Schedule {
  String dayOfMonth() default "first";
  String dayOfWeek() default "Mon";
  int hour() default 12;
}
```

The value of the ``@Repeatable`` meta-annotation, in parentheses, is the type of the container annotation that the Java compiler generates to store repeating annotations. In this example, the containing annotation type is ``Schedules``, so repeating ``@Schedule`` annotations is stored in an ``@Schedules`` annotation.

Applying the same annotation to a declaration without first declaring it to be repeatable results in a compile-time error.

+ Declare the Containing Annotation Type

The containing annotation type must have a value element with an array type. The component type of the array type must be the repeatable annotation type. The declaration for the Schedules containing annotation type is the following:

```java
public @interface Schedules {
    Schedule[] value();
}
```

### Retrieving Annotations

There are several methods available in the Reflection API that can be used to retrieve annotations. The behavior of the methods that return a single annotation, such as ``AnnotatedElement.getAnnotationByType(Class<T>)``, are unchanged in that they only return a single annotation if one annotation of the requested type is present. If more than one annotation of the requested type is present, you can obtain them by first getting their container annotation. In this way, legacy code continues to work. Other methods were introduced in Java SE 8 that scan through the container annotation to return multiple annotations at once, such as ``AnnotatedElement.getAnnotations(Class<T>)``. See the ``AnnotatedElement`` class specification for information on all of the available methods.

If you are familiar with Reflection code, you know reflection provides ``Class``, ``Method`` and ``Field`` objects. All of these have a ``getAnnotation()`` method which returns the annotation object. We need to cast this object as our custom annotation (after checking with ``instanceOf()``) and then we can call methods defined in our custom annotation. 

```java
Class businessLogicClass = BusinessLogic.class;
for(Method method : businessLogicClass.getMethods()) {
    Todo todoAnnotation = (Todo)method.getAnnotation(Todo.class);
    if(todoAnnotation != null) {
        System.out.println(" Method Name : " + method.getName());
        System.out.println(" Author : " + todoAnnotation.author());
        System.out.println(" Priority : " + todoAnnotation.priority());
        System.out.println(" Status : " + todoAnnotation.status());
    }
}
```

### Design Considerations

When designing an annotation type, you must consider the cardinality of annotations of that type. It is now possible to use an annotation zero times, once, or, if the annotation's type is marked as ``@Repeatable``, more than once. It is also possible to restrict where an annotation type can be used by using the ``@Target`` meta-annotation. For example, you can create a repeatable annotation type that can only be used on methods and fields. It is important to design your annotation type carefully to ensure that the programmer using the annotation finds it to be as flexible and powerful as possible.
