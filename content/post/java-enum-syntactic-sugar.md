+++
title = "Java Enum Syntactic Sugar"
date = "2018-07-10"
slug = "2018/07/10/java-enum-syntactic-sugar"
Categories = ["java", "dev"]
+++

Java从1.5引入枚举类型，EffectiveJava第2版item 30也建议我们使用枚举来代替int常量。我们从下面的``Enum``示例中看下枚举类型到底是什么。

```java
public class Main {
    enum ExceptionHandleStrategy {
        IGNORE,
        LOG {
            @Override
            public void handle(Exception e) {
                System.out.println(e.getLocalizedMessage());
            }
        },
        THROW {
            @Override
            public void handle(Exception e) {
                throw new RuntimeException(e.getMessage());
            }
        };

        public void handle(Exception e) {
        }
    }

    public static void main(String[] args) {
        System.out.println(ExceptionHandleStrategy.IGNORE.getClass());
        System.out.println(ExceptionHandleStrategy.LOG.getClass());
        System.out.println(ExceptionHandleStrategy.THROW.getClass());
        System.out.println(EnumWithoutMethod.A.getClass());
        System.out.println(EnumWithoutMethod.B.getClass());
        System.out.println(EnumWithoutMethod.C.getClass());
        System.out.println(EnumWithVariable.A.getClass());
        System.out.println(EnumWithVariable.B.getClass());
        System.out.println(EnumWithVariable.C.getClass());
        System.out.println(EnumWithVariable.A.name());
        System.out.println(EnumWithVariable.A.getxxx());
        ExceptionHandleStrategy.LOG.handle(new RuntimeException("system just broken"));
    }

    enum EnumWithoutMethod {
        A,
        B,
        C;
    }

    enum EnumWithVariable {
        A("wristband"),
        B("glucometer"),
        C("fit");

        private String name;
        public String getxxx() {
            return this.name;
        }

        EnumWithVariable(String name){
            this.name = name;
        }
    }
}
```

<!-- more -->

执行``java -jar ~/Software/confs/cfr-0.140.jar --sugarenums false Main.class``后我们得到如下结果：

```java
import java.io.PrintStream;

public class Main {
    public static void main(String[] arrstring) {
        System.out.println(((Object)((Object)ExceptionHandleStrategy.IGNORE)).getClass());
        System.out.println(((Object)((Object)ExceptionHandleStrategy.LOG)).getClass());
        System.out.println(((Object)((Object)ExceptionHandleStrategy.THROW)).getClass());
        System.out.println(((Object)((Object)EnumWithoutMethod.A)).getClass());
        System.out.println(((Object)((Object)EnumWithoutMethod.B)).getClass());
        System.out.println(((Object)((Object)EnumWithoutMethod.C)).getClass());
        System.out.println(((Object)((Object)EnumWithVariable.A)).getClass());
        System.out.println(((Object)((Object)EnumWithVariable.B)).getClass());
        System.out.println(((Object)((Object)EnumWithVariable.C)).getClass());
        System.out.println(EnumWithVariable.A.name());
        System.out.println(EnumWithVariable.A.getxxx());
        ExceptionHandleStrategy.LOG.handle(new RuntimeException("system just broken"));
    }

    static final class EnumWithVariable
    extends Enum<EnumWithVariable> {
        public static final /* enum */ EnumWithVariable A = new EnumWithVariable("A", 0, "wristband");
        public static final /* enum */ EnumWithVariable B = new EnumWithVariable("B", 1, "glucometer");
        public static final /* enum */ EnumWithVariable C = new EnumWithVariable("C", 2, "fit");
        private String name;
        private static final /* synthetic */ EnumWithVariable[] $VALUES;

        public static EnumWithVariable[] values() {
            return (EnumWithVariable[])$VALUES.clone();
        }

        public static EnumWithVariable valueOf(String string) {
            return Enum.valueOf(EnumWithVariable.class, string);
        }

        public String getxxx() {
            return this.name;
        }

        private EnumWithVariable(String string, int n, String string2) {
            super(string, n);
            this.name = string2;
        }

        static {
            $VALUES = new EnumWithVariable[]{A, B, C};
        }
    }

    static final class EnumWithoutMethod
    extends Enum<EnumWithoutMethod> {
        public static final /* enum */ EnumWithoutMethod A = new EnumWithoutMethod("A", 0);
        public static final /* enum */ EnumWithoutMethod B = new EnumWithoutMethod("B", 1);
        public static final /* enum */ EnumWithoutMethod C = new EnumWithoutMethod("C", 2);
        private static final /* synthetic */ EnumWithoutMethod[] $VALUES;

        public static EnumWithoutMethod[] values() {
            return (EnumWithoutMethod[])$VALUES.clone();
        }

        public static EnumWithoutMethod valueOf(String string) {
            return Enum.valueOf(EnumWithoutMethod.class, string);
        }

        private EnumWithoutMethod(String string, int n) {
            super(string, n);
        }

        static {
            $VALUES = new EnumWithoutMethod[]{A, B, C};
        }
    }

    static class ExceptionHandleStrategy
    extends Enum<ExceptionHandleStrategy> {
        public static final /* enum */ ExceptionHandleStrategy IGNORE = new ExceptionHandleStrategy("IGNORE", 0);
        public static final /* enum */ ExceptionHandleStrategy LOG = new ExceptionHandleStrategy("LOG", 1){

            @Override
            public void handle(Exception exception) {
                System.out.println(exception.getLocalizedMessage());
            }
        };
        public static final /* enum */ ExceptionHandleStrategy THROW = new ExceptionHandleStrategy("THROW", 2){

            @Override
            public void handle(Exception exception) {
                throw new RuntimeException(exception.getMessage());
            }
        };
        private static final /* synthetic */ ExceptionHandleStrategy[] $VALUES;

        public static ExceptionHandleStrategy[] values() {
            return (ExceptionHandleStrategy[])$VALUES.clone();
        }

        public static ExceptionHandleStrategy valueOf(String string) {
            return Enum.valueOf(ExceptionHandleStrategy.class, string);
        }

        private ExceptionHandleStrategy(String string, int n) {
            super(string, n);
        }

        public void handle(Exception exception) {
        }

        static {
            $VALUES = new ExceptionHandleStrategy[]{IGNORE, LOG, THROW};
        }

    }

}
```

可以看到我们的枚举类型都继承了``Enum``，编译器为我们生成了``$VALUES``类型数组，并在``static``静态块中进行了初始化。每个枚举类型实际上是个继承了``Enum``的类，并在类初始化的时候实例化了枚举类型的实例，可以看到非常重量级，实际上在Android开发中Google已经建议不要使用``Enum``，改而使用``@IntDef``之类的注解进行约束。

在看EffectiveJava的时候，看到作者认为Enum是实现单例的一种方式，``Enum``实现的单例帮我们处理了反射和序列化相关的问题，那它是怎么处理的呢？我们不妨看下源码来找下答案。

<strike>首先看下``Enum``类的``readObject``方法：</strike>

```java
/**
  * prevent default deserialization
  */
private void readObject(ObjectInputStream in) throws IOException,
    ClassNotFoundException {
    throw new InvalidObjectException("can't deserialize enum");
}
```

<strike>可以看到这个方法直接抛出了一个异常来阻止反序列化。</strike>

``ObjectInputStream``的``readEnum``方法使用了``Enum.valueOf``方法来进行``Enum``的反序列化，保证了实例的唯一。

再来看下如果我们想要使用反射来实例化一个枚举实例的时候会遇到什么问题，我们看下``Constructor``类的``newInstance``方法：

```java
public T newInstance(Object ... initargs)
        throws InstantiationException, IllegalAccessException,
               IllegalArgumentException, InvocationTargetException
    {
        if (!override) {
            if (!Reflection.quickCheckMemberAccess(clazz, modifiers)) {
                Class<?> caller = Reflection.getCallerClass();
                checkAccess(caller, clazz, null, modifiers);
            }
        }
        // 阻止了通过反射来实例化Enum实例
        if ((clazz.getModifiers() & Modifier.ENUM) != 0)
            throw new IllegalArgumentException("Cannot reflectively create enum objects");
        ConstructorAccessor ca = constructorAccessor;   // read volatile
        if (ca == null) {
            ca = acquireConstructorAccessor();
        }
        @SuppressWarnings("unchecked")
        T inst = (T) ca.newInstance(initargs);
        return inst;
    }
```

可以看到在方法内如果发现要反射类的Modifier中有``Enum``标志，会直接抛出异常表示无法通过反射的方式来创建``Enum``实例。

### reference

+ [How are Enums implemented?](https://www.benf.org/other/cfr/how-are-enums-implemented.html)
