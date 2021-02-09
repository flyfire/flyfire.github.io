+++
title = "Singleton Pattern"
date = "2019-08-12"
slug = "2019/08/12/singleton-pattern"
Categories = ["dev", "java"]
+++
本文主要讲解单例模式的各种实现，以及对反射和反序列化攻击的防御。

<!-- more -->

单例模式的实现主要注意以下几点：

+ 私有构造器
+ 线程安全
+ 延迟加载
+ 序列化和反序列化安全
+ 反射攻击

下面依次看各种实现：

### 饿汉式

饿汉式顾名思义就是在类加载的时候直接实例化单例。由于是在类加载的时候进行，所以可能拖慢启动速度，而且由于是static变量，生命周期比较长，比较占用内存，特别是如果实例化了而又没有用到的话就白白浪费了内存。

#### 对反序列化的防御

``ObjectInputStream``的``readObject``方法时会判断如果一个类实现了``Serializable``接口并且定义了``readResolve``方法的话会调用``readResolve``方法来进行反序列化，因此可以在``readResolve``方法里面进行处理。

#### 对反射的防御

由于实例在类加载的时候已经实例化，因此可以在构造函数中对实例进行判断，如果不为``null``直接抛出``RuntimeException``来阻止反射创建实例。

具体代码可见[HungrySingleton.java](https://github.com/flyfire/DesignPatternsLearning/blob/master/src/main/java/com/solarexsoft/designpatterns/pattern/creational/singleton/HungrySingleton.java)

### 懒汉式

顾名思义就是在用到的时候再进行实例化，同时使用static synchronized方法来保证线程安全。但是这种实现方式每次进``getInstance``方法都要获取锁，实际上只有实例化的时候需要获取锁，其他时候直接返回就好了，对这种方案的优化就是下面的双重检查锁方案，现在我们暂时先看这种方案。

#### 对反序列化的防御

同样，在``readResolve``方法中进行处理。

#### 对反射的防御

由于实例只有在调用``getInstance``方法之后才进行实例化，因此无法进行防御。

具体代码见[LazySingleton.java](https://github.com/flyfire/DesignPatternsLearning/blob/master/src/main/java/com/solarexsoft/designpatterns/pattern/creational/singleton/LazySingleton.java)

### 双重检查锁

如上所说，双重检查锁是为了解决加锁过重问题而产生的方案。这种方案有个问题，就是调用new来实例化一个对象并不是原子操作，其中初始化对象这步和将实例指向分配的内存地址这步可能会重排序，在某种情况下，可能出现其他线程使用一个未完全初始化的对象的问题。解决这个问题有两种方案，一是禁止重排序，通过在变量时加上volatile修饰符来解决，二是允许重排序，但是这种重排序不能对外界线程可见。双重检查锁使用的是volatile修饰变量来解决这种问题，使用第二种方案来解决问题的方式是我们下面要讲的静态内部类方式。

#### 对反序列化的防御

同样，在``readResolve``方法中进行处理。

#### 对反射的防御

和懒汉式一样，无法防御。

具体代码见[LazyDoubleCheckSingleton.java](https://github.com/flyfire/DesignPatternsLearning/blob/master/src/main/java/com/solarexsoft/designpatterns/pattern/creational/singleton/LazyDoubleCheckSingleton.java)

### 静态内部类

这种方式被推荐是因为使用了Class对象的初始化锁来保证指令重排序对外界线程不可见，实现方式很优雅。

#### 对反序列化的防御

在``readResolve``方法中处理即可。

#### 对反射的防御

和饿汉式一样，也是在类加载时初始化实例，因此可以在构造函数中进行判断并抛出异常。

具体代码见[StaticInnerClassSingleton.java](https://github.com/flyfire/DesignPatternsLearning/blob/master/src/main/java/com/solarexsoft/designpatterns/pattern/creational/singleton/StaticInnerClassSingleton.java)

### Enum实现单例

EffectiveJava中告诉我们可以使用Enum来实现单例，并且这种方式实现的单例天然具有防御反序列化和反射攻击的优势。下面我们看他是如何实现的。

#### 对反序列化的防御

``ObjectInputStream``方法中``readEnum``直接调用了``Enum.valueOf``方法来进行Enum的反序列化，因此得到的是同样的实例。

#### 对反射的防御

``Constructor``的``newInstance``方法会对要反射类的``modifiers``进行判断，如果是``Enum``会直接抛出``IllegalArgumentException``异常。

具体代码见[EnumSingleton.java](https://github.com/flyfire/DesignPatternsLearning/blob/master/src/main/java/com/solarexsoft/designpatterns/pattern/creational/singleton/EnumSingleton.java)

从[反编译后的代码](https://github.com/flyfire/DesignPatternsLearning/blob/master/src/main/java/com/solarexsoft/designpatterns/pattern/creational/singleton/EnumSingleton.jad)来看，Enum实现单例和饿汉式类似，都是在类加载静态初始化时初始化了实例对象。

### Kotlin简单单例

kotlin实现单例很简单，声明为``object``即可。具体代码可见[KotlinSimpleSingleton.kt](https://github.com/flyfire/DesignPatternsLearning/blob/master/src/main/java/com/solarexsoft/designpatterns/pattern/creational/singleton/KotlinSimpleSingleton.kt)。反编译代码可以看到，实际实现方式和饿汉式类似。

```java
public final class Solarex {
   public static final Solarex INSTANCE;

   static {
      Solarex var0 = new Solarex();
      INSTANCE = var0;
   }
}
```

### Kotlin带参数的单例

一般构造单例对象的时候需要注入一些对象参数，参考[这篇文章](https://medium.com/@BladeCoder/kotlin-singletons-with-argument-194ef06edd9e)实现了一下带参数的kotlin单例。具体代码可见[KotlinSingletonWithArguments.kt](https://github.com/flyfire/DesignPatternsLearning/blob/master/src/main/java/com/solarexsoft/designpatterns/pattern/creational/singleton/KotlinSingletonWithArguments.kt)。其实实现仍是双重检查+volatile。

### reference

+ [Singleton](https://github.com/flyfire/DesignPatternsLearning/tree/master/src/main/java/com/solarexsoft/designpatterns/pattern/creational/singleton)

+ [Kotlin singletons with argument](https://medium.com/@BladeCoder/kotlin-singletons-with-argument-194ef06edd9e)
