# Java 基础 - SPI 机制原理分析
title: Java 基础 - SPI 机制原理分析   
date: 2018/04/13 20:46:25
catalog: true
categories:

- Java
- Java 基础
tags: Java basement
copyright: true

---

本篇文章简要总结和分析下 Java 的  SPI 机制的原理。

# 1 概述

SPI，全称为 Sevice Provider Interface，是 JDK 内置的一种服务提供发现机制，用于框架扩展和代码替换，主要思想是将装配的控制权移到程序之外，在模块化设计中这个机制尤其重要，其核心思想就是解耦；

 Android 中的 dagger  依赖注入框架的内部就有 SPI 的应用！



# 2 基本使用

- 首先要定义一个统一的服务接口：

```java
package com.coolqi.myapplication;

public class ICommonInterface {
    void show(int name) {
    }
}
```

- 然后需要在应用的 **classpath** 目录下创建，**META-INF/services/** 目录，同时创建一个以 **ICommonInterface** 服务接口命名的文件；

```java
META-INF/services/com.coolqi.myapplication.ICommonInterface
```

- 然后，在 **com.coolqi.myapplication.ICommonInterface** 文件中记录那些实现了这个借口的类！

```java
com.coolqi.myapplication.CommonInterfaceA
com.coolqi.myapplication.CommonInterfaceB
com.coolqi.myapplication.CommonInterfaceC
```

- 最后我们要通过 JDK 的 **ServiceLoader** 来查询接口

```java
package com.coolqi.myapplication;

import java.util.ServiceLoader;

public class CoolSPI {

    public static void main(String[] args) {
        ServiceLoader<ICommonInterface> serviceLoader = ServiceLoader.load(ICommonInterface.class);
        for (ICommonInterface helloSPI : serviceLoader) {
            helloSPI.show(1);
        }
    }
}
```

上面就是 **SPI** 的基本使用方法；



# 3 ServiceLoader - 源码分析

下面我们来看看 ServiceLoader 的源码（JDK 1.8）：

## 3.1 成员属性

```java
public final class ServiceLoader<S>
    implements Iterable<S>
{
    //【1】表示要创建的那个一个以 xxx 服务接口命名的文件所在的路径；
    private static final String PREFIX = "META-INF/services/";

    //【2】用于保存要被加载的服务接口
    private final Class<S> service;

    //【3】类加载器；
    private final ClassLoader loader;

    //【4】可以看到，在 Android 里面，这里关闭了 java security 机制；
    // The access control context taken when the ServiceLoader is created
    // Android-changed: do not use legacy security code.
    // private final AccessControlContext acc; 

    //【4】缓存机制，保存加载起来的服务；key 是类的权限定名；value 是通过反射创建的实例；
    private LinkedHashMap<String,S> providers = new LinkedHashMap<>();

    //【4】查询迭代器，ServiceLoader 最终是通过 lookupIterator 来访问服务的；
    private LazyIterator lookupIterator;
```

- **ServiceLoader** 实现了 **Iterable** 接口，说明他具有迭代器的机制，对于迭代器，核心方法就是  **Iterator** 和 **forEach**

- ```java
   public @NonNull Iterator<@NullFromTypeParam T> iterator();
   public default void forEach(@NonNull Consumer<? super T> action) {...}
  ```

对于 ServiceLoader，我们只能通过 foreach 语法糖来访问内部数据，而 foreach 是通过迭代器 Iterator，不断获取next元素，而这里的迭代器是由 LazyIterator lookupIterator 是实现的！

##  3.2 load - 加载服务



```java
    public static <S> ServiceLoader<S> load(Class<S> service) {
        //【1】获取线程的类加载器；
        ClassLoader cl = Thread.currentThread().getContextClassLoader();
        //【2】调用另外一个方法：
        return ServiceLoader.load(service, cl);
    }
```

这里调用了另外一个参数：

```java
    public static <S> ServiceLoader<S> load(Class<S> service,
                                            ClassLoader loader)
    {
        //【-->3.3】创建 ServiceLoader 实例；
        return new ServiceLoader<>(service, loader); 
    }
```

## 3.3 new ServiceLoader

创建 ServiceLoader 实例：

```java
    private ServiceLoader(Class<S> svc, ClassLoader cl) {
        service = Objects.requireNonNull(svc, "Service interface cannot be null");
        //【1】这里是对类加载器做了下验证；
        loader = (cl == null) ? ClassLoader.getSystemClassLoader() : cl;
        // Android-changed: Do not use legacy security code.
        // On Android, System.getSecurityManager() is always null.
        // acc = (System.getSecurityManager() != null) ? AccessController.getContext() : null;
        //【-->3.4】执行 reload 方法；
        reload();
    }
```



##  3.4 reload

清空了缓存，通知创建 LazyIterator 实例：

```java
    public void reload() {
        providers.clear();
        //【-->4.1】创建了 LazyIterator 实例；
        lookupIterator = new LazyIterator(service, loader);
    }
```

核心是创建 **LazyIterator** 实例！



## 3.5 iterator - 核心方法

ServiceLoader 实现了 iterator 方法：

```java
    public Iterator<S> iterator() {
        //【1】这里返回了一个新的 Iterator<S> 匿名类实例！
        return new Iterator<S>() {
            //【-->3.1】内部持有 ServiceLoader 的 LinkedHashMap<String,S> providers 的迭代器实例；
            // 因为 providers 中保存的是缓存，所有优先在缓存中获取；
            Iterator<Map.Entry<String,S>> knownProviders
                = providers.entrySet().iterator();

            //【2】是否有下一个元素？
            public boolean hasNext() {
                if (knownProviders.hasNext()) //【2.1】如果缓存中有，那就从缓存中获取，返回 true；
                    return true;
                return lookupIterator.hasNext(); //【-->4.2】如果缓存没有，那就通过 lookupIterator 判断；
            }

            //【3】返回下一个元素；
            public S next() {
                if (knownProviders.hasNext()) //【3.1】如果缓存中有，那就从缓存中获取；
                    return knownProviders.next().getValue();
                return lookupIterator.next(); //【-->4.3】如果缓存没有，那就通过 lookupIterator 获取；
            }

            public void remove() {
                throw new UnsupportedOperationException();
            }

        };
    }
```

其内部返回了一个新的 Iterator 实例，用遍历数据！

- 可以看到先访问缓存，再访问 lookupIterator 实例；



# 4 LazyIterator

懒加载机制的 Iterator，之所以是懒加载，是因为并不是在创建的时候加载文件的，而是在第一次加载的时候，在初始化的；

## 4.1 成员变量和构造器

```java
    private class LazyIterator
        implements Iterator<S>
    {

        Class<S> service; // 所属的 service loader；
        ClassLoader loader; // 类加载器；
        Enumeration<URL> configs = null; // 用于保存要加载的文件的 url 路径；
        Iterator<String> pending = null; // 文件中的数据的迭代器；
        String nextName = null; // 用于继续下一个要加载的服务名称；
```

同时我们来看看构造器：

```java
        private LazyIterator(Class<S> service, ClassLoader loader) {
            this.service = service;
            this.loader = loader;
        }
```

不多说了；

## 4.2 hasNext

判断是否有下一个元素

```java
        public boolean hasNext() {
            // Android-changed: do not use legacy security code
            /* if (acc == null) { */
                //【-->4.2.1】调用下一个方法；
                return hasNextService();
            /*
            } else {
                PrivilegedAction<Boolean> action = new PrivilegedAction<Boolean>() {
                    public Boolean run() { return hasNextService(); }
                };
                return AccessController.doPrivileged(action, acc);
            }
            */
        }
```

继续看：

### 4.2.1 hasNextService

是否有下一个元素：

```java
        private boolean hasNextService() {
            //【1】如果 nextName 不为 null，说明有下一个元素；当然第一次肯定是 null‘
            if (nextName != null) {
                return true;
            }
            //【2】加载 META-INF/services/ 目录下的那个接口文件，第一次会读取；
            // 返回的是 Enumeration<URL> 的枚举实例；
            if (configs == null) {
                try {
                    String fullName = PREFIX + service.getName(); // 文件名；
                    if (loader == null)
                        configs = ClassLoader.getSystemResources(fullName);
                    else
                        configs = loader.getResources(fullName);
                } catch (IOException x) {
                    fail(service, "Error locating configuration files", x);
                }
            }
            //【3】第一次 pending == null，那么就会按行读取文件的内容，保存到 pending 中；
            // pending 是一个迭代器；
            while ((pending == null) || !pending.hasNext()) {
                if (!configs.hasMoreElements()) {
                    return false;
                }
                pending = parse(service, configs.nextElement());
            }
            //【4】nextname 永远保存下一个要加载的类的全限定名；
            // 也就是 pending 的下一个元素；
            nextName = pending.next();
            return true;
        }
```

方法很简单，不多说了！

## 4.3 next

返回下一个元素！

```java
public S next() {
    // Android-changed: do not use legacy security code
    /* if (acc == null) { */
        //【-->4.3.1】调用另外一个方法：
        return nextService();
    /*
    } else {
        PrivilegedAction<S> action = new PrivilegedAction<S>() {
            public S run() { return nextService(); }
        };
        return AccessController.doPrivileged(action, acc);
    }
    */
}
```

继续看：

### 4.3.1 nextService

返回下一个服务：

```java
        private S nextService() {
            //【-->4.2.1】这里先判断了下是否有下一个元素；
            if (!hasNextService())
                throw new NoSuchElementException();
            String cn = nextName;
            nextName = null;
            Class<?> c = null;
            try {
                //【1】启动类加载器，可以看到，loader 传入的是 ServiceLoader.loader；
                c = Class.forName(cn, false, loader);
            } catch (ClassNotFoundException x) {
                fail(service,
                     // Android-changed: Let the ServiceConfigurationError have a cause.
                     "Provider " + cn + " not found", x);
                     // "Provider " + cn + " not found");
            }
            //【2】判断下是否满足类继承关系，就是说我们显示传入的接口/类，必须是其父类；
            if (!service.isAssignableFrom(c)) {
                // Android-changed: Let the ServiceConfigurationError have a cause.
                ClassCastException cce = new ClassCastException(
                        service.getCanonicalName() + " is not assignable from " + c.getCanonicalName());
                fail(service,
                     "Provider " + cn  + " not a subtype", cce);
                // fail(service,
                //        "Provider " + cn  + " not a subtype");
            }
            try {
                //【3】通过反射创建服务的实例，并将其转为父类的类型：也就是多态；
                S p = service.cast(c.newInstance());
                //【4】将其加入缓存 providers 中；
                providers.put(cn, p);
                //【5】返回实例；
                return p;
            } catch (Throwable x) {
                fail(service,
                     "Provider " + cn + " could not be instantiated",
                     x);
            }
            throw new Error();          // This cannot happen
        }
```



- **Class.cast**

这里调用了 Class.cast 将子类实例的类型转为父类的：

```java
    public T cast(Object obj) {
        if (obj != null && !isInstance(obj))
            throw new ClassCastException(cannotCastMsg(obj));
        return (T) obj;
    }
```

这里的 T 实际上就是我们上面的 com.coolqi.myapplication.ICommonInterface；

```java
    public boolean isInstance(Object obj) {
        if (obj == null) {
            return false;
        }
        //【1】判断了下当前类是否是对象的类的父类；
        return isAssignableFrom(obj.getClass());
    }

```



# 5 总结

