---
title: "Java Dynamic Proxy"
date: 2022-02-13T10:49:07+08:00
draft: false
---

动态代理 ProxyGenerator生成的类代理了InvocationHandler，InvocationHandler代理了真实的对象。

```java
@Click({R.id.tv1, R.id.tv2})
fun abc(View view){}

val click = method.getAnnotation(Click::class.java)
val ids = click.value()
for(id in ids) {
    val view = activity.findViewById(id)
    val listener = Proxy.newProxyInstance(classloader, {View.OnClickListener::class.java}, InvocationHandler() {
        method.invoke(activity, view)
    })
    view.setOnClickListener(listener)
}
```