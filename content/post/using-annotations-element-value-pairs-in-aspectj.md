+++
title = "Using Annotations Element Value Pairs in Aspectj"
date = "2016-09-17"
slug = "2016/09/17/using-annotations-element-value-pairs-in-aspectj"
Categories = ["dev", "java"]
+++
## Annotations with element-value pairs

Annotations by themselves are really powerful. They give direct control over when an aspect should be run to the developer. Adding element-value pairs makes the already powerful annotations even more powerful, since it enables you to pass information into the aspect.

<!-- more -->

## Description

Creating an annotation with an element-value pair is in itself quite simple. It can take different parameters and it’s up to the aspect developer to use these in their aspects. The syntax is  ``@annotation(elementValuePairs)`` and the ``elementValuePairs`` is defined by ``[keyName] = [value]``. If you want more than one ``elementValuePairs``  you use the ``,``  as a delimiter.

You can get the code for this blog series at the Git repository here.

Let’s start coding!

Requirements: Maven, Java 8, (preferably) an IDE (I’m using IntelliJ)

Adding a element-value pair is quite simple. Getting access to it from the aspect and advice body is a little bit tricky. Let’s start with a example.

```
package com.jayway.blog;

import java.lang.annotation.ElementType;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;

/**
 * Created by Steve on 2015-09-07.
 */
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.METHOD)
public @interface YourAnnotation {
    public boolean isRun() default true;
}
```

The ``@Retention``  and ``@Target``  define the scope of the annotation and you can add multiple ``@Target``  values. We have a boolean with the name ``isRun()`` with a default value of ``true`` . So let’s continue and access the ``isRun()``  from the aspect.

```
package com.jayway.blog;

import org.aspectj.lang.ProceedingJoinPoint;
import org.aspectj.lang.annotation.*;
import org.aspectj.lang.JoinPoint;

@Aspect
public class YourAspect {

    //Defines a pointcut that we can use in the @Before,@After, @AfterThrowing, @AfterReturning,@Around specifications
    //The pointcut will look for the @YourAnnotation
    @Pointcut("@annotation(yourAnnotationVariableName)")
    public void annotationPointCutDefinition(YourAnnotation yourAnnotationVariableName){
    }

    //Defines a pointcut that we can use in the @Before,@After, @AfterThrowing, @AfterReturning,@Around specifications
    //The pointcut is a catch-all pointcut with the scope of execution
    @Pointcut("execution(* *(..))")
    public void atExecution(){}

    //Defines a pointcut where the @YourAnnotation exists
    //and combines that with an catch-all pointcut with the scope of execution
    @Around("annotationPointCutDefinition(yourAnnotationVariableName) && atExecution()")
    //ProceedingJointPoint = the reference of the call to the method.
    //The difference between ProceedingJointPoint and JointPoint is that a JointPoint can't be continued (proceeded)
    //A ProceedingJointPoint can be continued (proceeded) and is needed for an Around advice
    public Object aroundAdvice(ProceedingJoinPoint joinPoint, YourAnnotation yourAnnotationVariableName) throws Throwable {
        if(yourAnnotationVariableName.isRun()) {
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
            } finally {
                //If we want to be sure that some of our code is executed even if we get an exception
                System.out.println("YourAspect's aroundAdvice's body is now executed After yourMethodAround is called.");
            }
            return returnObject;
        }
        else{
            return joinPoint.proceed();
        }
    }

    @After("annotationPointCutDefinition(yourAnnotationVariableName) && atExecution()")
    //JointPoint = the reference of the call to the method
    public void printNewLine(JoinPoint pointcut, YourAnnotation yourAnnotationVariableName){
        //Just prints new lines after each method that's executed in
        System.out.print("\n\r");
    }
}
```

We specify that the  ``annotationPointCutDefinition(YourAnnotation yourAnnotationVariableName)`` contains a ``YourAnnotation`` reference. The variable reference is then used in the ``@Pointcut("@annotation(yourAnnotationVariableName")``  instead of the generic ``YourAnnotation`` that were used in the earlier examples. By passing the variable to the ``@annotation`` we gain access to it. We need to pass the variable into the ``@Around``  advice to gain access to the variable in the advice body.  Once in the advice body we can use it like a normal method parameter.

```
package com.jayway.blog;

public class YourClass {

    public static void main(String[] args) {
        YourClass yourClass = new YourClass();
        yourClass.yourMethodAroundDontRun();
        yourClass.yourMethodAroundRunTrue();
        yourClass.yourMethodAroundRun();
    }

    @YourAnnotation(isRun = false)
    public void yourMethodAroundDontRun(){
        System.out.println("Executing TestTarget.yourMethodAroundDontRun()");
    }

    @YourAnnotation(isRun = true)
    public void yourMethodAroundRunTrue(){
        System.out.println("Executing TestTarget.yourMethodAroundRunTrue()");
    }

    @YourAnnotation()
    public void yourMethodAroundRun(){
        System.out.println("Executing TestTarget.yourMethodAroundRun()");
    }
}
```

We add the annotation to our methods and have the option to omit the parameter or directly specify the value of the parameter. If the parameter is omitted the parameter will get it’s default value specified in the annotation.

## Conclusion

Annotations with element-value pairs can be really powerful for both the developer who is using the aspects and as the one developing them. It gives the ability to send parameters into the aspect code. By adding the ability to send parameters we open up the code to be more configurable. By having aspects that are configurable we open up the code to be separated into it’s own independent framework or library.

## reference

+ [Using annotations element-value pairs in AspectJ ](http://blog.jayway.com/2015/09/09/using-annotations-element-value-pairs-in-aspectj/)

