---
title: "What's Behind Kotlin Open Val"
date: 2021-04-05T20:07:32+08:00
draft: false
categories: ["kotlin"]
---

最近在做代码重构的时候遇到一个奇怪的问题

<!--more-->

用demo代码表示大概是这样

```kotlin
open class Data(open val hi: Int = 0)
data class Data2(override val hi: Int = 1, val world: Int = 0): Data(hi)

open class AAAA(open val data: Data = Data(), val hi: Int = 1) {
    init {
        println(data)
        println(hi)
    }
}

class BBBB(override val data: Data2 = Data2()): AAAA() {
    init {
        println(data)
    }
}

class CCCC(override val data: Data2 = Data2()): AAAA(data) {
    init {
        println(data)
    }
}

fun main() {
    BBBB()
    println("*" * 20)
    CCCC()
}
```

运行上面的代码，会得到下面的输出：

```
null
1
Data2(hi=1, world=0)
********************
null
1
Data2(hi=1, world=0)
```
