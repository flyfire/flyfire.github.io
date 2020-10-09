---
layout: post
title: "Python Tutorial"
date: 2015-03-29 11:17
comments: true
categories: 
- dev
- python
tags:
- python
---
<center><p><img src="/images/python_logo.png"></p></center>
*   [Whetting Your Appetite](#1)
*   [Using the Python Interpreter](#2)
    *   [Invoking the Interpreter](#2.1)
        *   [Argument Passing](#2.1.1)
        *   [Interactive Mode](#2.1.2)
    *   [The Interpreter and Its Environment](#2.2)
        *   [Source Code Encoding](#2.2.1)
*   [An Informal Introduction to Python](#3)
    *   [Using Python as Calculator](#3.1)
        *   [Numbers](#3.1.1)
        *   [Strings](#3.1.2)
        *   [Lists](#3.1.3)
    *   [First Steps Towards Programming](#3.2)
*   [More Control Flow Tools](#4)
    *   [if Statements](#4.1)
    *   [for Statements](#4.2)
    *   [The range() Function](#4.3)
    *   [break and continue Statements,and else Clauses and Loops](#4.4)
    *   [pass Statements](#4.5)
    *   [Defining Functions](#4.6)
    *   [More on Defining Functions](#4.7)
        *   [Default Argument Values](#4.7.1)
        *   [Keyword Arguments](#4.7.2)
        *   [Arbitrary Argument Lists](#4.7.3)
        *   [Unpacking Argument Lists](#4.7.4)
        *   [Lambda Expressions](#4.7.5)
        *   [Documentation Strings](#4.7.6)
        *   [Function Annotations](#4.7.7)
    *   [Intermezzo: Coding Style](#4.8)
*   [Data Structures](#5)
    *   [More on Lists](#5.1)
        *   [Using Lists as Stacks](#5.1.1)
        *   [Using Lists as Queues](#5.1.2)
        *   [List Comprehensions](#5.1.3)
        *   [Nested List Comprehensions](#5.1.4)
    *   [The del Statement](#.5.2)
    *   [Tuples and Sequences](#5.3)
    *   [Sets](#5.4)
    *   [Dictionaries](#5.5)
    *   [Looping Techniques](#5.6)
    *   [More on Conditions](#5.7)
    *   [Comparing Sequences and Other Types](#5.8)
*   [Modules](#6)
    *   [More on Modules](#6.1)
        *   [Executing modules as scripts](#6.1.1)
        *   [The Module Search Path](#6.1.2)
        *   ["Compiled" Python files](#6.1.3)
    *   [Standard Modules](#6.2)
    *   [The dir() Function](#6.3)
    *   [Packages](#6.4)
        *   [Importing * From a Package](#6.4.1)
        *   [Intra-package References](#6.4.2)
        *   [Packages in Multiple Directories](#6.4.3)
*   [Input and Output](#7)
    *   [Fancier Output Formatting](#7.1)
        *   [Old String Formatting](#7.1.1)
    *   [Reading and Writing Files](#7.2)
        *   [Methods of File Objects](#7.2.1)
        *   [Saving structured data with json](#7.2.2)
*   [Errors and Exceptions](#8)
    *   [Syntax Errors](#8.1)
    *   [Exceptions](#8.2)
    *   [Handling Exceptions](#8.3)
    *   [Raising Exceptions](#8.4)
    *   [User-defined Exceptions](#8.5)
    *   [Defining Clean-up Actions](#8.6)
    *   [Predefined Clean-up Actions](#8.7)
*   [Classes](#9)
    *   [A Word About Names and Objects](#9.1)
    *   [Python Scopes and Namespaces](#9.2)
        *   [Scopes and Namespacs Example](#9.2.1)
    *   [A First Look at Classes](#9.3)
        *   [Class Definition Syntax](#9.3.1)
        *   [Class Objects](#9.3.2)
        *   [Instance Objects](#9.3.3)
        *   [Method Objects](#9.3.4)
        *   [Class and Instance Variables](#9.3.5)
    *   [Random Remarks](#9.4)
    *   [Inheritance](#9.5)
        *   [Multiple Inheritance](#9.5.1)
    *   [Private Variables](#9.6)
    *   [Odds and Ends](#9.7)
    *   [Exceptions Are Classes Too](#9.8)
    *   [Iterators](#9.9)
    *   [Generators](#9.10)
    *   [Generator Expressions](#9.11)
*   [Brief Tour of the Standard Library](#10)
    *   [Operating System Interface](#10.1)
    *   [File Wildcards](#10.2)
    *   [Command Line Arguments](#10.3)
    *   [Error Output Redirection and Program Termination](#10.4)
    *   [String Pattern Matching](#10.5)
    *   [Mathematics](#10.6)
    *   [Internet Access](#10.7)
    *   [Dates and Times](#10.8)
    *   [Data Compression](#10.9)
    *   [Performance Measurement](#10.10)
    *   [Quality Control](#10.11)
    *   [Batteries Included](#10.12)
*   [Brief Tour of the Standard Library - Part II](#11)
    *   [Output Formatting](#11.1)
    *   [Templating](#11.2)
    *   [Working with Binary Data Record Layouts](#11.3)
    *   [Multi-threading](#11.4)
    *   [Logging](#11.5)
    *   [Weak References](#11.6)
    *   [Tools for Working with Lists](#11.7)
    *   [Decimal Floating Point Arithmetic](#11.8)

<!-- more -->
<h2 id="1">Whetting Your Appetite</h2>
<h2 id="2">Using the Python Interpreter</h2>
<h3 id="2.1">Invoking the Interpreter</h3>

``python -c command [arg]``,``python -m module [arg]``

<h4 id="2.1.1">Argument Passing</h4>

When known to the interpreter, the script name and additional arguments thereafter are turned into a list of strings and assigned to the argv variable in the sys module. You can access this list by executing ``import sys``.

<h4 id="2.1.2">Interactive Mode</h4>

<h3 id="2.2">The Interpreter and Its Environment</h3>
<h4 id="2.2.1">Source Code Encoding</h4>

It is also possible to specify a different encoding for source files. In order to do this, put one more special comment line right after the ``#!`` line to define the source file encoding:``# -*- coding: encoding -*-``.

<h2 id="3">An Informal Introduction to Python</h2>
<h3 id="3.1">Using Python as a Calculator</h3>
<h4 id="3.1.1">Numbers</h4>

Division (``/``) always returns a float. To do floor division and get an integer result (discarding any fractional result) you can use the ``//`` operator; to calculate the remainder you can use ``%``.With Python, it is possible to use the ``**`` operator to calculate powers.

In interactive mode, the last printed expression is assigned to the variable ``_``.This variable should be treated as read-only by the user. Don’t explicitly assign a value to it — you would create an independent local variable with the same name masking the built-in variable with its magic behavior.

<h4 id="3.1.2">Strings</h4>

Besides numbers, Python can also manipulate strings, which can be expressed in several ways. They can be enclosed in single quotes (``'...'``) or double quotes (``"..."``) with the same result. ``\`` can be used to escape quotes.

If you don’t want characters prefaced by ``\`` to be interpreted as special characters, you can use raw strings by adding an ``r`` before the first quote.``print(r'C:\some\name')``.

String literals can span multiple lines. One way is using triple-quotes: ``"""..."""`` or ``'''...'''``. End of lines are automatically included in the string, but it’s possible to prevent this by adding a ``\`` at the end of the line. 

```python
print("""\
Usage: thingy [OPTIONS]
     -h                        Display this usage message
     -H hostname               Hostname to connect to
""")
```

Strings can be concatenated (glued together) with the ``+`` operator, and repeated with ``*``.Two or more string literals (i.e. the ones enclosed between quotes) next to each other are automatically concatenated.This only works with two literals though, not with variables or expressions.If you want to concatenate variables or a variable and a literal, use ``+``.

Strings can be indexed (subscripted), with the first character having index 0. There is no separate character type; a character is simply a string of size one.Indices may also be negative numbers, to start counting from the right.Note that since -0 is the same as 0, negative indices start from -1.

In addition to indexing, slicing is also supported. While indexing is used to obtain individual characters, slicing allows you to obtain substring.Note how the start is always included, and the end always excluded. This makes sure that ``s[:i] + s[i:]`` is always equal to ``s``.

Slice indices have useful defaults; an omitted first index defaults to zero, an omitted second index defaults to the size of the string being sliced.

One way to remember how slices work is to think of the indices as pointing between characters, with the left edge of the first character numbered 0. Then the right edge of the last character of a string of n characters has index n, for example:

```bash
 +---+---+---+---+---+---+
 | P | y | t | h | o | n |
 +---+---+---+---+---+---+
 0   1   2   3   4   5   6
-6  -5  -4  -3  -2  -1
```

Python strings cannot be changed — they are **immutable**. Therefore, assigning to an indexed position in the string results in an error.

<h4 id="3.1.3">Lists</h4>

Like strings (and all other built-in sequence type), lists can be indexed and sliced.

All slice operations return a new list containing the requested elements. This means that the following slice returns a new (shallow) copy of the list:

```python
>>> squares[:]
[1, 4, 9, 16, 25]
```

Lists also support operations like concatenation:

```python
>>> squares + [36, 49, 64, 81, 100]
[1, 4, 9, 16, 25, 36, 49, 64, 81, 100]
```

Unlike strings, which are immutable, lists are a mutable type, i.e. it is possible to change their content.You can also add new items at the end of the list, by using the ``append()`` method.

Assignment to slices is also possible, and this can even change the size of the list or clear it entirely:

```python
>>> letters = ['a', 'b', 'c', 'd', 'e', 'f', 'g']
>>> letters
['a', 'b', 'c', 'd', 'e', 'f', 'g']
>>> # replace some values
>>> letters[2:5] = ['C', 'D', 'E']
>>> letters
['a', 'b', 'C', 'D', 'E', 'f', 'g']
>>> # now remove them
>>> letters[2:5] = []
>>> letters
['a', 'b', 'f', 'g']
>>> # clear the list by replacing all the elements with an empty list
>>> letters[:] = []
>>> letters
[]
```

It is possible to nest lists (create lists containing other lists).

<h3 id="3.2">First Steps Towards Programming</h3>

In Python, like in C, any non-zero integer value is true; zero is false. The condition may also be a string or list value, in fact any sequence; anything with a non-zero length is true, empty sequences are false. 

```python
>>> print('The value of i is', i)
The value of i is 65536
```

The keyword argument end can be used to avoid the newline after the output, or end the output with a different string.

```python
>>> a, b = 0, 1
>>> while b < 1000:
...     print(b, end=',')
...     a, b = b, a+b
...
1,1,2,3,5,8,13,21,34,55,89,144,233,377,610,987,
```

<h2 id="4">More Control Flow Tools</h2>
<h3 id="4.1">if Statements</h3>
<h3 id="4.2">for Statements</h3>

The for statement in Python differs a bit from what you may be used to in C or Pascal. Rather than always iterating over an arithmetic progression of numbers (like in Pascal), or giving the user the ability to define both the iteration step and halting condition (as C), Python’s for statement iterates over the items of any sequence (a list or a string), in the order that they appear in the sequence. 

If you need to modify the sequence you are iterating over while inside the loop (for example to duplicate selected items), it is recommended that you first make a copy. Iterating over a sequence does not implicitly make a copy. The slice notation makes this especially convenient:

```python
>>> for w in words[:]:  # Loop over a slice copy of the entire list.
...     if len(w) > 6:
...         words.insert(0, w)
...
>>> words
['defenestrate', 'cat', 'window', 'defenestrate']
```

<h3 id="4.3">The range() Function</h3>

If you do need to iterate over a sequence of numbers, the built-in function ``range()`` comes in handy. It generates arithmetic progressions.

```python
range(-10, -100, -30)
  -10, -40, -70
```

```python
>>> print(range(10))
range(0, 10)
```

In many ways the object returned by ``range()`` behaves as if it is a list, but in fact it isn’t. It is an object which returns the successive items of the desired sequence when you iterate over it, but it doesn’t really make the list, thus saving space.We say such an object is iterable, that is, suitable as a target for functions and constructs that expect something from which they can obtain successive items until the supply is exhausted.

<h3 id="4.4">break and continue Statements, and else Clauses on Loops</h3>

When used with a loop, the else clause has more in common with the else clause of a try statement than it does that of if statements: **a try statement’s else clause runs when no exception occurs, and a loop’s else clause runs when no break occurs**. 

<h3 id="4.5">pass Statements</h3>

The ``pass`` statement does nothing. It can be used when a statement is required syntactically but the program requires no action. 

```python
>>> while True:
...     pass  # Busy-wait for keyboard interrupt (Ctrl+C)
...
```

<h3 id="4.6">Defining Functions</h3>

The keyword ``def`` introduces a function definition. It must be followed by the function name and the parenthesized list of formal parameters. The statements that form the body of the function start at the next line, and must be indented.

The execution of a function introduces a new symbol table used for the local variables of the function. More precisely, all variable assignments in a function store the value in the local symbol table; whereas variable references first look in the local symbol table, then in the local symbol tables of enclosing functions, then in the global symbol table, and finally in the table of built-in names. Thus, global variables cannot be directly assigned a value within a function (unless named in a global statement), although they may be referenced.

A function definition introduces the function name in the current symbol table. The value of the function name has a type that is recognized by the interpreter as a user-defined function. This value can be assigned to another name which can then also be used as a function.

In fact, even functions without a return statement do return a value, albeit a rather boring one. This value is called None (it’s a built-in name). Writing the value None is normally suppressed by the interpreter if it would be the only value written. 

<h3 id="4.7">More on Defining Functions</h3>

<h4 id="4.7.1">Default Argument Values</h4>

```python
def ask_ok(prompt, retries=4, complaint='Yes or no, please!'):
    while True:
        ok = input(prompt)
        if ok in ('y', 'ye', 'yes'):
            return True
        if ok in ('n', 'no', 'nop', 'nope'):
            return False
        retries = retries - 1
        if retries < 0:
            raise OSError('uncooperative user')
        print(complaint)
```

The default values are evaluated at the point of function definition in the defining scope, so that

```python
i = 5

def f(arg=i):
    print(arg)

i = 6
f()
```

**Important warning**: The default value is evaluated only once. This makes a difference when the default is a mutable object such as a list, dictionary, or instances of most classes. For example, the following function accumulates the arguments passed to it on subsequent calls:

```python
def f(a, L=[]):
    L.append(a)
    return L

print(f(1))
print(f(2))
print(f(3))
```

This will print

```python
[1]
[1, 2]
[1, 2, 3]
```

If you don’t want the default to be shared between subsequent calls, you can write the function like this instead:

```python
def f(a, L=None):
    if L is None:
        L = []
    L.append(a)
    return L
```

<h4 id="4.7.2">Keyword Arguments</h4>

Functions can also be called using keyword arguments of the form kwarg=value. For instance, the following function:

```python
def parrot(voltage, state='a stiff', action='voom', type='Norwegian Blue'):
    print("-- This parrot wouldn't", action, end=' ')
    print("if you put", voltage, "volts through it.")
    print("-- Lovely plumage, the", type)
    print("-- It's", state, "!")
```

accepts one required argument (voltage) and three optional arguments (state, action, and type). This function can be called in any of the following ways:

```python
parrot(1000)                                          # 1 positional argument
parrot(voltage=1000)                                  # 1 keyword argument
parrot(voltage=1000000, action='VOOOOOM')             # 2 keyword arguments
parrot(action='VOOOOOM', voltage=1000000)             # 2 keyword arguments
parrot('a million', 'bereft of life', 'jump')         # 3 positional arguments
parrot('a thousand', state='pushing up the daisies')  # 1 positional, 1 keyword
```

When a final formal parameter of the form ``**name`` is present, it receives a dictionary containing all keyword arguments except for those corresponding to a formal parameter. This may be combined with a formal parameter of the form ``*name`` (described in the next subsection) which receives a tuple containing the positional arguments beyond the formal parameter list. (``*name`` must occur before ``**name``.) 

<h4 id="4.7.3">Arbitrary Argument Lists</h4>

Finally, the least frequently used option is to specify that a function can be called with an arbitrary number of arguments. These arguments will be wrapped up in a tuple.

<h4 id="4.7.4">Unpacking Argument Lists</h4>

The reverse situation occurs when the arguments are already in a list or tuple but need to be unpacked for a function call requiring separate positional arguments. For instance, the built-in ``range()`` function expects separate start and stop arguments. If they are not available separately, write the function call with the *-operator to unpack the arguments out of a list or tuple:

```python
>>> list(range(3, 6))            # normal call with separate arguments
[3, 4, 5]
>>> args = [3, 6]
>>> list(range(*args))            # call with arguments unpacked from a list
[3, 4, 5]
```

In the same fashion, dictionaries can deliver keyword arguments with the **-operator.

<h4 id="4.7.5">Lambda Expressions</h4>

Small anonymous functions can be created with the lambda keyword. This function returns the sum of its two arguments: lambda a, b: a+b. Lambda functions can be used wherever function objects are required. They are syntactically restricted to a single expression. Semantically, they are just syntactic sugar for a normal function definition. Like nested function definitions, lambda functions can reference variables from the containing scope:

```python
>>> def make_incrementor(n):
...     return lambda x: x + n
...
>>> f = make_incrementor(42)
>>> f(0)
42
>>> f(1)
43
```

The above example uses a lambda expression to return a function. Another use is to pass a small function as an argument(++++++question mark++++++):

```python
>>> pairs = [(1, 'one'), (2, 'two'), (3, 'three'), (4, 'four')]
>>> pairs.sort(key=lambda pair: pair[1])
>>> pairs
[(4, 'four'), (1, 'one'), (3, 'three'), (2, 'two')]
```

<h4 id="4.7.6">Documentation Strings</h4>
<h4 id="4.7.7">Function Annotations</h4>

Annotations are stored in the __annotations__ attribute of the function as a dictionary and have no effect on any other part of the function. Parameter annotations are defined by a colon after the parameter name, followed by an expression evaluating to the value of the annotation. Return annotations are defined by a literal ->, followed by an expression, between the parameter list and the colon denoting the end of the def statement. The following example has a positional argument, a keyword argument, and the return value annotated with nonsense:

```python
>>> def f(ham: 42, eggs: int = 'spam') -> "Nothing to see here":
...     print("Annotations:", f.__annotations__)
...     print("Arguments:", ham, eggs)
...
>>> f('wonderful')
Annotations: {'eggs': <class 'int'>, 'return': 'Nothing to see here', 'ham': 42}
Arguments: wonderful spam
```

<h3 id="4.8">Intermezzo: Coding Style</h3>

[PEP8](http://www.python.org/dev/peps/pep-0008)

+ Use 4-space indentation, and no tabs.
+ Wrap lines so that they don’t exceed 79 characters.
+ Use blank lines to separate functions and classes, and larger blocks of code inside functions.
+ When possible, put comments on a line of their own.
+ Use docstrings.
+ Use spaces around operators and after commas, but not directly inside bracketing constructs
+ Name your classes and functions consistently; the convention is to use CamelCase for classes and lower_case_with_underscores for functions and methods. Always use self as the name for the first method argument.

<h2 id="5">Data Structures</h2>

<h3 id="5.1">More on Lists</h3>
+ ``list.append(x)``,Equivalent to ``a[len(a):] = [x]``
+ ``list.extend(L)``,Extend the list by appending all the items in the given list. Equivalent to ``a[len(a):] = L``
+ ``list.insert(i, x)``,Insert an item at a given position. The first argument is the index of the element before which to insert, so ``a.insert(0, x)`` inserts at the front of the list, and ``a.insert(len(a), x)`` is equivalent to ``a.append(x)``.
+ ``list.remove(x)``,Remove the first item from the list whose value is x. It is an error if there is no such item.
+ ``list.pop([i])``,the square brackets around i indicates that i is optional,Remove the item at the given position in the list, and return it. If no index is specified, ``a.pop()`` removes and returns the last item in the list.
+ ``list.clear()``,Remove all items from the list. Equivalent to del a[:].
+ ``list.index(x)``,Return the index in the list of the first item whose value is x. It is an error if there is no such item.
+ ``list.count(x)``,Return the number of times x appears in the list.
+ ``list.sort()``,Sort the items of the list in place.
+ ``list.reverse()``,Reverse the elements of the list in place.
+ ``list.copy()``,Return a shallow copy of the list. Equivalent to a[:].

Methods like insert, remove or sort that only modify the list have no return value printed – they return the default None. This is a design principle for all mutable data structures in Python.

<h4 id="5.1.1">Using Lists as Stacks</h4>

``append/pop``

<h4 id="5.1.2">Using Lists as Queues</h4>

```python
from collections import dequeue
queue = dequeue(["eric", "john"])
append/popLeft
```

<h4 id="5.1.3">List Comprehensions</h4>

```python
squares = list(map(lambda x: x**2, range(10)))
squares = [x**2 for x in range(10)]
>>> [(x, y) for x in [1,2,3] for y in [3,1,4] if x != y]
[(1, 3), (1, 4), (2, 3), (2, 1), (2, 4), (3, 1), (3, 4)]
```

A list comprehension consists of brackets containing an expression followed by a for clause, then zero or more for or if clauses. The result will be a new list resulting from evaluating the expression in the context of the for and if clauses which follow it. If the expression is a tuple, it must be parenthesized.

<h4 id="5.1.4">Nested List Comprehensions</h4>

```python
>>> matrix = [
...     [1, 2, 3, 4],
...     [5, 6, 7, 8],
...     [9, 10, 11, 12],
... ]
>>> [[row[i] for row in matrix] for i in range(4)]
[[1, 5, 9], [2, 6, 10], [3, 7, 11], [4, 8, 12]]

>>> transposed = []
>>> for i in range(4):
...     transposed.append([row[i] for row in matrix])
...
>>> transposed
[[1, 5, 9], [2, 6, 10], [3, 7, 11], [4, 8, 12]]

>>> transposed = []
>>> for i in range(4):
...     # the following 3 lines implement the nested listcomp
...     transposed_row = []
...     for row in matrix:
...         transposed_row.append(row[i])
...     transposed.append(transposed_row)
...
>>> transposed
[[1, 5, 9], [2, 6, 10], [3, 7, 11], [4, 8, 12]]

>>> list(zip(*matrix))
[(1, 5, 9), (2, 6, 10), (3, 7, 11), (4, 8, 12)]
```

<h3 id="5.2">The del statement</h3>

There is a way to remove an item from a list given its index instead of its value: the ``del`` statement. This differs from the ``pop()`` method which returns a value.del can also be used to delete entire variables:``del a``,Referencing the name a hereafter is an error (at least until another value is assigned to it).

<h3 id="5.3">Tuples and Sequences</h3>

A tuple consists of a number of values separated by commas, tuples may be nested,tuples are immutable.

<h3 id="5.4">Sets</h3>

A set is an unordered collection with no duplicate elements. Basic uses include membership testing and eliminating duplicate entries. Set objects also support mathematical operations like union, intersection, difference, and symmetric difference.Curly braces or the ``set()`` function can be used to create sets. Note: to create an empty set you have to use ``set()``, not ``{}``; the latter creates an empty dictionary.

<h3 id="5.5">Dictionaries</h3>

Dictionaries are sometimes found in other languages as “associative memories” or “associative arrays”. Unlike sequences, which are indexed by a range of numbers, dictionaries are indexed by keys, which can be any immutable type; strings and numbers can always be keys. Tuples can be used as keys if they contain only strings, numbers, or tuples; if a tuple contains any mutable object either directly or indirectly, it cannot be used as a key. You can’t use lists as keys, since lists can be modified in place using index assignments, slice assignments, or methods like append() and extend().It is best to think of a dictionary as an unordered set of key: value pairs, with the requirement that the keys are unique (within one dictionary). A pair of braces creates an empty dictionary: {}. Placing a comma-separated list of key:value pairs within the braces adds initial key:value pairs to the dictionary; this is also the way dictionaries are written on output.The main operations on a dictionary are storing a value with some key and extracting the value given the key. It is also possible to delete a key:value pair with del. If you store using a key that is already in use, the old value associated with that key is forgotten. It is an error to extract a value using a non-existent key.

<h3 id="5.6">Looping Techniques</h3>

When looping through dictionaries, the key and corresponding value can be retrieved at the same time using the ``items()`` method.

When looping through a sequence, the position index and corresponding value can be retrieved at the same time using the ``enumerate()`` function.

To loop over two or more sequences at the same time, the entries can be paired with the zip() function.

```python
>>> questions = ['name', 'quest', 'favorite color']
>>> answers = ['lancelot', 'the holy grail', 'blue']
>>> for q, a in zip(questions, answers):
...     print('What is your {0}?  It is {1}.'.format(q, a))
...
What is your name?  It is lancelot.
What is your quest?  It is the holy grail.
What is your favorite color?  It is blue.
```

To loop over a sequence in reverse, first specify the sequence in a forward direction and then call the ``reversed()`` function.To loop over a sequence in sorted order, use the ``sorted()`` function which returns a new sorted list while leaving the source unaltered.

To change a sequence you are iterating over while inside the loop (for example to duplicate certain items), it is recommended that you first make a copy. Looping over a sequence does not implicitly make a copy. The slice notation makes this especially convenient:

```python
>>> words = ['cat', 'window', 'defenestrate']
>>> for w in words[:]:  # Loop over a slice copy of the entire list.
...     if len(w) > 6:
...         words.insert(0, w)
...
>>> words
['defenestrate', 'cat', 'window', 'defenestrate']
```

<h3 id="5.7">More on Conditions</h3>

The conditions used in while and if statements can contain any operators, not just comparisons.

The comparison operators ``in`` and ``not in`` check whether a value occurs (does not occur) in a sequence. The operators ``is`` and ``is not`` compare whether two objects are really the same object; this only matters for mutable objects like lists. All comparison operators have the same priority, which is lower than that of all numerical operators.

Comparisons can be chained. For example, ``a < b == c`` tests whether a is less than b and moreover b equals c.

Comparisons may be combined using the Boolean operators ``and`` and ``or``, and the outcome of a comparison (or of any other Boolean expression) may be negated with ``not``. These have lower priorities than comparison operators; between them, not has the highest priority and or the lowest, so that A and not B or C is equivalent to (A and (not B)) or C. As always, parentheses can be used to express the desired composition.

<h3 id="5.8">Comparing Sequences and Other Types</h3>

Sequence objects may be compared to other objects with the same sequence type. The comparison uses lexicographical ordering: first the first two items are compared, and if they differ this determines the outcome of the comparison; if they are equal, the next two items are compared, and so on, until either sequence is exhausted. If two items to be compared are themselves sequences of the same type, the lexicographical comparison is carried out recursively. If all items of two sequences compare equal, the sequences are considered equal. If one sequence is an initial sub-sequence of the other, the shorter sequence is the smaller (lesser) one. Lexicographical ordering for strings uses the Unicode code point number to order individual characters. 

<h2 id="6">Modules</h2>

Definitions from a module can be imported into other modules or into the main module (the collection of variables that you have access to in a script executed at the top level and in calculator mode).A module is a file containing Python definitions and statements. The file name is the module name with the suffix ``.py`` appended. Within a module, the module’s name (as a string) is available as the value of the global variable ``__name__``.

<h3 id="6.1">More on Modules</h3>

Each module has its own private symbol table, which is used as the global symbol table by all functions defined in the module. Thus, the author of a module can use global variables in the module without worrying about accidental clashes with a user’s global variables. On the other hand, if you know what you are doing you can touch a module’s global variables with the same notation used to refer to its functions, ``modname.itemname``.

Modules can import other modules. It is customary but not required to place all import statements at the beginning of a module (or script, for that matter). The imported module names are placed in the importing module’s global symbol table.There is a variant of the import statement that imports names from a module directly into the importing module’s symbol table. ``from fibo import fib, fib2``,This does not introduce the module name from which the imports are taken in the local symbol table (so in the example, ``fibo`` is not defined).There is even a variant to import all names that a module defines:``from fibo import *``,This imports all names except those beginning with an underscore (``_``). In most cases Python programmers do not use this facility since it introduces an unknown set of names into the interpreter, possibly hiding some things you have already defined.

For efficiency reasons, each module is only imported once per interpreter session. Therefore, if you change your modules, you must restart the interpreter – or, if it’s just one module you want to test interactively, use ``imp.reload()``, e.g. ``import imp; imp.reload(modulename)``.

<h4 id="6.1.1">Executing modules as scripts</h4>

When you run a Python module with ``python fibo.py <arguments>``,the code in the module will be executed, just as if you imported it, but with the ``__name__`` set to ``__main__``. That means that by adding this code at the end of your module:

```python
if __name__ == "__main__":
    import sys
    fib(int(sys.argv[1]))
```

you can make the file usable as a script as well as an importable module, because the code that parses the command line only runs if the module is executed as the “main” file:``$ python fibo.py 50``.

<h4 id="6.1.2">The Module Search Path</h4>

When a module named spam is imported, the interpreter first searches for a built-in module with that name. If not found, it then searches for a file named ``spam.py`` in a list of directories given by the variable ``sys.path``. ``sys.path`` is initialized from these locations:

+ The directory containing the input script (or the current directory when no file is specified).
+ ``PYTHONPATH`` (a list of directory names, with the same syntax as the shell variable PATH).
+ The installation-dependent default.

After initialization, Python programs can modify sys.path. The directory containing the script being run is placed at the beginning of the search path, ahead of the standard library path. This means that scripts in that directory will be loaded instead of modules of the same name in the library directory. This is an error unless the replacement is intended.

<h4 id="6.1.3">“Compiled” Python files</h4>

To speed up loading modules, Python caches the compiled version of each module in the ``__pycache__`` directory under the name ``module.version.pyc``, where the version encodes the format of the compiled file; it generally contains the Python version number. For example, in CPython release 3.3 the compiled version of ``spam.py`` would be cached as ``__pycache__/spam.cpython-33.pyc``. This naming convention allows compiled modules from different releases and different versions of Python to coexist.Python checks the modification date of the source against the compiled version to see if it’s out of date and needs to be recompiled. This is a completely automatic process. Also, the compiled modules are platform-independent, so the same library can be shared among systems with different architectures.

You can use the ``-O`` or ``-OO`` switches on the Python command to reduce the size of a compiled module. The ``-O`` switch removes assert statements, the ``-OO`` switch removes both assert statements and ``__doc__`` strings. Since some programs may rely on having these available, you should only use this option if you know what you’re doing. “Optimized” modules have a .pyo rather than a ``.pyc`` suffix and are usually smaller. Future releases may change the effects of optimization.The module ``compileall`` can create ``.pyc`` files (or ``.pyo`` files when ``-O`` is used) for all modules in a directory.

<h3 id="6.2">Standard Modules</h3>

The variable sys.path is a list of strings that determines the interpreter’s search path for modules. It is initialized to a default path taken from the environment variable ``PYTHONPATH``, or from a built-in default if ``PYTHONPATH`` is not set. You can modify it using standard list operations.

<h3 id="6.3">The ``dir()`` Function</h3>

The built-in function ``dir()`` is used to find out which names a module defines. It returns a sorted list of strings.Without arguments, ``dir()`` lists the names you have defined currently.

```python
>>> import fibo, sys
>>> dir(fibo)
['__name__', 'fib', 'fib2']
>>> dir(sys)  
['__displayhook__', '__doc__', '__excepthook__', '__loader__', '__name__',
 '__package__', '__stderr__', '__stdin__', '__stdout__',
 '_clear_type_cache', '_current_frames', '_debugmallocstats', '_getframe',
 '_home', '_mercurial', '_xoptions', 'abiflags', 'api_version', 'argv',
 'base_exec_prefix', 'base_prefix', 'builtin_module_names', 'byteorder',
 'call_tracing', 'callstats', 'copyright', 'displayhook',
 'dont_write_bytecode', 'exc_info', 'excepthook', 'exec_prefix',
 'executable', 'exit', 'flags', 'float_info', 'float_repr_style',
 'getcheckinterval', 'getdefaultencoding', 'getdlopenflags',
 'getfilesystemencoding', 'getobjects', 'getprofile', 'getrecursionlimit',
 'getrefcount', 'getsizeof', 'getswitchinterval', 'gettotalrefcount',
 'gettrace', 'hash_info', 'hexversion', 'implementation', 'int_info',
 'intern', 'maxsize', 'maxunicode', 'meta_path', 'modules', 'path',
 'path_hooks', 'path_importer_cache', 'platform', 'prefix', 'ps1',
 'setcheckinterval', 'setdlopenflags', 'setprofile', 'setrecursionlimit',
 'setswitchinterval', 'settrace', 'stderr', 'stdin', 'stdout',
 'thread_info', 'version', 'version_info', 'warnoptions']
```

Without arguments, ``dir()`` lists the names you have defined currently:

```python
>>> a = [1, 2, 3, 4, 5]
>>> import fibo
>>> fib = fibo.fib
>>> dir()
['__builtins__', '__name__', 'a', 'fib', 'fibo', 'sys']
```

Note that it lists all types of names: variables, modules, functions, etc.

``dir()`` does not list the names of built-in functions and variables. If you want a list of those, they are defined in the standard module ``builtins``.

<h3 id="6.4">Packages</h3>

When importing the package, Python searches through the directories on ``sys.path`` looking for the package subdirectory.The ``__init__.py`` files are required to make Python treat the directories as **containing packages**; this is done to prevent directories with a common name, such as string, from unintentionally hiding valid modules that occur later on the module search path. In the simplest case, ``__init__.py`` can just be an empty file, but it can also execute initialization code for the package or set the ``__all__`` variable.

<h4 id="6.4.1">Importing * From a Package</h4>

Now what happens when the user writes ``from sound.effects import *``? Ideally, one would hope that this somehow goes out to the filesystem, finds which submodules are present in the package, and imports them all. This could take a long time and importing sub-modules might have unwanted side-effects that should only happen when the sub-module is explicitly imported.The only solution is for the package author to provide an explicit index of the package. The import statement uses the following convention: if a package’s ``__init__.py`` code defines a list named ``__all__``, it is taken to be the list of module names that should be imported when ``from package import *`` is encountered. It is up to the package author to keep this list up-to-date when a new version of the package is released. 

<h4 id="6.4.2">Intra-package References</h4>
<h4 id="6.4.3">Packages in Multiple Directories</h4>

Packages support one more special attribute, ``__path__``. This is initialized to be a list containing the name of the directory holding the package’s ``__init__.py`` before the code in that file is executed. This variable can be modified; doing so affects future searches for modules and subpackages contained in the package.While this feature is not often needed, it can be used to extend the set of modules found in a package.

<h2 id="7">Input and Output</h2>
<h3 id="7.1">Fancier Output Formatting</h3>

The ``string`` module contains a ``Template`` class which offers yet another way to substitute values into strings.

```python
>>> print('We are the {} who say "{}!"'.format('knights', 'Ni'))
We are the knights who say "Ni!"
```

A number in the brackets can be used to refer to the position of the object passed into the ``str.format()`` method.

```python
>>> print('{0} and {1}'.format('spam', 'eggs'))
spam and eggs
>>> print('{1} and {0}'.format('spam', 'eggs'))
eggs and spam
```

If keyword arguments are used in the ``str.format()`` method, their values are referred to by using the name of the argument.

```python
>>> print('This {food} is {adjective}.'.format(
...       food='spam', adjective='absolutely horrible'))
This spam is absolutely horrible.
```

Positional and keyword arguments can be arbitrarily combined:

```python
>>> print('The story of {0}, {1}, and {other}.'.format('Bill', 'Manfred',
                                                       other='Georg'))
The story of Bill, Manfred, and Georg.
```

``'!a'`` (apply ``ascii()``), ``'!s'`` (apply ``str()``) and ``'!r'`` (apply ``repr()``) can be used to convert the value before it is formatted:

```python
>>> import math
>>> print('The value of PI is approximately {}.'.format(math.pi))
The value of PI is approximately 3.14159265359.
>>> print('The value of PI is approximately {!r}.'.format(math.pi))
The value of PI is approximately 3.141592653589793.
```

<h4 id="7.1.1">Old string formatting</h4>

The ``%`` operator can also be used for string formatting. It interprets the left argument much like a ``sprintf()``-style format string to be applied to the right argument, and returns the string resulting from this formatting operation. 

```python
>>> import math
>>> print('The value of PI is approximately %5.3f.' % math.pi)
The value of PI is approximately 3.142.
```

<h3 id="7.2">Reading and Writing Files</h3>

``open()`` returns a file object, and is most commonly used with two arguments: ``open(filename, mode)``.

<h4 id="7.2.1">Methods of File Objects</h4>

To read a file’s contents, call ``f.read(size)``, which reads some quantity of data and returns it as a string or bytes object. size is an optional numeric argument. When size is omitted or negative, the entire contents of the file will be read and returned; it’s your problem if the file is twice as large as your machine’s memory. Otherwise, at most size bytes are read and returned. If the end of the file has been reached, ``f.read()`` will return an empty string ('').

```python
>>> f.read()
'This is the entire file.\n'
>>> f.read()
''
```

If you want to read all the lines of a file in a list you can also use ``list(f)`` or ``f.readlines()``.

``f.write(string)`` writes the contents of string to the file, returning the number of characters written.

``f.tell()`` returns an integer giving the file object’s current position in the file represented as number of bytes from the beginning of the file when in binary mode and an opaque number when in text mode.

To change the file object’s position, use ``f.seek(offset, from_what)``. The position is computed from adding offset to a reference point; the reference point is selected by the from_what argument. A from_what value of 0 measures from the beginning of the file, 1 uses the current file position, and 2 uses the end of the file as the reference point. from_what can be omitted and defaults to 0, using the beginning of the file as the reference point.

When you’re done with a file, call ``f.close()`` to close it and free up any system resources taken up by the open file. After calling ``f.close()``, attempts to use the file object will automatically fail.

It is good practice to use the ``with`` keyword when dealing with file objects. This has the advantage that the file is properly closed after its suite finishes, even if an exception is raised on the way. It is also much shorter than writing equivalent ``try-finally`` blocks:

```python
>>> with open('workfile', 'r') as f:
...     read_data = f.read()
>>> f.closed
True
```

<h4 id="7.2.2">Saving structured data with json</h4>

Another variant of the ``dumps()`` function, called ``dump()``, simply serializes the object to a **text file**. So if f is a text file object opened for writing, we can do this:``json.dump(x, f)``.To decode the object again, if f is a text file object which has been opened for reading:``x = json.load(f)``

<h2 id="8">Errors and Exceptions</h2>


















