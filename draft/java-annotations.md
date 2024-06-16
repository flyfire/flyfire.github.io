---
title: "Java Annotations"
date: 2022-02-12T17:37:41+08:00
draft: false
---
注解本身没有任何意义，单独的注解就是一种注释，它需要结合其他如反射等技术才有意义。
Java注解又称为Java标注，是JDK1.5引入的一种注释机制，是元数据的一种形式，提供有关于程序但不属于程序本身的数据。注解对它们注解的代码的操作没有直接影响。

元注解：注解上的注解。
Target，声明的注解允许作用于哪些节点。
Retention，保留级别

+ RetentionPolicy.SOURCE 标记的注解仅保留在源码级别中，并被编译器忽略
+ RetentionPolicy.CLASS 标记的注解在编译时由编译器保留，但JVM会忽略
+ RetentionPolicy.RUNTIME 标记的注解由JVM保留，运行时可以通过反射获取。

| 级别 | 技术 | 说明 |
|  ----  | ----  | ----  |
| SOURCE | APT | 在编译期能够获取注解与注解声明的类包括类中所有成员信息，一般用于生成额外的辅助类 |
| CLASS | 字节码增强 | 在编译出Class后，通过修改Class数据以实现修改代码逻辑的目的，对于是否需要修改的区分或者修改为不同逻辑的判断可以使用注解 |
| RUNTIME | 反射 | 在程序运行期间，通过反射技术动态获取注解及其元素，从而完成不同的逻辑判定 |

APT ``resources/META-INF.services/javax.annotation.processing.Processor``

```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.SOURCE)
public @interface Test {
    int value();
    String id();
}
```

只有一个value方法的时候value可以省略，否则只能是key/value形式指定值。

```java
@Retention(SOURCE)
@Target(ANNOTATION_TYPE)
public @interface IntDef {
    int[] value() default {};
    boolean flag() default false;
}
```

IntDef元注解，提供语法检查

对象 对象头(12+)+成员+8字节对齐

```java
@IntDef({SUNDAY, MONDAY})
@Target({ElementType.FIELD, ElementType.PARAMETER})
@Retention(RententionPolicy.SOURCE)
public @interface WeekDay{}
```

语法检查： IDE插件实现的

字节码增强：在字节码中写代码。IO将class文件读取成byte[]，按照特定格式修改。 

class.getField() 子类及父类所有public field，不包括private
class.getDeclaredField() 子类所有的field，包括private的

加花括号是匿名内部类，是子类，有泛型信息，不加花括号是类，没有泛型类型相关信息。

匿名内部类，动态创建的类型记录了泛型签名信息。