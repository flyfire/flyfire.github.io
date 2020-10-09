---
layout: post
title: "Defining pointcuts by pattern"
date: 2016-09-15 16:16
comments: true
categories: 
- dev
- java
tags:
- java
---
##Pointcuts by pattern

Going through all the different ways of defining pointcuts and explaining them in an easy manner would be a near impossible feat for this post. Rather than aiming for the impossible let’s narrow down the scope, and talk about the most commonly used pointcut definitions and how we can experiment with them. We will get you started with defining your own pointcuts!

You can get the code for this blog series at the Git repository [here](https://github.com/Nosfert/AspectJ-Tutorial-jayway).

<!-- more -->

Let’s start coding!

Requirements: Maven, Java 8, (preferably) an IDE (I’m using IntelliJ)

Patterns are designed by the following elements and any element in the pattern can use wildcards (*)

**[modifiers] [return type] [(packageName)(className)(methodName)](parameters)**

**[modifiers]** can be left blank; that will be interpreted as ``*``
**[void/object/primitive type]** defined by either ``void`` , a specific object or a primitive type
**[(packageName)(className)(methodName)]** `` your.package.structure.ClassName.yourMethod``  is valid. You can add ``* ``anywhere in that line to extend the search pattern
**(parameters)** ``yourMetod(..)``  defines any method named yourMethod. The (..) is the parameter section. In this case it defines that it looks for none or any parameters.  If we replace the ``..``  with ``Integer``  we will only look for a method called ``yourMethod(Integer)``  and that has one parameter that is an ``Integer``.

A real example:

```
package com.jayway.blog;

import org.aspectj.lang.ProceedingJoinPoint;
import org.aspectj.lang.annotation.*;
import org.aspectj.lang.JoinPoint;

@Aspect
public class YourAspect {

    //Patterns
    //blank = modifier (public/private/protected or default(blank) should be looked for
    //* = return type to look for. Void/Object
    //com.jayway.YourClass.yourMethodBefore(..) = PackageName . ClassName . methodName (parameters)
    @Before("execution (* com.jayway.blog.YourClass.yourMethodBefore(..))")
    //JointPoint = the reference of the call to the method
    public void beforeAdvice(JoinPoint joinPoint) {
        System.out.println("YourAspect's BeforeAdvice's body is now executed Before yourMethodBefore is called.");
    }
    //Patterns
    //public = look for the specific modifier named public
    //!Object = Basically we are looking for void or primitives. But if we specified Object we could get a good pattern
    //com.jayway.YourClass.yourMethodBefore(..) = PackageName . ClassName . methodName (parameters)
    @After("execution (public !Object com.jayway.blog.YourClass.yourMethodAfter(..))")
    public void afterAdvice(JoinPoint joinPoint) {
        System.out.println("YourAspect's afterAdvice's body is now executed After yourMethodAfter is called.");
    }
    //Patterns
    //!private = look for any modifier thats not private
    //void = looking for method with void
    //com.jayway.YourClass.yourMethodBefore(..) = PackageName . ClassName . methodName (parameters)
    @Around("execution (!private void com.jayway.blog.YourClass.yourMethodAround(Integer,..))")
    //ProceedingJointPoint = the reference of the call to the method.
    //Difference between ProceedingJointPoint and JointPoint is that a JointPoint cant be continued(proceeded)
    //A ProceedingJointPoint can be continued(proceeded) and is needed for a Around advice
    public Object aroundAdvice(ProceedingJoinPoint joinPoint) throws Throwable {
        //Default Object that we can use to return to the consumer
        Object returnObject = null;
        try {
            System.out.println("YourAspect's aroundAdvice's body is now executed Before yourMethodAround is called.");
            //We choose to continue the call to the method in question
            returnObject = joinPoint.proceed();
            //If no exception is thrown we should land here and we can modify the returnObject, if we want to.
        } catch (Throwable throwable) {
            //Here we can catch and modify any exceptions that are called
            //We could potentially not throw the exception to the caller and instead return "null" or a default object.
            throw throwable;
        }
        finally {
            //If we want to be sure that some of our code is executed even if we get an exception
            System.out.println("YourAspect's aroundAdvice's body is now executed After yourMethodAround is called.");
        }

        return returnObject;
    }
    //Patterns//* = return type to look for. Void/Object/Primitive type
    //blank = modifier (public/private/protected or default(blank) should be looked for
    //* = return type to look for. Void/Object/Primitive type
    //com.jayway.YourClass.yourMethod*(..) = PackageName . ClassName . * (parameters)
    //Where the "*" will catch any method name
    @After("execution ( * com.jayway.blog.YourClass.*(..))")
    //JointPoint = the reference of the call to the method
    public void printNewLine(JoinPoint pointcut){
        //Just prints new lines after each method that's executed in
        System.out.print("\n\r");
    }
}
```

In this example we are using the combinator  execution([pattern])  to define when this pointcut should be triggered. There are a whole list of different [combinators](https://eclipse.org/aspectj/doc/released/progguide/semantics-pointcuts.html) at AspectJ’s homepage. Each one of them can take different patterns to help you define when your advice should be triggered.

##JointPoint and ProceedingJointPoint

A JointPoint in the code is the call to the actual (method/object) in question. The ProceedingJointPoint  extends JointPoint to add the option to continue (proceed) the call to the (method/object) in question. The JointPoint contains a lot of information (such as parameters) that can be useful when writing the advice body.

##Conclusion

Patterns can difficult – that’s why there are so many threads about how to define patterns online. Patterns gives us a lot of power, but demand that we know what we are looking for. Having the options to specify modifiers, return value and parameters gives us great tools to narrow down the complexity of our patterns.

##reference
+ [Defining pointcuts by pattern](http://blog.jayway.com/2015/09/08/defining-pointcuts-by-pattern/)





