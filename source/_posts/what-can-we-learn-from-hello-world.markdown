---
layout: post
title: "What can we learn from hello world"
date: 2014-09-01 07:57
comments: true
categories: 
- dev
- java
- c
tags:
- java
---
<center><p><img src="/images/java_logo.svg" width="150" height="275" alt="java"></p></center>

##C version

C版本的``hello,world``深入详解之前上学的时候在[SlideShare](http://www.slideshare.net/olvemaudal/deep-c)上看到过，记得当时大呼过瘾，许多C深入理解的知识基本上都被罗列了出来，现在看来仍然获益颇多。PDF版本可以在<a href="/downloads/files/DeepC0.pdf">这里和<a href="/downloads/files/DeepC1.pdf">这里</a>下载。

<!-- more -->

<iframe src="//www.slideshare.net/slideshow/embed_code/9626718" width="427" height="356" frameborder="0" marginwidth="0" marginheight="0" scrolling="no" style="border:1px solid #CCC; border-width:1px; margin-bottom:5px; max-width: 100%;" allowfullscreen> </iframe> <div style="margin-bottom:5px"> <strong> <a href="https://www.slideshare.net/olvemaudal/deep-c" title="Deep C" target="_blank">Deep C</a> </strong> from <strong><a href="http://www.slideshare.net/olvemaudal" target="_blank">Olve Maudal</a></strong> </div>
 
 Q:In C,Why dot you think static variables gets a default value(usually 0),while auto variables does not get a default value?
 
 A:Because C is all about execution speed. Setting static variables to default values is a one time cost, while defaulting auto variables might add a signficant runtime cost.Memsetting the global data segment to 0 however, is a one time cost that happens at start up.
 
 Q:Why don’t the C standard require that you always get a warning or error on invalid code?
 
 A:One of the design goals of C is that it should be relatively easy to write a compiler. Adding a requirement that the compilers should refuse or warn about invalid code would add a huge burden on the compiler writers.
 
###Memory layout
It is sometimes useful to assume that a C program uses a memory model where the instructions are stored in a **text segment**, and static variables are stored in a **data segment**. Automatic variables are allocated when needed together with housekeeping variables on an **execution stack** that is growing towards low address. The remaining memory, the **heap** is used for allocated storage.The stack and the heap is typically not cleaned up in any way at startup, or during execution, so before objects are explicitly initialized they typically get garbage values based on whatever is left in memory from discarded objects and previous executions. In other words, the programmer must do all the housekeeping on variables with automatic storage and allocated storage.And sometimes it is useful to assume that an **activation record** is created and
pushed onto the execution stack every time a function is called. The activation record
contains local auto variables, arguments to the functions, and housekeeping data such
as pointer to the previous frame and the return address.

Q:In C. Why is the evaluation order mostly unspecified?

A:Because there is a design goal to allow optimal execution speed on a wide range of architectures. In C the compiler can choose to evaluate expressions in the order that is most optimal for a particular platform. This allows for great optimization opportunities.

A sequence point is a point in the program's execution sequence where all previous side-effects shall have taken place and where all subsequent side-effects shall not have taken place.

###Sequence point in C:

+ At the end of a full expression there is a sequence point.
```c
a = i++;
++i;
if (++i == 42) { ... }
```
+ In a function call, there is a sequence point after the evaluation of the arguments, but before the actual call.
```c
foo(++i)
```
+ The logical and (&&) and logical or (||) guarantees a left-to-right evaluation, and if the second operand is evaluated, there is a sequence point between the evaluation of the first and second operands.
```c
if (p && *p++ == 42) { ... }
```
+ The comma operator (,) guarantees left-to-right evaluation and there is a sequence point between evaluating the left operand and the right operand.
```c
i = 39; a = (i++, i++, ++i);
```
+ For the conditional operator (?:), the first operand is evaluated; there is a sequence point between its evaluation and the evaluation of the second or third operand (whichever is evaluated)
```c
a++ > 42 ? --a : ++a;
```

###Behavior:

+ ``implementation-defined behavior``:the construct is not incorrect; the code must compile; the compiler must document the behavior.
+ ``unspecified behavior``: the same as implementation-defined except the behavior need not be documented
+ ``undefined behavior``: the standard imposes no requirements ; anything at all can happen, all bets are off, nasal demons might fly out of your nose.
+ ``locale-specific behavior``:the C standard defines the expected behavior, but says very little about how it should be implemented.this is a key feature of C, and one of the
reason why C is such a successful programming language on a wide range of hardware!

Integer overflow gives undefined behavior. If you want to prevent this to happen you must write the logic yourself. This is the spirit of C, you don’t get code you have not asked for.

<center><p><img src="/images/the_spirit_of_c.png" width="600" height="600" alt="the spirit of c"></p></center>

##Java version

```java
public class HelloWorld{
	public static void main(String[] args){
		System.out.println("HelloWorld.\n");
	}
}
```

Q:Why the “main” method is the program entrance and is static?

A: “static” means that the method is part of its class, not part of objects.Why is that? Why don’t we put a non-static method as program entrance?If a method is not static, then an object needs to be created first to use the method.Because the method has to be invoked on an object. For an entrance, this is not realistic. Therefore, program entrance method is static.The parameter “String[] args” indicates that an array of strings can be sent to the program to help with program initialization.

``javap -classpath . -c HelloWorld``,“javap -c” prints out disassembled code for each method in the class. Disassem-bled code means the instructions that comprise the Java bytecodes.``javap -classpath . -verbose HelloWorld`` take a further look of the class and its constant pool.

Q:How JVM loads the class and invoke the main method?

A:Before the main method is executed, JVM needs to 1) load, 2) link, and 3) initialize the class. 1) Loading brings binary form for a class/interface into JVM. 2) Linking incorporates the binary type data into the run-time state of JVM. Linking consists of 3 steps: verification, preparation, and optional resolution. Verification ensures the class/interface is structurally correct; preparation involves allocating memory needed by the class/interface; resolution resolves symbolic references. And finally 3) initialization assigns the class variables with proper initial values.

This loading job is done by Java Classloaders. When the JVM is started, three class
loaders are used:

+ Bootstrap class loader: loads the core Java libraries located in the /jre/lib directory. It is a part of core JVM, and is written in native code.
+ Extensions class loader: loads the code in the extension directories(e.g.,/jar/lib/ext).
+ System class loader: loads code found on CLASSPATH.

Finally, the main() frame is pushed into the JVM stack, and program counter(PC) is set accordingly. PC then indicates to push println() frame to the JVM stack. When the main() method completes, it will popped up from the stack and execution is done.

Ref:[When and how a java class is loaded and initialized](http://www.programcreek.com/2013/01/when-and-how-a-java-class-is-loaded-and-initialized/)
