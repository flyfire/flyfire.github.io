+++
title = "Aspectj Dictionary"
date = "2016-09-13"
slug = "2016/09/13/aspectj-dictionary"
Categories = ["dev", "java"]
+++
## What is [AOP](https://en.wikipedia.org/wiki/Aspect-oriented_programming)

Every time someone tries to explain AOP (Aspect Oriented Programming) they often use words like “magic” or “black magic”. And that’s understandable, especially if you come from an OOP world and come across AOP it really feels like “magic”: something is happening and you as a developer usually don’t know why, what or how.

AspectJ is one of the more well-known implementations for AOP in Java and is developed by the Eclipse Foundation. In this series of articles we will be concentrating on AspectJ’s implementation of AOP and how it works in Java.So, let’s dive into AspectJ and make an effort of dispersing the mystic around the so-called “black magic”. Let’s start by building a dictionary with short explanations of some key terms.

<!-- more -->

## AspectJ – Dictionary

+ Aspects:The easiest way to describe aspects is as a funky Java Class. An Aspect contains other things than a normal class such as; pointcuts, advice, advice bodies and inner-type declarations. An aspect may also contain regular java classes and methods.

+ Pointcuts:Defines, in a multitude of different ways, a point in the code. The pointcut defines when an advice should be run.

+ Advice / Advice Body:Similar to a java method; contains the code that will be run once a pointcut has been triggered.

+ Annotation – Not AOP specific:Consists of meta-data and can be used at methods, classes, parameters, packages and in variables.Annotations can contain an optional list of element-value pairs,In AspectJ we can define a pointcut by looking for annotations. The pointcut and advice can then use the element-value pairs from the annotation.

+ Weaving / Aspect weaving:There are a few different ways to inject the AOP code in our application, but one common denominator is that they all require some type of extra step to be applied to our code. This extra step is called weaving.

+ Compile-time weaving:If you have both the source code of the aspect and the code that you are using aspects in, you can compile your source-code and the aspect directly with an AspectJ compiler.

+ Post-compile weaving / Binary weaving:If you can’t, or don’t want to use source-code transforms to weave the aspects into the code, you can take already compiled classes or jars and inject aspects.

+ Load-time weaving:Acts the same way as post-compile weaving / binary weaving but waits to inject aspects into the code until the class loader loads the class file. This requires one or more weaving class loaders.

## reference

+ [AspectJ - Dictionary](http://blog.jayway.com/2015/09/03/aspectj-and-aop-the-black-magic-of-programming/)
