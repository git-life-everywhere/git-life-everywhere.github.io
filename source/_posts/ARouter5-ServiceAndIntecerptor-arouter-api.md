# ARouter 第五篇 - 服务和拦截器 (arouter-api)
title: ARouter 第五篇 - 服务和拦截器 (arouter-api)
date: 2019/04/25 20:46:25 
catalog: true
categories: 
- 开源库源码分析 
- ARouter
tags: ARouter
copyright: true
------
本系列文章主要分析 ARouter 框架的架构和原理。

> 这是阿里 ARouter 开源库的地址，大家可以直接访问
>
> https://github.com/alibaba/ARouter

本篇博文主要分析 arouter-api 模块，该模块涉及到 ARouter 一些核心逻辑：初始化，跳转，拦截，服务等，下面的几篇文章就要从这几个方向来分析；

在阅读过程中，涉及到方法跳转的时候，注释上有 `-->`的标志，这样的好处是，以类为单位，一次性分析其所有的方法！

# 1 开篇

这篇文章分析下 ARouter 的 Service 和 Interceptor，二者有如下的区别：

## 1.1 实现接口不同

- Service 直接或者间接的实现了 IProvider 接口：

```java
public interface IProvider {
    void init(Context context);
}
```

- Interceptor 直接或者间接的实现了 IInterceptor 接口：

```java
public interface IInterceptor extends IProvider {
    void process(Postcard postcard, InterceptorCallback callback);
}
```

## 1.2 注解不同

- Service 使用 @Route  注解处理：

```java
@Route(path = "/yourservicegroupname/single")
public class SingleService implements IProvider {
}
```

- Interceptor 使用 @Interceptor  注解处理：

```java
@Interceptor(priority = 7)
public class Test1Interceptor implements IInterceptor {
}
```



## 1.3 逻辑处理不同

- 拦截器会在 ARouter 初始化 init 的时候异步初始化，如果第一次路由的时候拦截器还没有初始化结束，路由会等待，直到初始化完成。
		- 这个下面可以看到，内部有一个同步锁来控制；
- 服务没有该限制，某一服务可能在 App 整个生命周期中都不会用到，所以服务只有被调用的时候才会触发初始化操作；

# 1 服务 Service



## 1.1 服务统一接口

ARouter 已经帮我们提供了一些 Service 统一接口，对于对内对外提供特定的功能模版：

```shell
-rwxr-xr-x  1 lishuaiqi820  235765416  468 May  8 20:46 AutowiredService.java
-rwxr-xr-x  1 lishuaiqi820  235765416  424 May  8 20:46 ClassLoaderService.java
-rwxr-xr-x  1 lishuaiqi820  235765416  590 May  8 20:46 DegradeService.java
-rwxr-xr-x  1 lishuaiqi820  235765416  575 May  8 20:46 InterceptorService.java
-rwxr-xr-x  1 lishuaiqi820  235765416  555 May  8 20:46 PathReplaceService.java
-rwxr-xr-x  1 lishuaiqi820  235765416  656 May  8 20:46 PretreatmentService.java
-rwxr-xr-x  1 lishuaiqi820  235765416  974 May  8 20:46 SerializationService.java
```

- **AutowiredService**：用于处理 Autowired 注解的变量的 Service，ARouter 内置了一个 AutowiredServiceImpl 实现了 AutowiredService，我们在分析 inject 的时候，再讲；
- **ClassLoaderService**：针对于 installrun 的 Service；
- **DegradeService**：用于在跳转不成功的情况下，做降级处理；
- **InterceptorService**：用于处理 Interceptor 的 Service，ARouter 内置了一个 InterceptorServiceImpl 实现了 InterceptorService，用于初始化所有的  Interceptor 和处理拦截，我们下面分析；
- **PathReplaceService**：用于对路由的 path 做预处理；
- **PretreatmentService**；用于在跳转之前做预处理操作；
- **SerializationService**：用于序列化 Object 对象，和 Autowired 注解配合使用，我们在分析 inject 的时候，再讲；



## 1.2 获取服务

ARouter 是通过路由跳转的方式获取服务的，我们来回顾 init 的流程：

- 获取拦截器处理服务：

```java
PathReplaceService pService = ARouter.getInstance().navigation(PathReplaceService.class);
```

- 获取拦截器处理服务：

```java
interceptorService = (InterceptorService) ARouter.getInstance().build("/arouter/service/interceptor").navigation();
```

之前在分析 init 的过程中，我们又遇到好几个获取 Service 的地方，上面是举了其中几个栗子！

- 获取序列化服务：


```java
serializationService = ARouter.getInstance().navigation(SerializationService.class);
```

  上面的代码是在处理 @Autowired 注解的时候，也就是 arouter complier 编译的时候处理的，用于传递自定义的对象；



## 1.3 获取流程分析

通过上面可以知道，获取一个 Service 的方法有两种：

```java
ARouter.getInstance().navigation(xxxx.class);

ARouter.getInstance().build(path).navigation();
```

下面我们会分析下这两种方式的流程！

**有些代码在前面的路由处理过程中分析了，这里不会再重复分析。**

### 1.3.1 navigation(className.class)

第一种方式是传入 Service 的父类，我们回顾下**调用链**：

```java
ARouter.getInstance().navigation(service.class);
		_ARouter.getInstance().navigation(service);
		    Postcard postcard = LogisticsCenter.buildProvider(service.getName());
               Postcard postcard = Warehouse.providersIndex.get(serviceName);
        LogisticsCenter.completion(postcard);
        _ARouter.getInstance()._navigation(...);
```

上面这部分的调用过程实际上，我们在路由初始化的时候见到过！

这里我们只看核心的逻辑，省略掉一些奇葩的

#### 1.3.1.1 _ARouter.navigation

我回顾下 _ARouter.navigation 方法；

```java
protected <T> T navigation(Class<? extends T> service) {
    try {
        //【-->1.3.1.2】通过 serviceName 找到，对应的 Service 的 RouteMeta 实例，然后创建 Postcard 实例
        // service.getName() 返回的是全限定名；
        Postcard postcard = LogisticsCenter.buildProvider(service.getName());
  
        //【1】如果是 null，说明使用的是旧版本的 compiler sdk，早期的 compiler 不使用全限定名区获取服务；
        if (null == postcard) {
            //【-->1.3.1.2】通过 serviceName 找到，对应的 Service 的 RouteMeta 实例，然后创建 Postcard 实例
            postcard = LogisticsCenter.buildProvider(service.getSimpleName());
        }
  
        if (null == postcard) {
            return null;
        }
        //【-->1.3.1.3】完成跳转！
        LogisticsCenter.completion(postcard);
      
        //【2】获取 Serivce；
        return (T) postcard.getProvider();
    } catch (NoRouteFoundException ex) {
        logger.warning(Consts.TAG, ex.getMessage());
        return null;
    }
}
```

这里的核心处理：

- LogisticsCenter.buildProvider
- LogisticsCenter.completion

#### 1.3.1.2 LogisticsCenter.buildProvider

```java
public static Postcard buildProvider(String serviceName) {
    //【1】我们知道 service 实现了 IProvider 实例，所以保存在了 Warehouse.providersIndex 中！
    RouteMeta meta = Warehouse.providersIndex.get(serviceName);

    if (null == meta) {
        return null;
    } else {
        //【2】创建路由跳转信息；
        return new Postcard(meta.getPath(), meta.getGroup());
    }
}
```

这里的 Warehouse.providersIndex 保存的是如下的数据：

```java
providers.put("com.alibaba.android.arouter.facade.service.SerializationService", RouteMeta.build(RouteType.PROVIDER, MySerializationService.class, 
                            "/coolqiService/MySerializationService", "coolqiService", null, -1, -2147483648));
```

#### 1.3.1.3 LogisticsCenter.completion

```java
public synchronized static void completion(Postcard postcard) {
    if (null == postcard) {
        throw new NoRouteFoundException(TAG + "No postcard!");
    }
    //【1】从 Warehouse.routes 中获取 path 对应的 RouteMeta 缓存数据；
    RouteMeta routeMeta = Warehouse.routes.get(postcard.getPath());
    //【2】如果找不到，那么就从 compiler 生成的数据中查找！
    if (null == routeMeta) { 
        //【2.1】从 Warehouse.routes 中获取 group 对应的 group 类文件；
        Class<? extends IRouteGroup> groupMeta = Warehouse.groupsIndex.get(postcard.getGroup());
        if (null == groupMeta) { // 【2.1.1】找不到抛出异常；
            throw new NoRouteFoundException(TAG + "There is no route match the path [" + postcard.getPath() + "], in group [" + postcard.getGroup() + "]");
        } else {

            try {
                if (ARouter.debuggable()) {
                    logger.debug(TAG, String.format(Locale.getDefault(), "The group [%s] starts loading, trigger by [%s]", postcard.getGroup(), postcard.getPath()));
                }
                //【2.1.2】创建 groupMeta 对应的实例；
                IRouteGroup iGroupInstance = groupMeta.getConstructor().newInstance();
                //【2.1.3】调用其 loadInto 将 group 对应的信息加入到缓存 Warehouse.routes 中！
                iGroupInstance.loadInto(Warehouse.routes);
                //【2.1.4】然后从 Warehouse.groupsIndex 删除这个组对应的信息；
                Warehouse.groupsIndex.remove(postcard.getGroup());

                if (ARouter.debuggable()) {
                    logger.debug(TAG, String.format(Locale.getDefault(), "The group [%s] has already been loaded, trigger by [%s]", postcard.getGroup(), postcard.getPath()));
                }
            } catch (Exception e) {
                throw new HandlerException(TAG + "Fatal exception when loading group meta. [" + e.getMessage() + "]");
            }
            //【-->1.3.1.3】重新加载；
            completion(postcard);
        }
    } else {
        //【3】这里是通过 RouteMeta 来设置 Postcard 对象，我们先不看；
        postcard.setDestination(routeMeta.getDestination());
        postcard.setType(routeMeta.getType()); 
        postcard.setPriority(routeMeta.getPriority());
        postcard.setExtra(routeMeta.getExtra());
        Uri rawUri = postcard.getUri();
        if (null != rawUri) { 
            Map<String, String> resultMap = TextUtils.splitQueryParameters(rawUri);
            Map<String, Integer> paramsType = routeMeta.getParamsType(); 

            if (MapUtils.isNotEmpty(paramsType)) {
                for (Map.Entry<String, Integer> params : paramsType.entrySet()) {
                    setValue(postcard,
                            params.getValue(),
                            params.getKey(),
                            resultMap.get(params.getKey()));
                }
                postcard.getExtras().putStringArray(ARouter.AUTO_INJECT, paramsType.keySet().toArray(new String[]{}));
            }
            postcard.withString(ARouter.RAW_URI, rawUri.toString());
        }

        //【4】这里是关键点，判断类型，可以看到 activity 这里是不处理的！；
        switch (routeMeta.getType()) {
            case PROVIDER: 
                //【4.1】我们要获取的 Serivce，类型就是 PROVIDER，routeMeta.getDestination 返回的是要访问的目标类：service.class;
                Class<? extends IProvider> providerMeta = (Class<? extends IProvider>) routeMeta.getDestination();
                //【4.2】然后优先从 Warehouse.providers 缓存中获取；
                IProvider instance = Warehouse.providers.get(providerMeta);
                if (null == instance) {
                    IProvider provider;
                    try {
                        //【4.3】创建 providerMeta 对应的实例，就是 Service 实例；
                        provider = providerMeta.getConstructor().newInstance();
                        //【4.3.1】执行 init 方法；
                        provider.init(mContext);
                        //【4.3.2】然后将加入到 Warehouse.providers 中去；
                        Warehouse.providers.put(providerMeta, provider);
                        instance = provider;
                    } catch (Exception e) {
                        throw new HandlerException("Init provider failed! " + e.getMessage());
                    }
                }
                //【5】将 instance 保存到 postcard.provider 中去，因为跳转目标是 IProvider 的子类；
                postcard.setProvider(instance);
                postcard.greenChannel();  //【6】跳过所有的拦截器！
                break;
            case FRAGMENT:
                postcard.greenChannel(); // 跳过所有的拦截器！
            default:
                break;
        }
    }
}
```

这里有所谓的 “按组加载”

可以看到，最后获取了 Service，并调用了其 init 方法；

最后将获得的 Service 保存到了 Postcard 中；

### 1.3.2 build(path).navigation()

第二种方式是通过 path 来查找 Service，我们回顾下**调用链**：

```java
ARouter.getInstance().build(path).navigation() 
      Postcard postcard = _ARouter.getInstance().build(path)
      Object object = Postcard.navigation();
           Object object = ARouter.getInstance().navigation(context, this, -1, callback)
           			Object object = _ARouter.getInstance().navigation(mContext, postcard, requestCode, callback)      
										 LogisticsCenter.completion(postcard);       
                     Object object = _ARouter.getInstance()._navigation(...);
```

上面这部分的调用过程实际上，我们在路由初始化的时候见到过！

这里我们只看核心的逻辑，省略掉一些非核心的代码；

#### 1.3.2.1 _ARouter.navigation

```java
protected Object navigation(final Context context, final Postcard postcard, final int requestCode, final NavigationCallback callback) {
    //【1】这里的获取方式是一样的；
    PretreatmentService pretreatmentService = ARouter.getInstance().navigation(PretreatmentService.class);
    if (null != pretreatmentService && !pretreatmentService.onPretreatment(context, postcard)) {
        return null;
    }

    try {
        //【-->1.3.1.3】完善跳转信息！
        LogisticsCenter.completion(postcard);
    } catch (NoRouteFoundException ex) {
        logger.warning(Consts.TAG, ex.getMessage());

        if (debuggable()) {
            runInMainThread(new Runnable() {
                @Override
                public void run() {
                    Toast.makeText(mContext, "There's no route matched!\n" +
                            " Path = [" + postcard.getPath() + "]\n" +
                            " Group = [" + postcard.getGroup() + "]", Toast.LENGTH_LONG).show();
                }
            });
        }

        if (null != callback) {
            callback.onLost(postcard);
        } else {
            //【2】这里的获取方式是一样的；
            DegradeService degradeService = ARouter.getInstance().navigation(DegradeService.class);
            if (null != degradeService) {
                degradeService.onLost(context, postcard);
            }
        }

        return null;
    }

    if (null != callback) {
        callback.onFound(postcard);
    }
    //【3】对于 Service 是会跳过拦截器的；
    if (!postcard.isGreenChannel()) {
        interceptorService.doInterceptions(postcard, new InterceptorCallback() {

            @Override
            public void onContinue(Postcard postcard) {
                //【-->1.3.2.2】最终的处理；
                _navigation(context, postcard, requestCode, callback);
            }

            @Override
            public void onInterrupt(Throwable exception) {
                if (null != callback) {
                    callback.onInterrupt(postcard);
                }
                logger.info(Consts.TAG, "Navigation failed, termination by interceptor : " + exception.getMessage());
            }
        });
    } else {
        //【-->1.3.2.2】最终的处理；
        return _navigation(context, postcard, requestCode, callback);
    }

    return null;
}
```

#### 1.3.2.4 \_ARouter._navigation

最终处理：

```java
 private Object _navigation(final Context context, final Postcard postcard, final int requestCode, final NavigationCallback callback) {
     final Context currentContext = null == context ? mContext : context;
 
     switch (postcard.getType()) {
         case ACTIVITY:
             ... ... ...
             break;
         case PROVIDER:
             //【1】返回了 iprovider 实例，就是我们的 Service 对象；
             return postcard.getProvider();
         case BOARDCAST:
         case CONTENT_PROVIDER:
         case FRAGMENT:
             ... ... ...
         case METHOD:
         case SERVICE:
         default:
             return null;
     }
 
     return null;
 }
```

就这样，我们获得了 Service 对象！



## 1.4 内置服务 


我们来看看内置服务接口！

对与 AutowiredService，InterceptorService，SerializationService 我们后面会分析，这里就不重点分析了，累！
### 1.4.1 DegradeService

降级服务，当跳转失败后，可以在这里做处理：

```java
public interface DegradeService extends IProvider {

    void onLost(Context context, Postcard postcard);
}
```

### 1.4.2 PathReplaceService

路径 path 替换服务，我们可以在启动跳转之前，对 path 进行拦截，替换新的 path： 

```java
public interface PathReplaceService extends IProvider {
    String forString(String path);
    Uri forUri(Uri uri);
}
```

可以针对 path 和 uri 两种方式！

### 1.4.3 PretreatmentService

跳转预处理服务，我们可以在启动跳转之前，针对跳转路由数据做预处理：

```java
public interface PathReplaceService extends IProvider {
    String forString(String path);
    Uri forUri(Uri uri);
}
```

可以针对 path 和 uri 两种方式！


# 2 拦截器 Interceptor

## 2.1 InterceptorServiceImpl - 统一管理拦截器 

在 ARouter 框架里面，有一个 InterceptorServiceImpl 服务，用于统一管理 Interceptor：

```java
static void afterInit() {
    interceptorService = (InterceptorService) ARouter.getInstance().build("/arouter/service/interceptor").navigation();
}
```

这里我们就不多说了，这个是获取拦截器管理服务的方式，流程上面分析了；



## 2.2 初始化 Interceptor

Interceptor 的初始化由 InterceptorServiceImpl 完成，

核心的逻辑在 **LogisticsCenter.completion** 中！

###  2.2.1 LogisticsCenter.completion

这里我们省略掉无关的代码：

```java
public synchronized static void completion(Postcard postcard) {
    if (null == postcard) {
        throw new NoRouteFoundException(TAG + "No postcard!");
    }
    RouteMeta routeMeta = Warehouse.routes.get(postcard.getPath());
    if (null == routeMeta) { 
    		... ... ...
    } else {
        ... ... ...

        //【1】这里是关键点，判断类型，可以看到 activity 这里是不处理的！；
        switch (routeMeta.getType()) {
            case PROVIDER: 
                //【2.1】我们要获取的 InterceptorServiceImpl，类型就是 PROVIDER，routeMeta.getDestination 返回的是要访问的目标类：InterceptorServiceImpl.class;
                Class<? extends IProvider> providerMeta = (Class<? extends IProvider>) routeMeta.getDestination();
                //【1.2】然后优先从 Warehouse.providers 缓存中获取；
                IProvider instance = Warehouse.providers.get(providerMeta);
                if (null == instance) {
                    IProvider provider;
                    try {
                        //【1.2.1】创建 providerMeta 对应的实例，就是 InterceptorServiceImpl；
                        provider = providerMeta.getConstructor().newInstance();
                        //【--->2.2.2】执行 init 方法；
                        provider.init(mContext);
                        //【1.2.2】然后将加入到 Warehouse.providers 中去；
                        Warehouse.providers.put(providerMeta, provider);
                        instance = provider;
                    } catch (Exception e) {
                        throw new HandlerException("Init provider failed! " + e.getMessage());
                    }
                }
                //【1.2.3】将 instance 保存到 postcard.provider 中去，因为跳转目标是 IProvider 的子类；
                postcard.setProvider(instance);
                postcard.greenChannel();  //【1.2.4】跳过所有的拦截器！
                break;
            case FRAGMENT:
                postcard.greenChannel(); // 跳过所有的拦截器！
            default:
                break;
        }
    }
}
```

**回顾**：

我们知道在路由初始化的过程中，afterInit 会获得 InterceptorServiceImpl 方法并执行其 init 的初始化操作！

### 2.2.2 InterceptorServiceImpl.init

在 InterceptorServiceImpl 的 init 方法中，会获取所有的 Interceptor，并对其做初始化操作；

```java
@Override
public void init(final Context context) {
    //【1】这里是在由线程池管理的子线程中执行 init 操作；
    LogisticsCenter.executor.execute(new Runnable() {
        @Override
        public void run() {
            if (MapUtils.isNotEmpty(Warehouse.interceptorsIndex)) {
                //【1】从 Warehouse.interceptorsIndex 获取所有注解生成的拦截器；
                for (Map.Entry<Integer, Class<? extends IInterceptor>> entry : Warehouse.interceptorsIndex.entrySet()) {
                    Class<? extends IInterceptor> interceptorClass = entry.getValue();
                    try {
                        //【2】创建 interceptors 实例，并执行 init 初始化；
                        IInterceptor iInterceptor = interceptorClass.getConstructor().newInstance();
                        iInterceptor.init(context);
                        //【3】将其加入到缓存 Warehouse.interceptors
                        Warehouse.interceptors.add(iInterceptor);
                    } catch (Exception ex) {
                        throw new HandlerException(TAG + "ARouter init interceptor error! name = [" 
                                                   + interceptorClass.getName() + "], reason = [" + ex.getMessage() + "]");
                    }
                }

                interceptorHasInit = true; //【4】init 状态设置为 true；

                logger.info(TAG, "ARouter interceptors init over.");

                synchronized (interceptorInitLock) { //【5】当 init 操作完成后 notifyAll 通知等待 init 的线程；
                    interceptorInitLock.notifyAll();
                }
            }
        }
    });
}
```

这里是在由线程池管理的字现场中执行 init 操作！

注意，这里有一个同步锁，如果在路由的时候，发现 interceptorHasInit 为 false，那么会调用 interceptorInitLock.wait 进入阻塞状态，等待初始化完成，被 notifyAll 唤醒！

## 2.3 拦截操作

我们来看看拦截操作是如何做的，核心代码在 _ARouter.navigation 中：

### 2.3.1 \_ARouter.navigation

我们只关注核心的逻辑：

```java
protected Object navigation(final Context context, final Postcard postcard, final int requestCode, final NavigationCallback callback) {
    ... ... ...

    if (null != callback) {
        callback.onFound(postcard);
    }
    //【1】如果跳转不能避开所有的拦截器，那么就要在这里处理，我们知道 Fragment 和 IProvider 的子类是会避开拦截器的！
    if (!postcard.isGreenChannel()) {
        //【-->2.3.2】这一部分设计拦截器功能，我们在跳转那一篇再分析；
        interceptorService.doInterceptions(postcard, new InterceptorCallback() {  //【-->2.3.1.1】处理拦截结果；

            @Override
            public void onContinue(Postcard postcard) {
                _navigation(context, postcard, requestCode, callback);
            }

            @Override
            public void onInterrupt(Throwable exception) {
                if (null != callback) {
                    callback.onInterrupt(postcard);
                }

                logger.info(Consts.TAG, "Navigation failed, termination by interceptor : " + exception.getMessage());
            }
        });
    } else {
        //【-->3.2.7】最终的处理；
        return _navigation(context, postcard, requestCode, callback);
    }

    return null;
}
```

我们看到，这里传入了一个拦截结果回调：

#### 2.3.1.1 InterceptorCallback

位于 callback 包下：

```java
public interface InterceptorCallback {
    void onContinue(Postcard postcard);
    void onInterrupt(Throwable exception);
}
```



### 2.3.2 InterceptorServiceImpl.doInterceptions

当我们路由跳转时，如果指定了 Interceptor，那么就要执行拦截操作：

```java
    @Override
    public void doInterceptions(final Postcard postcard, final InterceptorCallback callback) {
        if (null != Warehouse.interceptors && Warehouse.interceptors.size() > 0) {
            //【-->2.3.2.1】判断下 init 操作是否完成；
            checkInterceptorsInitStatus();

            if (!interceptorHasInit) {
                callback.onInterrupt(new HandlerException("Interceptors initialization takes too much time."));
                return;
            }
            //【1】这里是在由线程池管理的子线程中执行 init 操作；
            LogisticsCenter.executor.execute(new Runnable() {
                @Override
                public void run() {
                    //【2】创建了一个 CountDownLatch 对象，这个对象以 Warehouse.interceptors 的 size 为计数基准；
                    // 没处理一个 inteceptor，计数减一，知道计数为 0，才会释放持有的锁！；
                    CancelableCountDownLatch interceptorCounter = new CancelableCountDownLatch(Warehouse.interceptors.size());
                    try {
                        //【-->2.3.2.2】执行拦截操作！
                        _excute(0, interceptorCounter, postcard);
                        //【3】调用 await，子线程进入等待中；
                        interceptorCounter.await(postcard.getTimeout(), TimeUnit.SECONDS);
                      
                        //【4】拦截器处理完成（CountDownLatch 计数归 0），或者 await 超时退出；
                        if (interceptorCounter.getCount() > 0) { // Cancel the navigation this time, if it hasn't return anythings.
                            callback.onInterrupt(new HandlerException("The interceptor processing timed out."));
                        } else if (null != postcard.getTag()) { // Maybe some exception in the tag.
                            callback.onInterrupt(new HandlerException(postcard.getTag().toString()));
                        } else {
                            callback.onContinue(postcard);
                        }
                    } catch (Exception e) {
                        callback.onInterrupt(e);
                    }
                }
            });
        } else {
            callback.onContinue(postcard);
        }
    }
```

这里其实可以看出，拦截器使用了责任链模式！

这里有一个新的类型：CancelableCountDownLatch，其实就是一个 CountDownLatch，代码很简单，不多说了！

#### 2.3.2.1 checkInterceptorsInitStatus

判断是否初始化完成：

```java
    private static void checkInterceptorsInitStatus() {
        synchronized (interceptorInitLock) {
            //【1】主要是判断 interceptorHasInit 是否为 true！
            while (!interceptorHasInit) {
                try {
                    //【1】进入等待状态，超时时间是 10s！
                    interceptorInitLock.wait(10 * 1000);
                } catch (InterruptedException e) {
                    throw new HandlerException(TAG + "Interceptor init cost too much time error! reason = [" + e.getMessage() + "]");
                }
            }
        }
    }
```

interceptorInitLock 是 InterceptorServiceImpl 内部的一个锁对象；

#### 2.3.2.2 _excute

index 的值为 0，开始执行拦截：

```java
    private static void _excute(final int index, final CancelableCountDownLatch counter, final Postcard postcard) {
        if (index < Warehouse.interceptors.size()) {
            //【1】获得 index 对应的拦截器；
            IInterceptor iInterceptor = Warehouse.interceptors.get(index);
            //【2】执行拦截器的 process 方法，同时传入一个回调：【-->2.3.1.1】InterceptorCallback
            iInterceptor.process(postcard, new InterceptorCallback() {
                @Override
                public void onContinue(Postcard postcard) {
                    //【2.1】不拦截，CountDownLatch 计数减去 1；
                    counter.countDown();
                    //【-->2.3.2.2】继续调用 _excute 方法，index 加一，下一个拦截器；
                    _excute(index + 1, counter, postcard);  // When counter is down, it will be execute continue ,but index bigger than interceptors size, then U know.
                }

                @Override
                public void onInterrupt(Throwable exception) {
                    //【2.2】拦截，CountDownLatch 计数归 0；
                    postcard.setTag(null == exception ? new HandlerException("No message.") : exception.getMessage());    // save the exception message for backup.
                    counter.cancel();
                    // Be attention, maybe the thread in callback has been changed,
                    // then the catch block(L207) will be invalid.
                    // The worst is the thread changed to main thread, then the app will be crash, if you throw this exception!
//                    if (!Looper.getMainLooper().equals(Looper.myLooper())) {    // You shouldn't throw the exception if the thread is main thread.
//                        throw new HandlerException(exception.getMessage());
//                    }
                }
            });
        }
    }
```

整个过程很简单，不多说了。

# 3 线程池

刚刚我们有看到，拦截器的初始化和拦截都是在子线程中做的，ARouter 通过内部的一个线程池来管理：

```java
DefaultPoolExecutor.java  // 线程池
DefaultThreadFactory.java  // 线程工厂
```

## 3.1 线程池初始化

```java
final class _ARouter {
    static ILogger logger = new DefaultLogger(Consts.TAG);
    private volatile static boolean monitorMode = false;
    private volatile static boolean debuggable = false;
    private volatile static boolean autoInject = false;
    private volatile static _ARouter instance = null;
    private volatile static boolean hasInit = false;
    //【1】DefaultPoolExecutor 是 _ARouter 的静态变量；
    private volatile static ThreadPoolExecutor executor = DefaultPoolExecutor.getInstance();
```

然后再初始化 LogisticsCenter 的时候传递给了 LogisticsCenter；



## 3.2 DefaultPoolExecutor

我们来看下线程池的构造，这里要重点看看线程池的核心参数：



### 3.2.1 getInstance

单例模式：

```java
    public static DefaultPoolExecutor getInstance() {
        if (null == instance) {
            synchronized (DefaultPoolExecutor.class) {
                if (null == instance) {
                    //【-->3.2.2】创建线程池；
                    instance = new DefaultPoolExecutor(
                            INIT_THREAD_COUNT,
                            MAX_THREAD_COUNT,
                            SURPLUS_THREAD_LIFE,
                            TimeUnit.SECONDS,
                            new ArrayBlockingQueue<Runnable>(64),
                            new DefaultThreadFactory());
                }
            }
        }
        return instance;
    }
```

### 3.2.2 new DefaultPoolExecutor

我们来研究下 DefaultPoolExecutor 的一些核心参数：

```java
   private DefaultPoolExecutor(int corePoolSize, int maximumPoolSize, long keepAliveTime, TimeUnit unit, BlockingQueue<Runnable> workQueue, 
                               																																												ThreadFactory threadFactory) {
        super(corePoolSize, maximumPoolSize, keepAliveTime, unit, workQueue, threadFactory, new RejectedExecutionHandler() {
            @Override
            public void rejectedExecution(Runnable r, ThreadPoolExecutor executor) {
                ARouter.logger.error(Consts.TAG, "Task rejected, too many task!");
            }
        });
    }
```

这里涉及到了 DefaultPoolExecutor 内部的常量：

```java
    private static final int CPU_COUNT = Runtime.getRuntime().availableProcessors(); // Java 虚拟机的可用的处理器数量；
    private static final int INIT_THREAD_COUNT = CPU_COUNT + 1;  // 核心线程数
    private static final int MAX_THREAD_COUNT = INIT_THREAD_COUNT;  // 最大线程数；
    private static final long SURPLUS_THREAD_LIFE = 30L; // 空闲线程在
```

可以看到，线程池的参数如下：

- corePoolSize：核心线程数是可用的处理器数量 + 1；
- maximumPoolSize：最大线程数是可用的处理器数量；
- keepAliveTime：空闲线程存活时间：30s；
- workQueue：阻塞队列是 ArrayBlockingQueue，数组实现的阻塞队列，有界 64；
- threadFactory：线程工厂类，自定义的 DefaultThreadFactory 类；
- RejectedExecutionHandler：线程池在无法处理添加的 runnnable 时的处理机制，这里是自定义了一个 RejectedExecutionHandler，只是打印了一个 Log；



## 3.3 DefaultThreadFactory

ARouter 内部自定义的线程工厂类，DefaultThreadFactory 需要实现 ThreadFactory 接口；

```java
    public DefaultThreadFactory() {
        //【1】这里通过 SecurityManager 来设置 thread 的 group；
        SecurityManager s = System.getSecurityManager();
        group = (s != null) ? s.getThreadGroup() :
                Thread.currentThread().getThreadGroup();
        namePrefix = "ARouter task pool No." + poolNumber.getAndIncrement() + ", thread No.";
    }
```

我们来看看 newThread 方法：

```java
    public Thread newThread(@NonNull Runnable runnable) {
        String threadName = namePrefix + threadNumber.getAndIncrement();
        ARouter.logger.info(Consts.TAG, "Thread production, name is [" + threadName + "]");
        //【1】创建线程，一个线程对应一个 runnable；
        Thread thread = new Thread(group, runnable, threadName, 0);
        if (thread.isDaemon()) {   //【2】设为非后台线程
            thread.setDaemon(false);
        }
        if (thread.getPriority() != Thread.NORM_PRIORITY) { // 【2】优先级为 normal
            thread.setPriority(Thread.NORM_PRIORITY);
        }

        //【3】捕获多线程处理中的异常
        thread.setUncaughtExceptionHandler(new Thread.UncaughtExceptionHandler() {
            @Override
            public void uncaughtException(Thread thread, Throwable ex) {
                ARouter.logger.info(Consts.TAG, "Running task appeared exception! Thread [" + thread.getName() + "], because [" + ex.getMessage() + "]");
            }
        });
        return thread;
    }
```

不多说了！

# 4 总结

本篇文章分析了 ARouter 的服务和拦截器的相关机制

- 拦截器的初始化和拦截操作都是在子线程中处理的，拦截器使用了责任链模式；
- 子线程通过线程池管理，采用了单例模式；
- 拦截器是使用了责任链模式，通过它使用 CountDownLatch 来实现了路由等待的操作；

但是遗留了几个问题：

- AutowiredService 和 AutowiredServiceImpl 是如何工作的；
- ClassLoaderService 是如何工作的；

我们下次再说～～～