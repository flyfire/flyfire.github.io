+++
title = "Defining Pointcuts by Annotations"
date = "2016-09-16"
slug = "2016/09/16/defining-pointcuts-by-annotations"
Categories = ["dev", "java"]
+++
## Pointcuts by annotations

Using annotations is more convenient than using patterns. While patterns might be anything between a big cannon and a scalpel the annotations are definitely a scalpel, by only getting the pointcuts that the developer has manually specified.

You can get the code for this blog series at the Git repository [here](https://github.com/Nosfert/AspectJ-Tutorial-jayway).

<!-- more -->

Let’s start coding!

Requirements: Maven, Java 8, (preferably) an IDE (I’m using IntelliJ)

The use of annotations is a precise way to define when an aspect should be run. They are only run when a developer has used the annotation on an object or method.

The errors that can and will occur

Using only annotations creates another problem that we don’t need to think about while using patterns; It will make our advice run twice(or more), because the annotation pointcut don’t specify if it should be run during execution or initialization. The reason for the advice in the pattern example not being executed twice is that the pattern uses the [combinator](https://eclipse.org/aspectj/doc/released/progguide/semantics-pointcuts.html) execute(pattern) . We are practically saying that the advice should only look for code that’s executed. There are different combinators that we can use to define when we should run our advice; one of them is execute.

So instead of only using annotations we need to use annotations and a combinator with a pattern. The simplest way is to use a catch all pattern with the combinator execution and then combine it with an annotation.

A real example:

```
package com.jayway.blog;

import org.aspectj.lang.ProceedingJoinPoint;
import org.aspectj.lang.annotation.*;
import org.aspectj.lang.JoinPoint;

@Aspect
public class YourAspect {
    //Defines a pointcut where the @YourAnnotation exists
    //And combines that with a catch all pointcut with the scope of execution
    @Around("@annotation(YourAnnotation) && execution(* *(..))")
    //ProceedingJointPoint = the reference of the call to the method.
    //Difference between ProceedingJointPoint and JointPoint is that a JointPoint can't be continued(proceeded)
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
}
```

In this example we are using the combinator execute with a catch-all pattern and annotations. Basically we are looking for any code that’s executed and has the annotation @YourAnnotation. There is a whole list of different combinators at AspectJ’s homepage. Each one of them can take different patterns to help you define when your advice should be triggered.

Implementing annotation in a class

```
package com.jayway.blog;

public class YourClass {

    public static void main(String[] args) {
        YourClass yourClass = new YourClass();
        yourClass.yourMethodAround();
    }

    @YourAnnotation
    public void yourMethodAround(){
        System.out.println("Executing TestTarget.yourMethodAround()");
    }
}
```

By adding the ``@YourAnnotation``  before any method the aspect will find the annotation and execute the advice body.

## Complexity and @Pointcut

When there is a need to define pointcuts that are a bit more complex we can define a standalone pointcut that we can reuse. By using the @Pointcut attribute we can define a specific pointcut and when it should get run. We can then use the name of the pointcut as a reference in the @Before ,  @After, @AfterThrowing,  @AfterReturn  and @Around .

A real example:

```
package com.jayway.blog;

import org.aspectj.lang.ProceedingJoinPoint;
import org.aspectj.lang.annotation.*;
import org.aspectj.lang.JoinPoint;

@Aspect
public class YourAspect {

    //Defines a pointcut that we can use in the @Before,@After, @AfterThrowing, @AfterReturning,@Around specifications
    //The pointcut will look for the @YourAnnotation
    @Pointcut("@annotation(YourAnnotation)")
    public void annotationPointCutDefinition(){
    }

    //Defines a pointcut that we can use in the @Before,@After, @AfterThrowing, @AfterReturning,@Around specifications
    //The pointcut is a catch all pointcut with the scope of execution
    @Pointcut("execution(* *(..))")
    public void atExecution(){}

    @After("annotationPointCutDefinition() && atExecution()")
    //JointPoint = the reference of the call to the method
    public void printNewLine(JoinPoint pointcut){
        //Just prints new lines after each method that's executed in
        System.out.print("\n\r");
    }
}
```

We got the same result as the earlier example but we are using the name of the pointcuts so we can reuse them.

## Conclusion

Annotations can be a precise tool, as a pointcut will not trigger if the annotations are not in the code. But there are some weak points that we need to try and cover up with the use of combinators and patterns. Combining patterns and annotations is a good middle road when we want to specify when the advice should be run.

## reference

+ [Defining pointcuts by annotations](http://blog.jayway.com/2015/09/08/defining-pointcuts-by-annotations/)
