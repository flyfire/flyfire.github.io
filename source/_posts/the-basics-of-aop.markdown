---
layout: post
title: "The basics of AOP"
date: 2016-09-14 16:15
comments: true
categories: 
- dev
- java
tags:
- java
---
##Aspect Oriented Programming

AOP is both a complex and quite simple thing. There has been a lot of buzz around AOP but recently the buzz has faded. The question that people still ask is “what do I do with it”. Hopefully you will have an idea of what it is and create your own thoughts on how you could use it.

In this post I aim to describe what AOP actually is and how it works.  

<!-- more -->

##Description

Aspect Oriented Programming circle around Aspects. An aspect contains descriptions of Joint-points and Advice bodies. The joint-points defines rules of when they should be triggered, once triggered they will execute the advice body.

An aspect acts like a middleman/proxy between the consumer of a (method/object) and the (method/object) in question. This can be achieved in five different scenarios such as; **Before, After, AfterReturn, AfterThrowing and Around** a (method/object) is called.

##The power of advice body

Once a joint-point has triggered the advice body will be executed. The advice body has tremendous power and here lies the actual code that so many people think is “magic”.

Some of the powerful things that the advice body can do are

+ modify any parameters that are passed into the call.
+ modify the return value
+ choose to not continue the call to the (method/object) in question
+ call other (methods/objects)
+ catch any exception that is thrown by the (method/object) in question

##Before, After and Around

###Before – @Before 

The before advice can only modify things before the (method/object) in question is called. The advice has the power to throw an exception and cancel the call to the (method/object) in question and modify the parameters of the call.  

<center><img src="/images/aspectj-before.png"/></center>

###After – @After , @AfterReturning  or @AfterThrowing

The after advice can only intercept the eventual return value of the (method/object) that is being called. The after advice contain three different definitions for the three different use-cases. @After  let you do things after a call and can return it’s own return value.  But @After  does not give you access to any potential return value or any potential exception that the called (method/object) creates. @AfterReturning  gives you the same abilities as @After  but gives you access to the return value. @AfterThrowing  gives you the same abilities as @After  but access to any exception that is thrown. 

<center><img src="/images/aspectj-after.png"/></center>

###Around – @Around

An around advice is like a combination of the Before and After advice. Once it has intercepted a call to the (method/object) in question it has the same control as the Before. The big difference is that when the (method/object) has run the advice gets access to any potential return value or exception. While it has access to the potential return or exception value it has the option to continue doing operations with/on that value. The advice specifies what to return to the caller. 

<center><img src="/images/aspectj-around.png"/></center>

##Conclusion

The advice can and will intercept calls to (method/object) according to the joint-point definition. The advice can specify five different situations of when they should be run. Each situation gives the advice different options to do its operations. Before, After, AfterReturn, AfterThrowing and Around are powerful in their own ways. Figuring out when to use what advice is the tricky part.

##Example

The most classical example of AOP would be logging information. Printing out information before and after a method have been called and even printing out specific exception information. By using AOP for logging we could easily create a standardized pattern that will apply to all our code.  By having the logging information in the aspect lets us modify  a few rows of code and see it take affect everywhere at the same time.  And the developers don’t need to implement their own logging for each method/class.

##reference
+ [The basics of AOP](https://blog.jayway.com/2015/09/07/the-basics-of-aop/)