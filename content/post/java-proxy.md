+++
title = "Java Proxy"
date = "2018-04-26"
slug = "2018/04/26/java-proxy"
Categories = ["dev", "android"]
+++
Java代理有静态代理和动态代理之分。

### 静态代理
静态代理类图如下：

<p><center><img src="/images/java-proxy-static.png" width=225 height=225/></center></p>

``ProxyObject``持有``RealObject``的引用，在``someOperation``方法中可以代理``RealObject``做操作。

<!-- more -->

### 动态代理

动态代理有Java的InvocationHandler实现方式和CGLib实现方式，我没用过CGLib，这里只讨论``InvocationHandler``实现方式。主要有三个步骤，代码示例可以在[github](https://github.com/flyfire/YouDontKnowJava/tree/master/src/com/solarexsoft/test/proxy)上面看

+ 定义一个interface
+ 定义一个InvocationHandler实现类，将要代理的对象传入InvocationHandler中，也即让InvocationHandler实现类代理传入的对象
+ 调用Proxy.newProxyInstance

动态代理的秘密主要是藏在``Proxy.newProxyInstance``当中

```java
public static Object newProxyInstance(ClassLoader loader,Class<?>[] interfaces,InvocationHandler h) throws IllegalArgumentException
{
    ...
        /*
         * Look up or generate the designated proxy class.
         */
        Class<?> cl = getProxyClass0(loader, intfs);
        final Constructor<?> cons = cl.getConstructor(constructorParams);
        final InvocationHandler ih = h;
        return cons.newInstance(new Object[]{h});
}
```

可以看出它先调用``getProxyClass(loader, interfaces)``得到动态代理类，然后将``InvocationHandler``作为代理类构造函数入参新建代理类对象。

```java
    /**
     * Returns the {@code java.lang.Class} object for a proxy class
     * given a class loader and an array of interfaces.  The proxy class
     * will be defined by the specified class loader and will implement
     * all of the supplied interfaces.  If any of the given interfaces
     * is non-public, the proxy class will be non-public. If a proxy class
     * for the same permutation of interfaces has already been defined by the
     * class loader, then the existing proxy class will be returned; otherwise,
     * a proxy class for those interfaces will be generated dynamically
     * and defined by the class loader.
     *
     * <p>There are several restrictions on the parameters that may be
     * passed to {@code Proxy.getProxyClass}:
     *
     * <ul>
     * <li>All of the {@code Class} objects in the
     * {@code interfaces} array must represent interfaces, not
     * classes or primitive types.
     *
     * <li>No two elements in the {@code interfaces} array may
     * refer to identical {@code Class} objects.
     *
     * <li>All of the interface types must be visible by name through the
     * specified class loader.  In other words, for class loader
     * {@code cl} and every interface {@code i}, the following
     * expression must be true:
     * <pre>
     *     Class.forName(i.getName(), false, cl) == i
     * </pre>
     *
     * <li>All non-public interfaces must be in the same package;
     * otherwise, it would not be possible for the proxy class to
     * implement all of the interfaces, regardless of what package it is
     * defined in.
     *
     * <li>For any set of member methods of the specified interfaces
     * that have the same signature:
     * <ul>
     * <li>If the return type of any of the methods is a primitive
     * type or void, then all of the methods must have that same
     * return type.
     * <li>Otherwise, one of the methods must have a return type that
     * is assignable to all of the return types of the rest of the
     * methods.
     * </ul>
     *
     * <li>The resulting proxy class must not exceed any limits imposed
     * on classes by the virtual machine.  For example, the VM may limit
     * the number of interfaces that a class may implement to 65535; in
     * that case, the size of the {@code interfaces} array must not
     * exceed 65535.
     * </ul>
     *
     * <p>If any of these restrictions are violated,
     * {@code Proxy.getProxyClass} will throw an
     * {@code IllegalArgumentException}.  If the {@code interfaces}
     * array argument or any of its elements are {@code null}, a
     * {@code NullPointerException} will be thrown.
     *
     * <p>Note that the order of the specified proxy interfaces is
     * significant: two requests for a proxy class with the same combination
     * of interfaces but in a different order will result in two distinct
     * proxy classes.
     *
     */
/**
 * 得到代理类，不存在则动态生成
 * @param loader 代理类所属 ClassLoader
 * @param interfaces 代理类需要实现的接口
 * @return
 */
public static Class<?> getProxyClass(ClassLoader loader,
                                     Class<?>... interfaces)
    throws IllegalArgumentException
{
    if (interfaces.length > 65535) {
        throw new IllegalArgumentException("interface limit exceeded");
    }
    // 代理类类对象
    Class proxyClass = null;

    /* collect interface names to use as key for proxy class cache */
    String[] interfaceNames = new String[interfaces.length];

    Set interfaceSet = new HashSet();       // for detecting duplicates

    /**
     * 入参 interfaces 检验，包含三部分
     * （1）是否在入参指定的 ClassLoader 内
     * （2）是否是 Interface
     * （3）interfaces 中是否有重复
     */
    for (int i = 0; i < interfaces.length; i++) {
        String interfaceName = interfaces[i].getName();
        Class interfaceClass = null;
        try {
            interfaceClass = Class.forName(interfaceName, false, loader);
        } catch (ClassNotFoundException e) {
        }
        if (interfaceClass != interfaces[i]) {
            throw new IllegalArgumentException(
                interfaces[i] + " is not visible from class loader");
        }

        if (!interfaceClass.isInterface()) {
            throw new IllegalArgumentException(
                interfaceClass.getName() + " is not an interface");
        }

        if (interfaceSet.contains(interfaceClass)) {
            throw new IllegalArgumentException(
                "repeated interface: " + interfaceClass.getName());
        }
        interfaceSet.add(interfaceClass);

        interfaceNames[i] = interfaceName;
    }

    // 以接口名对应的 List 作为缓存的 key
    Object key = Arrays.asList(interfaceNames);

    /*
     * loaderToCache 是个双层的 Map
     * 第一层 key 为 ClassLoader，第二层 key 为 上面的 List，value 为代理类的弱引用
     */
    Map cache;
    synchronized (loaderToCache) {
        cache = (Map) loaderToCache.get(loader);
        if (cache == null) {
            cache = new HashMap();
            loaderToCache.put(loader, cache);
        }
    }

    /*
     * 以上面的接口名对应的 List 为 key 查找代理类，如果结果为：
     * (1) 弱引用，表示代理类已经在缓存中
     * (2) pendingGenerationMarker 对象，表示代理类正在生成中，等待生成完成通知。
     * (3) null 表示不在缓存中且没有开始生成，添加标记到缓存中，继续生成代理类
     */
    synchronized (cache) {
        do {
            Object value = cache.get(key);
            if (value instanceof Reference) {
                proxyClass = (Class) ((Reference) value).get();
            }
            if (proxyClass != null) {
                // proxy class already generated: return it
                return proxyClass;
            } else if (value == pendingGenerationMarker) {
                // proxy class being generated: wait for it
                try {
                    cache.wait();
                } catch (InterruptedException e) {
                }
                continue;
            } else {
                cache.put(key, pendingGenerationMarker);
                break;
            }
        } while (true);
    }

    try {
        String proxyPkg = null;     // package to define proxy class in

        /*
         * 如果 interfaces 中存在非 public 的接口，则所有非 public 接口必须在同一包下面，后续生成的代理类也会在该包下面
         */
        for (int i = 0; i < interfaces.length; i++) {
            int flags = interfaces[i].getModifiers();
            if (!Modifier.isPublic(flags)) {
                String name = interfaces[i].getName();
                int n = name.lastIndexOf('.');
                String pkg = ((n == -1) ? "" : name.substring(0, n + 1));
                if (proxyPkg == null) {
                    proxyPkg = pkg;
                } else if (!pkg.equals(proxyPkg)) {
                    throw new IllegalArgumentException(
                        "non-public interfaces from different packages");
                }
            }
        }

        if (proxyPkg == null) {     // if no non-public proxy interfaces,
            proxyPkg = "";          // use the unnamed package
        }

        {
            // 得到代理类的类名，jdk 1.6 版本中缺少对这个生成类已经存在的处理。
            long num;
            synchronized (nextUniqueNumberLock) {
                num = nextUniqueNumber++;
            }
            String proxyName = proxyPkg + proxyClassNamePrefix + num;

            // 动态生成代理类的字节码
            // 最终调用 sun.misc.ProxyGenerator.generateClassFile() 得到代理类相关信息写入 DataOutputStream 实现
            byte[] proxyClassFile = ProxyGenerator.generateProxyClass(
                proxyName, interfaces);
            try {
                // native 层实现，虚拟机加载代理类并返回其类对象
                proxyClass = defineClass0(loader, proxyName,
                    proxyClassFile, 0, proxyClassFile.length);
            } catch (ClassFormatError e) {
                throw new IllegalArgumentException(e.toString());
            }
        }
        // add to set of all generated proxy classes, for isProxyClass
        proxyClasses.put(proxyClass, null);

    } finally {
        // 代理类生成成功则保存到缓存，否则从缓存中删除，然后通知等待的调用
        synchronized (cache) {
            if (proxyClass != null) {
                cache.put(key, new WeakReference(proxyClass));
            } else {
                cache.remove(key);
            }
            cache.notifyAll();
        }
    }
    return proxyClass;
}

```

函数主要包括三部分：

+ 入参 interfaces 检验，包含是否在入参指定的 ClassLoader 内、是否是 Interface、interfaces 中是否有重复
以接口名对应的 List 为 key 查找代理类，如果结果为：
  - 弱引用，表示代理类已经在缓存中；
  - pendingGenerationMarker 对象，表示代理类正在生成中，等待生成完成返回；
  - null 表示不在缓存中且没有开始生成，添加标记到缓存中，继续生成代理类。
+ 如果代理类不存在调用ProxyGenerator.generateProxyClass(…)生成代理类并存入缓存，通知在等待的缓存。

函数中几个注意的地方：

+ 代理类的缓存 key 为接口名对应的 List，接口顺序不同表示不同的 key 即不同的代理类。
+ 如果 interfaces 中存在非 public 的接口，则所有非 public 接口必须在同一包下面，后续生成的代理类也会在该包下面。
+ 代理类如果在 ClassLoader 中已经存在的情况没有做处理。

可以开启 System Properties 的``sun.misc.ProxyGenerator.saveGeneratedFiles``开关，保存动态类到目的地址。

Java 1.7 的实现略有不同，通过``getProxyClass0(…)``函数实现，实现中调用代理类的缓存，判断代理类在缓存中是否已经存在，存在直接返回，不存在则调用proxyClassCache的valueFactory属性进行动态生成，valueFactory的apply函数与上面的``getProxyClass(…)``函数逻辑类似。

```
import java.lang.reflect.InvocationHandler;
import java.lang.reflect.Method;
import java.lang.reflect.Proxy;
import java.lang.reflect.UndeclaredThrowableException;

public final class $Proxy0 extends Proxy
  implements Operate
{
  private static Method m4;
  private static Method m1;
  private static Method m5;
  private static Method m0;
  private static Method m3;
  private static Method m2;

  public $Proxy0(InvocationHandler paramInvocationHandler)
    throws 
  {
    super(paramInvocationHandler);
  }

  public final void operateMethod1()
    throws 
  {
    try
    {
      h.invoke(this, m4, null);
      return;
    }
    catch (Error|RuntimeException localError)
    {
      throw localError;
    }
    catch (Throwable localThrowable)
    {
      throw new UndeclaredThrowableException(localThrowable);
    }
  }

  public final boolean equals(Object paramObject)
    throws 
  {
    try
    {
      return ((Boolean)h.invoke(this, m1, new Object[] { paramObject })).booleanValue();
    }
    catch (Error|RuntimeException localError)
    {
      throw localError;
    }
    catch (Throwable localThrowable)
    {
      throw new UndeclaredThrowableException(localThrowable);
    }
  }

  public final void operateMethod2()
    throws 
  {
    try
    {
      h.invoke(this, m5, null);
      return;
    }
    catch (Error|RuntimeException localError)
    {
      throw localError;
    }
    catch (Throwable localThrowable)
    {
      throw new UndeclaredThrowableException(localThrowable);
    }
  }

  public final int hashCode()
    throws 
  {
    try
    {
      return ((Integer)h.invoke(this, m0, null)).intValue();
    }
    catch (Error|RuntimeException localError)
    {
      throw localError;
    }
    catch (Throwable localThrowable)
    {
      throw new UndeclaredThrowableException(localThrowable);
    }
  }

  public final void operateMethod3()
    throws 
  {
    try
    {
      h.invoke(this, m3, null);
      return;
    }
    catch (Error|RuntimeException localError)
    {
      throw localError;
    }
    catch (Throwable localThrowable)
    {
      throw new UndeclaredThrowableException(localThrowable);
    }
  }

  public final String toString()
    throws 
  {
    try
    {
      return (String)h.invoke(this, m2, null);
    }
    catch (Error|RuntimeException localError)
    {
      throw localError;
    }
    catch (Throwable localThrowable)
    {
      throw new UndeclaredThrowableException(localThrowable);
    }
  }

  static
  {
    try
    {
      m4 = Class.forName("com.codekk.java.test.dynamicproxy.Operate").getMethod("operateMethod1", new Class[0]);
      m1 = Class.forName("java.lang.Object").getMethod("equals", new Class[] { Class.forName("java.lang.Object") });
      m5 = Class.forName("com.codekk.java.test.dynamicproxy.Operate").getMethod("operateMethod2", new Class[0]);
      m0 = Class.forName("java.lang.Object").getMethod("hashCode", new Class[0]);
      m3 = Class.forName("com.codekk.java.test.dynamicproxy.Operate").getMethod("operateMethod3", new Class[0]);
      m2 = Class.forName("java.lang.Object").getMethod("toString", new Class[0]);
      return;
    }
    catch (NoSuchMethodException localNoSuchMethodException)
    {
      throw new NoSuchMethodError(localNoSuchMethodException.getMessage());
    }
    catch (ClassNotFoundException localClassNotFoundException)
    {
      throw new NoClassDefFoundError(localClassNotFoundException.getMessage());
    }
  }
}
```

可以看到实际上在底层生成了一个class的字节流，并且被ClassLoader加载了，生成了一个继承``Proxy``实现了接口的类，这个类以``InvocationHandler``为构造函数参数，所以实际上是在调用方法时，生成的``$Proxy0``对象代理了``InvocationHandler``实现对象，``InvocationHandler``对象代理了实际的对象。


