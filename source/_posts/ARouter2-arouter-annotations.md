# ARouter 第二篇 - 注解定义(arouter-annotation)
title: ARouter 第二篇 - 注解定义 (arouter-annotation)
date: 2019/04/17 20:46:25 
catalog: true
categories: 
- 开源库源码分析 
- ARouter
tags: ARouter
copyright: true
------

本系列文章主要分析 ARouter 框架的架构和原理。

> 这是阿里 ARouter 开源库的地址，大家可以直接访问
> https://github.com/alibaba/ARouter

本篇博文主要分析 arouter-annotation 模块；

# 1 模块结构

下面我们来看看 arouter-annotation 的结构；

```java
src -> main -> java:
|____com
| |____alibaba
| | |____android
| | | |____arouter
| | | | |____facade
| | | | | |____enums
| | | | | | |____RouteType.java
| | | | | | |____TypeKind.java
| | | | | |____annotation
| | | | | | |____Interceptor.java
| | | | | | |____Route.java
| | | | | | |____Param.java
| | | | | | |____Autowired.java
| | | | | |____model
| | | | | | |____TypeWrapper.java
| | | | | | |____RouteMeta.java
```

一共有三个 package：

- enums：包含了一些枚举类：
- annotation：包含了一些注解；
- model：包含了一些跳转所需的数据；

# 2 源码分析

## 2.1 enums

这个 package 包含了一些枚举类：

### 2.1.1 RouteType

枚举类，每一个成员都用于保存 id 和 className 的映射！

className 包括 android 的 Activity，Service，ContentProvider，Fragment，以及 ARouter 自己的 IProvider 类型！

```java
public enum RouteType {
    ACTIVITY(0, "android.app.Activity" ),
    SERVICE(1, "android.app.Service"),
    PROVIDER(2, "com.alibaba.android.arouter.facade.template.IProvider"),
    CONTENT_PROVIDER(-1, "android.app.ContentProvider"),
    BOARDCAST(-1, ""),
    METHOD(-1, ""),
    FRAGMENT(-1, "android.app.Fragment"),
    UNKNOWN(-1, "Unknown route type");

    int id;
    String className;

    public int getId() {
        return id;
    }

    public RouteType setId(int id) {
        this.id = id;
        return this;
    }

    public String getClassName() {
        return className;
    }

    public RouteType setClassName(String className) {
        this.className = className;
        return this;
    }

    RouteType(int id, String className) {
        this.id = id;
        this.className = className;
    }

    public static RouteType parse(String name) {
        for (RouteType routeType : RouteType.values()) {
            if (routeType.getClassName().equals(name)) {
                return routeType;
            }
        }

        return UNKNOWN;
    }
}
```

代码很简单，不多说。

它的作用是，我们可以通过 RouteType 判断判断 @Route 修饰的是类是那种类型；

### 2.1.2 TypeKind

枚举类，每一个枚举成员都用于表示一个类型，包括基本类型，可序列化类型，字符串，对象等等；

```java
public enum TypeKind {
    // Base type
    BOOLEAN,
    BYTE,
    SHORT,
    INT,
    LONG,
    CHAR,
    FLOAT,
    DOUBLE,

    // Other type
    STRING,
    SERIALIZABLE,
    PARCELABLE,
    OBJECT;
}
```

它的作用是在设置跳转数据的时候，通过 TypeKind 来判断数据的类型，然后调用 Postcard.withXXX 方法，设置不同的类型；

## 2.2 annotation

这个 package 包含了一些注解类：

### 2.2.1 Route

用于注解 RouteType 指定的那些 type：

```java
@Target({ElementType.TYPE})
@Retention(RetentionPolicy.CLASS)
public @interface Route {
    //【1】用于指定路由跳转的路径，至少包含两级目录；
    String path();
    //【2】用于指定路由跳转的分组，组名必须要使用相同的名称；
    String group() default "";
    //【3】路由跳转的名称，用于 javadoc；
    String name() default "";
    //【4】用于指定额外的数据，一共 32 位，每一位都是一个开关；
    int extras() default Integer.MIN_VALUE;
    //【5】路由跳转的优先级，值越小，优先级越高；
    int priority() default -1;
}
```

不多说了。

#### 2.2.1.1 简单使用

- **注解服务**

```java
@Route(path = "/xxx/xxx")
public class PretreatmentServiceImpl implements PretreatmentService {
    ... ... ...
}
```

- **注解 android 组件**

```
@Route(path = "/test/activity")
public class Test1Activity extends Activity {
    ... ... ...
}
```

### 2.2.2 Autowired

用于修饰成员变量：

```java
@Target({ElementType.FIELD})
@Retention(RetentionPolicy.CLASS)
public @interface Autowired {

    //【1】变量（属性或服务）的名称
    String name() default "";

    //【2】是否必须不为 null，如果为 true，应用会 crash 当其为 null 的情况；
    // private 类型的变量不会检查是否为 null；
    boolean required() default false;

    //【2】属性的描述
    String desc() default "";
}
```

#### 2.2.2.1 简单使用

- **注解服务**

```java
@Route(path = "/xxx/xxx")
public class PretreatmentServiceImpl implements PretreatmentService {
    ... ... ...
}
```

- **注解 android 组件**

```
@Route(path = "/test/activity")
public class Test1Activity extends Activity {
    ... ... ...
}
```

### 2.2.3 Interceptor

用于注解拦截器：

```java
@Target({ElementType.TYPE})
@Retention(RetentionPolicy.CLASS)
public @interface Interceptor {
    //【1】拦截器的优先级，ARouter 会根据优先级执行拦截器；
    int priority();

    //【2】拦截器的名称，用于 javadoc；
    String name() default "Default";
}
```

#### 2.2.3.1 简单使用

- **注解拦截器**

```java
@Interceptor(priority = 8, name = "测试用拦截器")
public class TestInterceptor implements IInterceptor {
    ... ... ...
}
```

### 2.2.4 Param（DEPRECATED）

这个注解也是用来修饰成员变量的，但是不推荐使用了，请使用 Autowired！

```java
@Target({ElementType.FIELD})
@Retention(RetentionPolicy.CLASS)
@Deprecated
public @interface Param {
    //【1】属性的名称；
    String name() default "";

    //【2】属性的描述
    String desc() default "No desc.";
}
```

因为已经不在推荐使用，不多说了！

## 2.3 model

这个 package 下主要保存了一些数据类，这些类保存了跳转需要的数据，已经目前对象的类型：

### 2.3.1 RouteMeta

用于保存路由跳转的数据：

```java
public class RouteMeta {
    private RouteType type;         //【1】路由的类型，枚举类实例；
    private Element rawType;        //【2】@Route 注解修饰的元素；
    private Class<?> destination;   //【3】路由跳转的目标类
    private String path;            //【4】路由跳转的路径 path
    private String group;           //【5】路由跳转的组 group
    private int priority = -1;      //【6】路由跳转的优先级，值越小，优先级越高；
    private int extra;              //【7】路由跳转携带的额外数据，23 位开关；
    private Map<String, Integer> paramsType;  //【8】(Autowired 注解的属性) 保存 fieldName/Autowired.name --> 属性类型对应的枚举序号
    private String name; //【9】路由跳转的名称；

    private Map<String, Autowired> injectConfig;  //【10】(Autowired 注解的属性) 保存 fieldName/Autowired.name --> 对应的 Autowired 实例

    public RouteMeta() {
    }
    
    ... ... ... ...// 这里我们先省略内部方法，后续分析！ 
}
```

其实可以看到，RouteMeta 内部的数据很多事 compiler 解析 Route、Autowired 注解获得的！

### 2.3.2 TypeWrapper<T>

用于保存路由跳转的目标对象类型：

```java
public class TypeWrapper<T> {
    //【1】用于保存泛型 T；
    protected final Type type;
    
    protected TypeWrapper() {
        //【2】调用 getClass() 获得当前类的 class 对象；
        //【3】然后再调用 getGenericSuperclass() 获得带有泛型的父类；
        Type superClass = getClass().getGenericSuperclass();
        //【4】将 superClass 强转为 ParameterizedType 类型；
        //【5】getActualTypeArguments() 返回表示此类型实际类型参数的 Type 对象的数组；
        //【6】[0] 就是这个数组中第一个了，简而言之就是获得超类的泛型参数的实际类型
        type = ((ParameterizedType) superClass).getActualTypeArguments()[0];
    }

    public Type getType() {
        return type;
    }
}
```

TypeWrapper<T> 是一个泛型类，泛型 T 表示目标对象类型！

后面我们再具体分析。

# 4 总结

本篇文章分析了 ARouter 中的 arouter-annotation 模块，其内部定义了 ARouter 必须的注解类，数据类，已经枚举类。

下篇文章将分析 arouter-compiler 模块，探寻在 App 编译期间，Gradle 事如何使用 arouter-compiler  对注解进行解析，和动态生成中间类的！