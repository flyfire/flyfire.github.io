---
layout: post
title: "Kotlin是如何实现方法默认参数的"
date: 2019-04-18 10:36
comments: true
categories: 
- dev
- kotlin
tags:
- kotlin
---

学习Kotlin的时候，发现可以给方法设置默认参数，Java是不支持给方法设置默认参数的，那Kotin是如何实现的呢？不妨看下下面的kotlin文件，kotlin允许这样写：

```kotlin
class frob() {
    fun fred(x: Int = 300, y: frob = mkfrob(x)) {
        println("${this}${x}${y}")
    }
    fun mkfrob(x: Int): frob {
        return this;
    }
    fun boobar() {
        fred();
        fred(100);
        fred(100, frob());
    }
}
```

使用``kotlinc``编译成class文件之后，我们使用``cfr``反编译一下class文件看下编译器帮我们做了什么黑魔法。

<!-- more -->

```java
import java.io.PrintStream;
import kotlin.Metadata;
import kotlin.jvm.internal.Intrinsics;
import org.jetbrains.annotations.NotNull;

@Metadata(mv={1, 1, 13}, bv={1, 0, 3}, k=1, d1={"\u0000\u001c\n\u0002\u0018\u0002\n\u0002\u0010\u0000\n\u0002\b\u0002\n\u0002\u0010\u0002\n\u0002\b\u0002\n\u0002\u0010\b\n\u0002\b\u0002\u0018\u00002\u00020\u0001B\u0005\u00a2\u0006\u0002\u0010\u0002J\u0006\u0010\u0003\u001a\u00020\u0004J\u001a\u0010\u0005\u001a\u00020\u00042\b\b\u0002\u0010\u0006\u001a\u00020\u00072\b\b\u0002\u0010\b\u001a\u00020\u0000J\u000e\u0010\t\u001a\u00020\u00002\u0006\u0010\u0006\u001a\u00020\u0007"}, d2={"Lfrob;", "", "()V", "boobar", "", "fred", "x", "", "y", "mkfrob"})
public final class frob {
    public final void fred(int x, @NotNull frob y) {
        Intrinsics.checkParameterIsNotNull((Object)y, (String)"y");
        String string = "" + this + x + y;
        System.out.println((Object)string);
    }

    public static /* synthetic */ void fred$default(frob frob2, int n, frob frob3, int n2, Object object) {
        if ((n2 & 1) != 0) {
            n = 300;
        }
        if ((n2 & 2) != 0) {
            frob3 = frob2.mkfrob(n);
        }
        frob2.fred(n, frob3);
    }

    @NotNull
    public final frob mkfrob(int x) {
        return this;
    }

    public final void boobar() {
        frob.fred$default(this, 0, null, 3, null);
        frob.fred$default(this, 100, null, 2, null);
        this.fred(100, new frob());
    }
}
```

可以看到编译器底层自动为我们生成了一个``static``,``synthetic``的方法，在这个方法中，第一个参数是对象接收者，第二个和第三个参数和我们声明的``fred``方法相同，第四个参数``n2``从方法体可以看出是一个bitmask。

从生成的方法``fred$default``实现来看，每个可以有默认参数的位置的参数有个mask值，为``2^x``，x为参数出现的顺序，如果方法调用的时候用到了某个位置上的默认参数，``fred$default``方法的第四个参数``n2``就会加上``2^x``。

在Java中是不能使用默认参数调用``fred``这种方法的，如果想要在Java中调用，需要给方法加上``@JvmOverloads``注解，我们加上这个注解再去反编译一下看下有什么变化。

```kotlin
class frob() {
    @JvmOverloads
    fun fred(x: Int = 300, y: frob = mkfrob(x)) {
        println("${this}${x}${y}")
    }
    fun mkfrob(x: Int): frob {
        return this;
    }
    fun boobar() {
        fred();
        fred(100);
        fred(100, frob());
    }
}
```

再反编译一下我们看下有什么变化：

```java
import java.io.PrintStream;
import kotlin.Metadata;
import kotlin.jvm.JvmOverloads;
import kotlin.jvm.internal.Intrinsics;
import org.jetbrains.annotations.NotNull;

@Metadata(mv={1, 1, 13}, bv={1, 0, 3}, k=1, d1={"\u0000\u001c\n\u0002\u0018\u0002\n\u0002\u0010\u0000\n\u0002\b\u0002\n\u0002\u0010\u0002\n\u0002\b\u0002\n\u0002\u0010\b\n\u0002\b\u0002\u0018\u00002\u00020\u0001B\u0005\u00a2\u0006\u0002\u0010\u0002J\u0006\u0010\u0003\u001a\u00020\u0004J\u001c\u0010\u0005\u001a\u00020\u00042\b\b\u0002\u0010\u0006\u001a\u00020\u00072\b\b\u0002\u0010\b\u001a\u00020\u0000H\u0007J\u000e\u0010\t\u001a\u00020\u00002\u0006\u0010\u0006\u001a\u00020\u0007"}, d2={"Lfrob;", "", "()V", "boobar", "", "fred", "x", "", "y", "mkfrob"})
public final class frob {
    @JvmOverloads
    public final void fred(int x, @NotNull frob y) {
        Intrinsics.checkParameterIsNotNull((Object)y, (String)"y");
        String string = "" + this + x + y;
        System.out.println((Object)string);
    }

    @JvmOverloads
    public static /* synthetic */ void fred$default(frob frob2, int n, frob frob3, int n2, Object object) {
        if ((n2 & 1) != 0) {
            n = 300;
        }
        if ((n2 & 2) != 0) {
            frob3 = frob2.mkfrob(n);
        }
        frob2.fred(n, frob3);
    }

    @JvmOverloads
    public final void fred(int x) {
        frob.fred$default(this, x, null, 2, null);
    }

    @JvmOverloads
    public final void fred() {
        frob.fred$default(this, 0, null, 3, null);
    }

    @NotNull
    public final frob mkfrob(int x) {
        return this;
    }

    public final void boobar() {
        frob.fred$default(this, 0, null, 3, null);
        frob.fred$default(this, 100, null, 2, null);
        this.fred(100, new frob());
    }
}
```

可以看到比上面多生成了几个重载的方法，在重载的方法内部调用了生成方法``fred$default``。

### reference

+ [How does Kotlin generate default arguments?](https://www.benf.org/other/cfr/kotlin-defaults.html)