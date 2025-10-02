# ARouter 第一篇 - 基本使用
title: ARouter 第一篇 - 基本使用
date: 2019/04/13 20:46:25 
categories: 
- 开源库源码分析 
- ARouter
tags: ARouter
copyright: true
---

本系列文章主要分析 ARouter 框架的原理。

**这篇文章** 里面的一些内容来自：

> https://github.com/alibaba/ARouter/blob/master/README_CN.md

官网对其使用已经总结的很好了，本篇博文在其基础上整理了下～～

# 1 简单介绍

对于 ARouter 大家只要做过模块化开发，那么就一定有所了解，ARouter 是阿里巴巴开源的一款路由框架，用于解决模块化开发中的模块依赖。

<br/>

## 2.1 主要模块

官方提供了下面的四个插件模块：

- arouter-api：对外提供功能相关的 Api；
- arouter-compiler：用于解析注解，生成代码；
- arouter-register：用于 App 加固时的自动注册；
- arouter-idea-plugin：Idea 插件，用于关联路径和目标类；

<br/>

## 2.2 功能介绍

官方文档中讲到 ARouter 支持如下的功能：

1. **支持直接解析标准 URL 进行跳转，并自动注入参数到目标页面中；**
2. **支持多模块工程使用；**
3. **支持添加多个拦截器，自定义拦截顺序；**
4. **支持依赖注入，可单独作为依赖注入框架使用；**
5. **支持 InstantRun；**
6. **支持 MultiDex；** (Google 方案)
7. 映射关系按组分类、多级管理，按需初始化；
8. **支持用户指定全局降级与局部降级策略**；
9. 页面、拦截器、服务等组件均自动注册到框架；
10. **支持多种方式配置转场动画**；
11. **支持获取 Fragment**；
12. **完全支持 Kotlin 以及混编**；
13. **支持第三方 App 加固；**(使用 arouter-register 实现自动注册)
14. **支持生成路由文档；**
15. **提供 IDE 插件便捷的关联路径和目标类；**

<br/>

- 当然，我们后面通过源码分析；

# 2 ARouter 使用（官网整理）

以下内容来自对 https://github.com/alibaba/ARouter/edit/master/README_CN.md 的整理：

## 2.1 Gradle 配置

```java
android {
    defaultConfig {
        ...
        javaCompileOptions {
            annotationProcessorOptions {
                arguments = [AROUTER_MODULE_NAME: project.getName()]
            }
        }
    }
}

dependencies {
    compile 'com.alibaba:arouter-api:x.x.x'
    annotationProcessor 'com.alibaba:arouter-compiler:x.x.x'
    ...
}
```
这里两个库均要使用最新版本，防止兼容问题发生；


## 2.2 基本使用

### 2.2.1 添加注解

```java
@Route(path = "/test/activity")
public class YourActivity extend Activity {
    ...
}
```

### 2.2.2 初始化操作

```java
//【1】这两行必须写在init之前，否则这些配置在init过程中将无效
if (isDebug()) {      
    //【2】打印日志
    ARouter.openLog();
    //【3】开启调试模式(如果在InstantRun模式下运行，必须开启调试模式！
    // 线上版本需要关闭,否则有安全风险)
    ARouter.openDebug();
}
ARouter.init(mApplication); // 尽可能早，推荐在Application中初始化
```

### 2.2.3 路由跳转

```java
//【1】应用内简单的跳转(通过URL跳转在'进阶用法'中)
ARouter.getInstance().build("/test/activity").navigation();

//【3】跳转并携带参数
ARouter.getInstance().build("/test/1")
            .withLong("key1", 666L)
            .withString("key3", "888")
            .withObject("key4", new Test("Jack", "Rose"))
            .navigation();
```

## 2.3 进阶使用

### 2.3.1 通过 URL 跳转

我们除了可以使用 @Route 方式指定 path 来跳转，我们还可以通过 url 跳转：

当通过 URL 跳转时，URL 中不能传递 Parcelable 类型数据，通过 ARouter api 才能传递 Parcelable 对象；

``` java
// 新建一个 Activity 用于监听 Schame 事件, 
// 之后直接把 url 传递给 ARouter 即可；
public class SchameFilterActivity extends Activity {
    @Override
    protected void onCreate(Bundle savedInstanceState) {
    super.onCreate(savedInstanceState);

    Uri uri = getIntent().getData();
    ARouter.getInstance().build(uri).navigation();
    finish();
    } 
}
```

- 说明书 AndroidManifest.xml 中的配置：

``` xml
<activity android:name=".activity.SchameFilterActivity">
    <!-- Schame -->
    <intent-filter>
        <data
        android:host="m.aliyun.com"
        android:scheme="arouter"/>

        <action android:name="android.intent.action.VIEW"/>

        <category android:name="android.intent.category.DEFAULT"/>
        <category android:name="android.intent.category.BROWSABLE"/>
    </intent-filter>
</activity>
```

### 2.3.2 自动解析参数

自动解析参数。

为每一个参数声明一个字段，并使用 @Autowired 标注，我们传递的值将会自动赋值给所属变量。

``` java
@Route(path = "/test/activity")
public class Test1Activity extends Activity {
    @Autowired
    public String name;
    @Autowired
    int age;
    
    //【1】通过 name 来映射 URL 中的不同参数
    @Autowired(name = "girl") 
    boolean boy;
    
    //【2】支持解析自定义对象，URL中使用 json 传递
    @Autowired
    TestObj obj;      
    
    //【3】使用 withObject 传递 List 和 Map 的实现了
    // Serializable 接口的实现类(ArrayList/HashMap)
    // 的时候，接收该对象的地方不能标注具体的实现类类型
    // 应仅标注为 List 或 Map，否则会影响序列化中类型
    // 的判断, 其他类似情况需要同样处理        
    @Autowired
    List<TestObj> list;
    @Autowired
    Map<String, List<TestObj>> map;
    
    @Override
    protected void onCreate(Bundle savedInstanceState) {
    super.onCreate(savedInstanceState);
    //【4】这里很关键，执行 ativity 的注入；
    ARouter.getInstance().inject(this);
    }
}

```

如果需要传递自定义对象（比如上面的 TestObj），新建一个类，实现 SerializationService, 并使用 @Route 注解标注！

这个类的作用是自定义对象的序列化方式：

```java
@Route(path = "/yourservicegroupname/json")
public class JsonServiceImpl implements SerializationService {
    @Override
    public void init(Context context) {

    }

    @Override
    public <T> T json2Object(String text, Class<T> clazz) {
        return JSON.parseObject(text, clazz);
    }

    @Override
    public String object2Json(Object instance) {
        return JSON.toJSONString(instance);
    }
}
```

这里使用的是 JSON 序列化。

### 2.3.3 跳转拦截器

拦截器用于拦截跳转过程，面向切面编程，比较经典的应用就是在跳转过程中处理登陆事件，这样就不需要在目标页重复做登陆检查！

拦截器会在跳转之间执行，多个拦截器会按优先级顺序依次执行！

``` java
@Interceptor(priority = 8, name = "测试用拦截器")
public class TestInterceptor implements IInterceptor {
    @Override
    public void process(Postcard postcard, InterceptorCallback callback) {
        ...
        // 处理完成，交还控制权
        callback.onContinue(postcard); 
        // 觉得有问题，中断路由流程，以上两种至少需要调用其中一种，否则不会继续路由
        // callback.onInterrupt(new RuntimeException("我觉得有点异常")); 
    }

    @Override
    public void init(Context context) {
        //【1】拦截器的初始化，会在 sdk 初始化的时候调用该方法，仅会调用一次
    }
}
```
上面说的初始化，就是 ARouter.init 方法；

### 2.3.4 处理跳转结果

navigation 方法支持传入一个 **NavigationCallback 回调**，处理跳转结果：

``` java
ARouter.getInstance().build("/test/1").navigation(this, new NavigationCallback() {
    @Override
    public void onFound(Postcard postcard) {
    ...
    }

    @Override
    public void onLost(Postcard postcard) {
    ...
    }
});
```

### 2.3.5 自定义全局降级策略

自定义类，实现 DegradeService 接口，并加上一个 Path 内容任意的注解：

``` java
@Route(path = "/xxx/xxx")
public class DegradeServiceImpl implements DegradeService {
    @Override
    public void onLost(Context context, Postcard postcard) {
        // do something.
    }
    
    @Override
    public void init(Context context) {
    }
}
```

### 2.3.6 为目标页面声明更多信息

@Route 还有一个 extras 属性，用于设置一些额外的属性，他是一个 int 值，有 32 位，可以配置 32 个开关；

我们可以通过设置指定的开关位，然后在拦截器中可以拿到这个标记进行业务逻辑判断！

``` java
@Route(path = "/test/activity", extras = Consts.XXXX)
public class Test1Activity extends Activity {
    ... ... ...
}
```

### 2.3.7 依赖注入解耦

ARouter 通过定义统一的访问接口来实现解耦 module 依赖；

核心接口：IProvider！

#### 2.3.7.1 暴露服务

``` java
// 声明接口,其他组件通过接口来调用服务
public interface HelloService extends IProvider {
    //【1】module 间通信接口； 
    String sayHello(String name);
}

//【2】实现接口，也是我们实际要暴漏的服务；
@Route(path = "/yourservicegroupname/hello", name = "测试服务")
public class HelloServiceImpl implements HelloService {

    @Override
    public String sayHello(String name) {
    return "hello, " + name;
    }

    @Override
    public void init(Context context) {

    }
}
```
暴漏的服务，需要通过 @Route 注解去修饰！

#### 2.3.7.2 访问服务

当我们暴漏了服务后，需要

``` java
public class Test {
    //【1】依赖注入的方式
    @Autowired
    HelloService helloService;
    @Autowired(name = "/yourservicegroupname/hello")
    HelloService helloService2;
    //【2】依赖查找的方式
    HelloService helloService3;
    HelloService helloService4;

    public Test() {
        //【3】依赖注入的方式；
        ARouter.getInstance().inject(this);
    }
    
    public void testService() {
        ... ... ...
    }
}
```
访问服务的方式有下面两种方式：

- **使用依赖注入的方式发现服务**

这也是推荐的方式，通过注解标注字段, 即可使用，无需主动获取

Autowired 注解标注 name 之后，将会使用 **byName** 的方式注入对应的字段；不设置 name 属性，会默认使用**byType** 的方式发现服务(当同一接口有多个实现的时候，必须使用 byName 的方式发现服务)

```java
helloService.sayHello("Vergil");
helloService2.sayHello("Vergil");
```

- **使用依赖查找的方式发现服务**

使用依赖查找的方式发现服务，主动去发现服务并使用，也有 byName 和 byType 两种方式：

```java
helloService3 = ARouter.getInstance().navigation(HelloService.class);
helloService4 = (HelloService) ARouter.getInstance().build("/yourservicegroupname/hello").navigation();
helloService3.sayHello("Vergil");
helloService4.sayHello("Vergil");
```

### 2.3.8 预处理服务

预处理服务和拦截器的概念很类似：

``` java
// 实现 PretreatmentService 接口，并加上一个Path内容任意的注解即可
@Route(path = "/xxx/xxx")
public class PretreatmentServiceImpl implements PretreatmentService {
    @Override
    public boolean onPretreatment(Context context, Postcard postcard) {
        // 跳转前预处理，如果需要自行处理跳转，该方法返回 false 即可
    }

    @Override
    public void init(Context context) {

    }
}
```

# 3 源码结构

我们来看下 ARouter 的源码结构，下面列出关键的目录：

```xml
|-ARouter-master.iml      
|-README_CN.md            
|-arouter-api             
|-arouter-idea-plugin
|-arouter-compiler
|-arouter-annotation
|-arouter-gradle-plugin
|- ... ... ...
```

这里分别解释下每个 module 的作用：

- **arouter-annotation**：
    - 定义了 ARouter 使用到的所有的注解；
- **arouter-api**：
    - **对应 "arouter-api" 插件**，对外提供功能相关的 Api；
- **arouter-compiler**：
    - **对应 "arouter-compiler" 插件**，用于解析注解，生成代码；
- **arouter-gradle-plugin**：
    - **对应 "arouter-register" 插件**，用于 App 加固时的自动注册；
- **arouter-idea-plugin**：
    - **对应 "arouter-idea-plugin" 插件**，用于关联路径和目标类；


## 3.1 Module 依赖关系

下图我们来看看这几个 module 的依赖关系：

![](leanote://file/getImage?fileId=5d3ec537ab6441734a002e54)

依赖关系还是很简单的，毕竟只有几个 module。


# 4 总结

本篇博文整理了下 ARouter 官网的一些内容，总结了 ARouter 的基本使用和进阶使用，接下来，会通过分析每个 module 的源码，来进一步分析 ARouter 的原理！