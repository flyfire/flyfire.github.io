---
title: "Java泛型"
date: 2022-02-12T10:31:52+08:00
draft: false
---

方法相同，参数不同
指定泛型参数，可以编译时报错，避免显式强制类型转换

泛型类
泛型接口

实现泛型接口

+ 泛型类型向下传递
+ 指定泛型类型

泛型方法可以存在于普通类中，``<T>``

泛型类中的泛型方法的泛型T可以跟泛型类中的泛型T不一致。

```java
class A<T> {
    public void show1(T t) {}
    public <E> void show2(E e){}
    public <T> void show3(T t){}
}
泛型类A的泛型T只影响到show1方法的类型，show3的T类型可以和A类的T类型不一致
```

类型变量的限定

```java
public <T extends Comparable&Seriable> T min(T a, T b) {
    if(a.compareTo(b) > 0) {
        return b;
    } else {
        return a;
    }
}
```

T extends的可以是类或者接口，如果既有类又有接口，类要写在第一个。

泛型限制

+ 不能实例化类型变量
+ 静态域或者方法里不能引用类型变量
+ 只能是包装类型，不能是基础类型
+ 不能用instanceof判定泛型变量类型
+ 可以定义泛型数组，不能创建泛型数组
+ 泛型类不能extends Exception/Throwable,不能捕获泛型类对象，可以抛出去 

```java
Restrict<Double>[] array; // right
Restrict<Double>[] array = new Restrict<Double>[10]; // wrong

class Problem<T> extends Exception {} // wrong
public <T extends Throwable> void do1(T t) {
    try {

    } catch (T t) {} // wrong
}

public <T extends Throwable> void do2(T t) throws T {
    try {

    } catch(Throwable e) {
        throw t;  // right
    }
}
```

```
class Pair<T>
A extends B, Pair<A>和Pair<B>没有任何关系
```

泛型类可以继承或者扩展其他泛型类。

class Sub<T> extends Parent<T>
class Sub extends Parent<Double>

通配符使用在方法上的时候。
? extends Fruit 用于安全的访问数据，规定了类型的上界
? super Apple 用于安全的写入数据，写入类及其子类数据 ，规定了类型的下界

```java

class GenericType<T> {
    T t;

    void setT(T t) {
        this.t = t;
    }

    T getT() {
        return t;
    }
}
GenericType<Food>  food = new GenericType();
GenericType<Fruit> fruit = new GenericType();
GenericType<Apple> apple = new GenericType();
GenericType<Orange> orange = new GenericType();
GenericType<HongFuShi> hongFushi = new GenericType();

static void printExtends(GenericType<? extends Fruit> p){}

printExtends(fruit) // right
printExtends(apple) // right
printExtends(orange) // right
printExtends(hongfushi) // right
printExtends(food) // wrong

GenericType<? extends Fruit> c = new GenericType<>();
c.setT(new Apple()); // wrong
c.setT(new Fruit()); // wrong
Fruit x = c.getT();

static void printSuper(GenericType<? super Apple> p){}

printSuper(fruit) // right
printSuper(apple) // right
printSuper(orange) // wrong
printSuper(hongFushi) // wrong

GenericType<? super Apple> x = new GenericType();
x.setT(new Apple()) // right
x.setT(new HongFuShi()) // right Apple和HongFuShi可以安全地转型为Apple
x.setT(new Fruit()) // wrong
Object t = x.getT();
```

为了获得最大限度的灵活性，要在表示生产者或者消费者的输入参数上使用通配符类型。如果某个输入参数既是生产者，又是消费者，那么通配符类型对你就没有什么好处了：因为你需要的是严格的类型匹配，这是不用任何通配符而得到的。下面的助记符便于让你记住要使用哪种通配符类型：PECS表示producer-extends,consumer-super。换句话说，如果参数化类型表示一个T生产者，就使用<? extends T>;如果它表示一个T消费者，就使用<? super T>。所有的comparator和comparable都是消费者，消费T并产生表示顺序关系的数值。

```java
public static <E> void swap(List<E> list, int i, int j)
public static void swap(List<?> list, int i, int j)
```

一般来说，如果类型参数只在方法声明中出现一次，就可以用通配符取代它。如果是无限制的类型参数，就用无限制的通配符取代它，如果是有限制的类型参数，就用有限制的通配符取代它。

泛型擦除，Class字节码Signature保留泛型信息。

```
class Test<T extends ArrayList&Comparable>{}

字节码擦除成ArrayList，调用Comparable方法的地方插入必要的强制转型
```