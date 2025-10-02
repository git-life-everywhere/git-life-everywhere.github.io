#  ARouter 第四篇 - 路由初始化 (arouter-api)

title: ARouter 第四篇 - 路由初始化 (arouter-api)
date: 2019/04/23 20:46:25 
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
>

本篇博文主要分析 arouter-api 模块，该模块涉及到 ARouter 一些核心逻辑：初始化，跳转，拦截，服务等，下面的几篇文章就要从这几个方向来分析；

绘图工具：PlantXML

在阅读过程中，涉及到方法跳转的时候，注释上有 `-->`的标志，这样的好处是，以类为单位，一次性分析其所有的方法：

#  1 模块结构

我们先来看看模块结构：

```
└── com
    └── alibaba
        └── android
            └── arouter
                ├── base
                │   └── UniqueKeyTreeMap.java
                ├── core
                │   ├── AutowiredLifecycleCallback.java
                │   ├── AutowiredServiceImpl.java
                │   ├── InstrumentationHook.java
                │   ├── InterceptorServiceImpl.java
                │   ├── LogisticsCenter.java
                │   └── Warehouse.java
                ├── exception
                │   ├── HandlerException.java
                │   ├── InitException.java
                │   └── NoRouteFoundException.java
                ├── facade
                │   ├── Postcard.java
                │   ├── callback
                │   │   ├── InterceptorCallback.java
                │   │   ├── NavCallback.java
                │   │   └── NavigationCallback.java
                │   ├── service
                │   │   ├── AutowiredService.java
                │   │   ├── ClassLoaderService.java
                │   │   ├── DegradeService.java
                │   │   ├── InterceptorService.java
                │   │   ├── PathReplaceService.java
                │   │   ├── PretreatmentService.java
                │   │   └── SerializationService.java
                │   └── template
                │       ├── IInterceptor.java
                │       ├── IInterceptorGroup.java
                │       ├── ILogger.java
                │       ├── IPolicy.java
                │       ├── IProvider.java
                │       ├── IProviderGroup.java
                │       ├── IRouteGroup.java
                │       ├── IRouteRoot.java
                │       └── ISyringe.java
                ├── launcher
                │   ├── ARouter.java
                │   └── _ARouter.java
                ├── thread
                │   ├── CancelableCountDownLatch.java
                │   ├── DefaultPoolExecutor.java
                │   └── DefaultThreadFactory.java
                └── utils
                    ├── ClassUtils.java
                    ├── Consts.java
                    ├── DefaultLogger.java
                    ├── MapUtils.java
                    ├── PackageUtils.java
                    └── TextUtils.java

```

 可以看到，有如下的 package：

- **base**：数据缓存类，内部提供了一个 treeMap 实现，用于存储 intercepter；
- **core**：核心类，ARouter的核心功能都在这里实现；
- **exception**：异常相关，主要是定义了内部的一些异常；
- **facade**：通过外观模式对外提供统一的接口，下面有三个子包：
	- **callback**：提供回调接口，以及默认的回调处理；
	- **service**：ARouter 内部已经实现的一些 Service，对外提供拦截等功能；
	- **template**：包含模版，提供了大量的模版接口，可以通过实现接口，配合注解，实现自定义的功能；
- **launcher**：ARouter 的入口；
- **thread**：线程操作类；
- **utils**：提供多个工具类



#  2  初始化方法

我们在使用时，必须要做初始化：

```java
//这两行必须写在 init 之前，否则这些配置在 init 过程中将无效；     
//【1】打印日志；
ARouter.openLog();
//【2】开启调试模式（如果在 InstantRun 模式下运行，必须开启调试模式！线上版本需要关闭,否则有安全风险）；
ARouter.openDebug();
ARouter.init(mApplication); // 尽可能早，推荐在 Application 中初始化；
```

接下来，我们来看看 init 的过程：

#  3 Launcher 包

##  3.1 ARouter

ARouter 是整个库的入口！

### 3.1.1 成员变量

```java
    // Key of raw uri
    public static final String RAW_URI = "NTeRQWvye18AkPd6G";
    public static final String AUTO_INJECT = "wmHzgD4lOj5o4241";

    private volatile static ARouter instance = null;
    private volatile static boolean hasInit = false;
    public static ILogger logger;
```

###  3.1.2 init

我们来看看 init 初始化的方法：

```java
    public static void init(Application application) {
        if (!hasInit) {
            logger = _ARouter.logger;
            _ARouter.logger.info(Consts.TAG, "ARouter init start.");
            //【-->3.2.2】执行初始化；
            hasInit = _ARouter.init(application);
            if (hasInit) {
                //【-->3.2.3】执行初始化后面的操作；
                _ARouter.afterInit();
            }

            _ARouter.logger.info(Consts.TAG, "ARouter init over.");
        }
    }
```



###  3.1.3 getInstance



获得 ARouter 的实例（单例模式）：

```java
    public static ARouter getInstance() {
        if (!hasInit) {
            throw new InitException("ARouter::Init::Invoke init(context) first!");
        } else {
            if (instance == null) {
                synchronized (ARouter.class) {
                    if (instance == null) {
                        //【1】ARouter 的构造器方法体是空的；
                        instance = new ARouter(); 
                    }
                }
            }
            return instance;
        }
    }
```

这里使用了单例模式创建：ARouter ！

###  3.1.4 build

afterInit 方法中传入了 "/arouter/service/interceptor" 参数，创建跳转信息！

```java
    public Postcard build(String path) { // Route.path
        //【-->3.2.4】返回 _ARouter 的实例
        //【-->3.2.5】创建跳转信息；
        return _ARouter.getInstance().build(path);
    }
    ... ... ...// 先不关注其他的方法；
```

ARouter 提供了下面的多个方法用于创建跳转信息：

- `Postcard build(String path)`: 指定 Route.path，**跳转/初始化**都会使用到该方法；

- `Postcard build(String path, String group)`：指定 Route.path, Route.group，跳转时使用；
- `Postcard build(Uri url)`：指定 uri，uri 需要在说明书中设置；

这里我们**先关注 init 过程中调用的**！



可以看到，最后调用的是 _ARouter 的方法，注意这个方法返回的是：

- **Postcard**：继承了 RouteMeta，用于封装跳转信息；

### 3.1.5 navigation

执行跳转：

```java
    public <T> T navigation(Class<? extends T> service) {
        //【-->3.2.6】通过跳转，返回服务对象；
        return _ARouter.getInstance().navigation(service);
    }

    public Object navigation(Context mContext, Postcard postcard, int requestCode, NavigationCallback callback) {
        //【-->3.2.6】执行跳转，这也是真正的跳转接口；
        return _ARouter.getInstance().navigation(mContext, postcard, requestCode, callback);
    }
```

ARouter 提供了 2 个跳转接口：

- `navigation(Class<? extends T> service)`: 用于**获取泛型指定的 Service**，实际上并不是跳转接口；
- `navigation(Context mContext, Postcard postcard, int requestCode, NavigationCallback callback)`：这个才是真正的跳转接口！



这篇博文先不讲 **navigation**，我们分析初始化 init 的过程：

- `interceptorService = (InterceptorService) ARouter.getInstance().build("/arouter/service/interceptor").navigation();`	
  - 在执行 **afterInit** 的时候，**会通过 navigation 方法返回 InterceptorServiceIpml 实例**，这个方法我们跟踪了代码，调的是第二个 **navigation** 方法；
- `PathReplaceService pService = ARouter.getInstance().navigation(PathReplaceService.class);`
  - 而在获得  InterceptorService 实例的时候，会**先调用 build 方法**，获得 **PathReplaceService** 实例，这里就是**第一个 navigation 方法**，但是这里我们不分析它；
  - 实际上 **PathReplaceService 和 InterceptorServiceImpl 的获取方式是一样的**！



## 3.2 _ARouter

### 3.2.1 成员变量

下面是 _ARouter 的成员属性：

```java
    static ILogger logger = new DefaultLogger(Consts.TAG); // Log 系统，位于 utils 包；
    private volatile static boolean monitorMode = false;
    private volatile static boolean debuggable = false;
    private volatile static boolean autoInject = false;
    private volatile static _ARouter instance = null;
    private volatile static boolean hasInit = false;
    private volatile static ThreadPoolExecutor executor = DefaultPoolExecutor.getInstance(); // 线程池对象，由 DefaultPoolExecutor 创建；
    private static Handler mHandler;
    private static Context mContext;

    private static InterceptorService interceptorService; // 用于执行所有的 Interceptor；
```



###  3.2.2 init

```java
    protected static synchronized boolean init(Application application) {
        mContext = application;
        //【-->4.1.2】初始化 LogisticsCenter
        LogisticsCenter.init(mContext, executor);
        logger.info(Consts.TAG, "ARouter init success!");
        hasInit = true; // 判断是否 init；
        mHandler = new Handler(Looper.getMainLooper()); // 主线程的 handler

        return true;
    }
```

在初始化 LogisticsCenter 的时候，传入了一个线程池！



###  3.2.3 afterInit

 在 ARouter 执行完成初始化之后，会触发 interceptor 的 init 操作：

```java
    static void afterInit() {
        //【-->3.1.3】获取 ARouter 的实例；
        //【-->3.1.4】build 跳转信息，返回一个 PostCard 实例；
        //【-->6.1.3】执行 PostCard 的 nativagation 方法，获得系统服务 InterceptorServiceImpl 实例；
        interceptorService = (InterceptorService) ARouter.getInstance().build("/arouter/service/interceptor").navigation();
    }
```

可以看到，获取 InterceptorServiceImpl 实例，是通过 PostCard.navigation 方法的！

**build 的参数传入的是 "/arouter/service/interceptor"**，这里我们要获取一个 InterceptorServiceImpl 实例 ！

### 3.2.4 getInstance

```java
    protected static _ARouter getInstance() {
        if (!hasInit) {
            throw new InitException("ARouterCore::Init::Invoke init(context) first!");
        } else {
            if (instance == null) {
                synchronized (_ARouter.class) {
                    if (instance == null) {
                        //【1】_ARouter 的构造器是空的，就不再分析了；
                        instance = new _ARouter();
                    }
                }
            }
            return instance;
        }
    }
```

通过单例模式创建 _ARouter 对象；



### 3.2.5 build

ARouter.build 的方法，最后会掉到 _ARouter 中来；

```java
    protected Postcard build(String path) { // Route.path；
        if (TextUtils.isEmpty(path)) {
            throw new HandlerException(Consts.TAG + "Parameter is invalid!");
        } else {
            //【-->3.1.5】创建 PathReplaceService，用于在跳转前，拦截 path，并对 path 做处理！
            PathReplaceService pService = ARouter.getInstance().navigation(PathReplaceService.class);
            if (null != pService) {
                //【--->6.3.1】通过 PathReplaceService 对 path 做预处理！
                path = pService.forString(path);
            }
            //【2】调用第三个方法：
            return build(path, extractGroup(path));
        }
    }
    ... ... ...// 先不看其他的 build 方法！

    protected Postcard build(String path, String group) { // Route,path, Route.group；
        if (TextUtils.isEmpty(path) || TextUtils.isEmpty(group)) {
            throw new HandlerException(Consts.TAG + "Parameter is invalid!");
        } else {
            //【-->3.1.5】创建 PathReplaceService，用于在跳转前，拦截 uri，并对 uri 做处理！
            PathReplaceService pService = ARouter.getInstance().navigation(PathReplaceService.class);
            if (null != pService) {
                //【--->6.3.1】通过 PathReplaceService 对 path 做预处理！
                path = pService.forString(path);
            }
            //【-->5.1】返回跳转信息；
            //【-->6.1.1】创建跳转实例 Postcard；
            return new Postcard(path, group);
        }
    }
```

同样，也是有三个重载函数：

- `Postcard build(String path)`: 指定 Route.path，**跳转/初始化**都会使用到该方法，这个方法会调用第二个；
- `Postcard build(String path, String group)`：指定 Route.path, Route.group，跳转时使用；
- `Postcard build(Uri url)`：指定 uri，uri 需要在说明书中设置；

这里我们**先关注 init 过程中调用的**！

（注意：这里有一个 **PathReplaceService**，用于在跳转前，拦截 path，并对 path 做处理，这个 Service 和路由跳转有关系，初始化这里我们先不过多分析！）

#### 3.2.5.1 extractGroup

这个方法的作用是对 path 做修正，看 path 是否正确，同时根据 path 生成 group：

```java
    private String extractGroup(String path) {
        //【1】校验 path 是否正确
        if (TextUtils.isEmpty(path) || !path.startsWith("/")) {
            throw new HandlerException(Consts.TAG + "Extract the default group failed, the path must be start with" 
                                       + "'/' and contain more than 2 '/'!");
        }

        try {
            //【3】根据 path 生成 group；
            String defaultGroup = path.substring(1, path.indexOf("/", 1));
            if (TextUtils.isEmpty(defaultGroup)) {
                throw new HandlerException(Consts.TAG + "Extract the default group failed! There's nothing between 2 '/'!");
            } else {
                return defaultGroup;
            }
        } catch (Exception e) {
            logger.warning(Consts.TAG, "Failed to extract default group! " + e.getMessage());
            return null;
        }
    }
```



### 3.2.6 navigation

最后会进入 _ARouter 的 navigation 方法中，我们看到，该方法的逻辑还是很多的，注意，这里我们传入的 callback 是 null 的！

```java
    protected Object navigation(final Context context, final Postcard postcard, final int requestCode, final NavigationCallback callback) {
        //【*important】这里又和 PathReplaceService 一样的，又是通过跳转的方式获取 PretreatmentService 服务，对 Postcard 做预处理；
        // 这个同样的，我们先不看；
        PretreatmentService pretreatmentService = ARouter.getInstance().navigation(PretreatmentService.class);
        if (null != pretreatmentService && !pretreatmentService.onPretreatment(context, postcard)) {
            return null;
        }

        try {
            //【-->4.1.2】完善跳转信息！
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
                ///【1】当完善失败，则通过 callback.onLost 提示用户！
                callback.onLost(postcard);
            } else {
                //【*important】如果没有指定 callback，显然通过降级服务处理！
                // 这里又和 PathReplaceService 一样的，又是通过跳转的方式获取 DegradeService 服务，对 Postcard 做预处理；
                // 这个同样的，我们先不看；
                DegradeService degradeService = ARouter.getInstance().navigation(DegradeService.class);
                if (null != degradeService) {
                    degradeService.onLost(context, postcard);
                }
            }

            return null;
        }

        if (null != callback) {
            //【2】回调 onFound 方法，表示跳转信息有效；
            callback.onFound(postcard);
        }
        //【-->6.1】如果跳转不能避开所有的拦截器，那么就要在这里处理，我们知道 Fragment 和 IProvider 的子类是会避开拦截器的！
        if (!postcard.isGreenChannel()) {
            //【*important】这一部分设计拦截器功能，我们在跳转那一篇再分析；
            interceptorService.doInterceptions(postcard, new InterceptorCallback() {

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

**_ARouter 也提供了 2 个跳转接口**：

- `navigation(Context mContext, Postcard postcard, int requestCode, NavigationCallback callback)`：这个才是真正的跳转接口！

同样的，我们也只看 `init`的过程，也就是获取 **InterceptorServiceImpl** 实例的过程，这个过程调用的是上面的四参方法；

- `<T> T navigation(Class<? extends T> service)`: 用于**获取泛型指定的 Service**！

  ```java
      protected <T> T navigation(Class<? extends T> service) {
          try {
              //【-->4.1.3】通过 serviceName 找到，对应的 Service 的 RouteMeta 实例，然后创建 Postcard 实例
              // service.getName() 返回的是全限定名；
              Postcard postcard = LogisticsCenter.buildProvider(service.getName());
  
              //【1】如果是 null，说明使用的是旧版本的 compiler sdk，早期的 compiler 不使用全限定名区获取服务；
              if (null == postcard) {
                  //【-->4.1.3】通过 serviceName 找到，对应的 Service 的 RouteMeta 实例，然后创建 Postcard 实例
                  postcard = LogisticsCenter.buildProvider(service.getSimpleName());
              }
  
              if (null == postcard) {
                  return null;
              }
              //【-->4.1.4】完成跳转！
              LogisticsCenter.completion(postcard);
              return (T) postcard.getProvider();
          } catch (NoRouteFoundException ex) {
              logger.warning(Consts.TAG, ex.getMessage());
              return null;
          }
      }
  ```


实际上并不是跳转接口，**PathReplaceService**，**PretreatmentService**，**DegradeService** 都是通过这个方法获取！

我们后面统一进行分析！

### 3.2.7 _navigation

最后会调用 _navigation 返回我们的 InterceptorServiceImpl 实例，我们知道 **InterceptorServiceImpl 是 PROVIDER 类型的**：

```java
    private Object _navigation(final Context context, final Postcard postcard, final int requestCode, final NavigationCallback callback) {
        final Context currentContext = null == context ? mContext : context;

        switch (postcard.getType()) {
            case ACTIVITY:
                // Build intent
                final Intent intent = new Intent(currentContext, postcard.getDestination());
                intent.putExtras(postcard.getExtras());

                // Set flags.
                int flags = postcard.getFlags();
                if (-1 != flags) {
                    intent.setFlags(flags);
                } else if (!(currentContext instanceof Activity)) {    // Non activity, need less one flag.
                    intent.setFlags(Intent.FLAG_ACTIVITY_NEW_TASK);
                }

                // Set Actions
                String action = postcard.getAction();
                if (!TextUtils.isEmpty(action)) {
                    intent.setAction(action);
                }

                // Navigation in main looper.
                runInMainThread(new Runnable() {
                    @Override
                    public void run() {
                        startActivity(requestCode, currentContext, intent, postcard, callback);
                    }
                });

                break;
            case PROVIDER:
                //【over】返回了 iprovider 实例，就是我们的 InterceptorServiceImpl 对象；
                return postcard.getProvider();
            case BOARDCAST:
            case CONTENT_PROVIDER:
            case FRAGMENT:
                Class fragmentMeta = postcard.getDestination();
                try {
                    Object instance = fragmentMeta.getConstructor().newInstance();
                    if (instance instanceof Fragment) {
                        ((Fragment) instance).setArguments(postcard.getExtras());
                    } else if (instance instanceof android.support.v4.app.Fragment) {
                        ((android.support.v4.app.Fragment) instance).setArguments(postcard.getExtras());
                    }

                    return instance;
                } catch (Exception ex) {
                    logger.error(Consts.TAG, "Fetch fragment instance error, " + TextUtils.formatStackTrace(ex.getStackTrace()));
                }
            case METHOD:
            case SERVICE:
            default:
                return null;
        }

        return null;
    }
```

这里涉及到其他类型的处理，我们在路由跳转的时候再分析；

#  4 core 包

## 4.1 LogisticsCenter - 核心一号种子



### 4.1.1 成员变量

 ```java
    private static Context mContext;
    static ThreadPoolExecutor executor; // 线程池；
    private static boolean registerByPlugin; // 是否通过插件自动注册；
 ```



###  4.1.2 init

执行 init 操作；

```java
public synchronized static void init(Context context, ThreadPoolExecutor tpe) throws HandlerException {
  mContext = context;
  executor = tpe;

  try {
    long startInit = System.currentTimeMillis();
    //【-->5.1.1】是否使用 arouter-auto-register 插件来加载路由表；
    loadRouterMap();
    if (registerByPlugin) {
      logger.info(TAG, "Load router map by arouter-auto-register plugin.");
    } else { // 默认情况下是进入这里：
      Set<String> routerMap;

      //【1】如果开启了 debug 模式或者说 Apk 发生了更新，那么 ARouter 会重建路由表；
      //【-->9.2.2】isNewVersion 判断 apk 是否是新的安装；
      if (ARouter.debuggable() || PackageUtils.isNewVersion(context)) {
        logger.info(TAG, "Run with debug mode or new install, rebuild router map.");
 
        //【-->9.1.2】这里更新路由表，通过包名 com.alibaba.android.arouter.routes，扫描包下面包含的所有的类的 ClassName
        // 这个包是在 arouter complier 阶段生成的，里面包含解析注解生成的 java 类；
        routerMap = ClassUtils.getFileNameByPackageName(mContext, ROUTE_ROOT_PAKCAGE);
        if (!routerMap.isEmpty()) {
          //【1.1】当我们能够扫描到路由信息后，会将这个信息保存到本地 sp 中；
          context.getSharedPreferences(AROUTER_SP_CACHE_KEY, Context.MODE_PRIVATE).edit().putStringSet(AROUTER_SP_KEY_MAP, routerMap).apply();
        }

        PackageUtils.updateVersion(context); //【-->9.2.3】保存版本号；
      } else {
        logger.info(TAG, "Load router map from cache.");
        //【2】其他情况，是默认从本地缓存中读取路由表的！
        routerMap = new HashSet<>(context.getSharedPreferences(AROUTER_SP_CACHE_KEY, Context.MODE_PRIVATE).getStringSet(AROUTER_SP_KEY_MAP, new HashSet<String>()));
      }

      logger.info(TAG, "Find router map finished, map size = " + routerMap.size() + ", cost " + (System.currentTimeMillis() - startInit) + " ms.");
      startInit = System.currentTimeMillis();

      //【2】处理路由表的信息，在前面，我们将 aouter-compiler 在编译时期生成的 class 都加载到了 routerMap 中了；
      for (String className : routerMap) {
        if (className.startsWith(ROUTE_ROOT_PAKCAGE + DOT + SDK_NAME + SEPARATOR + SUFFIX_ROOT)) {
          //【2.1】判断前缀：com.alibaba.android.arouter.routes.ARouter&&Root，符合前缀的都是 IRouteRoot 的子类
          // 调用其 loadInto --> Warehouse.groupsIndex；
          ((IRouteRoot) (Class.forName(className).getConstructor().newInstance())).loadInto(Warehouse.groupsIndex);
         
        } else if (className.startsWith(ROUTE_ROOT_PAKCAGE + DOT + SDK_NAME + SEPARATOR + SUFFIX_INTERCEPTORS)) {
          //【2.2】判断前缀：com.alibaba.android.arouter.routes.ARouter&&Interceptors，符合前缀的都是 IInterceptorGroup 的子类
          // 调用其 loadInto --> Warehouse.interceptorsIndex；
          ((IInterceptorGroup) (Class.forName(className).getConstructor().newInstance())).loadInto(Warehouse.interceptorsIndex);

        } else if (className.startsWith(ROUTE_ROOT_PAKCAGE + DOT + SDK_NAME + SEPARATOR + SUFFIX_PROVIDERS)) {
          //【2.3】判断前缀：com.alibaba.android.arouter.routes.ARouter&&Providers，符合前缀的都是 IProviderGroup 的子类
          // 调用其 loadInto --> Warehouse.providersIndex；
          ((IProviderGroup) (Class.forName(className).getConstructor().newInstance())).loadInto(Warehouse.providersIndex);

        }
      }
    }

    logger.info(TAG, "Load root element finished, cost " + (System.currentTimeMillis() - startInit) + " ms.");

    if (Warehouse.groupsIndex.size() == 0) {
      logger.error(TAG, "No mapping files were found, check your configuration please!");
    }

    if (ARouter.debuggable()) {
      logger.debug(TAG, String.format(Locale.getDefault(), "LogisticsCenter has already been loaded, GroupIndex[%d], “ 
                                      + ”InterceptorIndex[%d], ProviderIndex[%d]", Warehouse.groupsIndex.size(),“ 
                                      + Warehouse.interceptorsIndex.size(), Warehouse.providersIndex.size()));
    }
  } catch (Exception e) {
    throw new HandlerException(TAG + "ARouter init logistics center exception! [" + e.getMessage() + "]");
  }
}
```

我们看到，ARouter 会将路由表存储在本地缓存 sp 中，在版本号发生变化的时候会处理，上面的这些常量定义在 Consts 中。



我们还记得 arouter compiler 在动态生成代码的时候，会创建 IRouteRoot，IInterceptorGroup，IProviderGroup 的子类，我们通过 Route，intercepor  注解的元素都会被封装成 RouteMeta 实例，通过 loadInto 方法，加入到 Warehouse 对应的集合中！



下面我们通过之前动态代码来分析：



- 如果前缀是 `ARouter$$Root`, 那么会触发 `ARouter$$Root$$moduleName.loadInto`方法：

```java
public class ARouter$$Root$$app implements IRouteRoot {
  @Override
  public void loadInto(Map<String, Class<? extends IRouteGroup>> routes) {
    routes.put("coolqiActivity", ARouter$$Group$$coolqiActivity.class);
    routes.put("coolqiProvider", ARouter$$Group$$coolqiProvider.class);
    routes.put("coolqiService", ARouter$$Group$$coolqiService.class);
  }
}
```

这部分数据会加载到：**Warehouse.groupsIndex**，这样，我们就可以通过它按组加载了；



- 如果前缀是 `ARouter$$Interceptors`, 那么会触发 `ARouter$$Interceptors$$moduleName.loadInto`方法：

```java
public class ARouter$$Interceptors$$app implements IInterceptorGroup {
  @Override
  public void loadInto(Map<Integer, Class<? extends IInterceptor>> interceptors) {
    interceptors.put(8, TestInterceptor.class);
  }
}
```

这部分数据会加载到：**Warehouse.interceptorsIndex**



- 如果前缀是 `ARouter$$Providers`, 那么会触发 `ARouter$$Providers$$moduleName.loadInto`方法：

```java
public class ARouter$$Providers$$app implements IProviderGroup {
  @Override
  public void loadInto(Map<String, RouteMeta> providers) {
    providers.put("com.alibaba.android.arouter.facade.service.SerializationService", RouteMeta.build(RouteType.PROVIDER, MySerializationService.class, 
                            "/coolqiService/MySerializationService", "coolqiService", null, -1, -2147483648));
    providers.put("com.pa.sales2.test.MyIProvider", RouteMeta.build(RouteType.PROVIDER, MyIProvider.class, 
                            "/coolqiProvider/MyIProvider", "coolqiProvider", null, -1, -2147483648));
  }
}
```

这部分数据会加载到：**Warehouse.providersIndex**

 

#### 4.2.1.1 loadRouterMap

这个方法用于判断是否通过 arouter-auto-register 插件自动注册路由；

```java
    private static void loadRouterMap() {
        registerByPlugin = false;
        //auto generate register code by gradle plugin: arouter-auto-register
        // looks like below:
        // registerRouteRoot(new ARouter..Root..modulejava());
        // registerRouteRoot(new ARouter..Root..modulekotlin());
    }
```

可以看到，如果使用了 arouter-auto-register 插件，那么会自动执行 registerRouteRoot 相关代码；

这里我们先不看和 arouter-auto-register 相关的代码：

这个方法默认是将 registerByPlugin 设置为 false；

### 4.1.3 buildProvider

通过 serviceName 找到，对应的 Service 的 RouteMeta 实例，然后创建 Postcard 实例：

```java
    public static Postcard buildProvider(String serviceName) {
        //【-->4.2.1】我们知道 service 实现了 IProvider 实例，所以保存在了 Warehouse.providersIndex 中！
        RouteMeta meta = Warehouse.providersIndex.get(serviceName);

        if (null == meta) {
            return null;
        } else {
            //【-->6.1.2】创建路由跳转信息；
            return new Postcard(meta.getPath(), meta.getGroup());
        }
    }
```

###  4.1.4 completion

完善跳转信息，completion 会通过 Warehouse 的数据，填充 Postcard！

```java
    public synchronized static void completion(Postcard postcard) {
        if (null == postcard) {
            throw new NoRouteFoundException(TAG + "No postcard!");
        }
        //【-->4.2.1】从 Warehouse.routes 中获取 path 对应的 RouteMeta 缓存数据；
        RouteMeta routeMeta = Warehouse.routes.get(postcard.getPath());
        //【1】如果找不到，那么就从 compiler 生成的数据中查找！
        if (null == routeMeta) { 
            //【-->4.2.1】从 Warehouse.routes 中获取 group 对应的 group 类文件；
            Class<? extends IRouteGroup> groupMeta = Warehouse.groupsIndex.get(postcard.getGroup());
            if (null == groupMeta) { // 【1.1】找不到抛出异常；
                throw new NoRouteFoundException(TAG + "There is no route match the path [" + postcard.getPath() + "], in group [" + postcard.getGroup() + "]");
            } else {

                try {
                    if (ARouter.debuggable()) {
                        logger.debug(TAG, String.format(Locale.getDefault(), "The group [%s] starts loading, trigger by [%s]", postcard.getGroup(), postcard.getPath()));
                    }
                    //【1.2】创建 groupMeta 对应的实例；
                    IRouteGroup iGroupInstance = groupMeta.getConstructor().newInstance();
                    //【-->4.2.1】调用其 loadInto 将 group 对应的信息加入到缓存 Warehouse.routes 中！
                    iGroupInstance.loadInto(Warehouse.routes);
                    //【-->4.2.1】然后从 Warehouse.groupsIndex 删除这个组对应的信息；
                    Warehouse.groupsIndex.remove(postcard.getGroup());

                    if (ARouter.debuggable()) {
                        logger.debug(TAG, String.format(Locale.getDefault(), "The group [%s] has already been loaded, trigger by [%s]", postcard.getGroup(), postcard.getPath()));
                    }
                } catch (Exception e) {
                    throw new HandlerException(TAG + "Fatal exception when loading group meta. [" + e.getMessage() + "]");
                }
                //【-->4.2.2】重新加载；
                completion(postcard);
            }
        } else {
            //【2】这里是通过 RouteMeta 来设置 Postcard 对象；
            postcard.setDestination(routeMeta.getDestination());
            postcard.setType(routeMeta.getType()); 
            postcard.setPriority(routeMeta.getPriority());
            postcard.setExtra(routeMeta.getExtra());
            //【3】如果指定了 uri 就要从 uri 中设置传递的数据了，显然，这里我们并没有设置 Uri；
            // 我们也没有传递数据，只是为了获取 InterceptorServiceImpl 实例，我们先不看！
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
                    // 这里和 AutoInject 有关系，我们先不看！
                    postcard.getExtras().putStringArray(ARouter.AUTO_INJECT, paramsType.keySet().toArray(new String[]{}));
                }
                postcard.withString(ARouter.RAW_URI, rawUri.toString());
            }

            //【4】这里是关键点，判断类型，可以看到 activity 这里是不处理的！；
            switch (routeMeta.getType()) {
                case PROVIDER: 
                    //【4.1】我们要获取的 InterceptorServiceImpl，类型就是 PROVIDER；
                    // routeMeta.getDestination 返回的是要访问的目标类：InterceptorServiceImpl.class;
                    Class<? extends IProvider> providerMeta = (Class<? extends IProvider>) routeMeta.getDestination();
                    //【-->4.2.1】然后优先从 Warehouse.providers 缓存中获取；
                    IProvider instance = Warehouse.providers.get(providerMeta);
                    if (null == instance) {
                        IProvider provider;
                        try {
                            //【-->4.3】创建 providerMeta 对应的实例，就是 InterceptorServiceImpl；
                            provider = providerMeta.getConstructor().newInstance();
                            //【-->4.3.1】执行 init 方法；
                            provider.init(mContext);
                            //【-->4.2.1】然后将加入到 Warehouse.providers 中去；
                            Warehouse.providers.put(providerMeta, provider);
                            instance = provider;
                        } catch (Exception e) {
                            throw new HandlerException("Init provider failed! " + e.getMessage());
                        }
                    }
                    //【-->6.1.1】将 instance 保存到 postcard.provider 中去，因为跳转目标是 IProvider 的子类；
                    postcard.setProvider(instance);
                    postcard.greenChannel();  //【-->6.1.1】跳过所有的拦截器！
                    break;
                case FRAGMENT:
                    postcard.greenChannel(); // 跳过所有的拦截器！
                default:
                    break;
            }
        }
    }
```



**缓存处理**：

- **Warehouse.groupsIndex —> Warehouse.routes**

这里优先从 Warehouse.groupsIndex 中读取，Warehouse.groupsIndex 中保存的是类似下面的数据；

```java
routes.put("coolqiActivity", ARouter$$Group$$coolqiActivity.class);
routes.put("coolqiProvider", ARouter$$Group$$coolqiProvider.class);
routes.put("coolqiService", ARouter$$Group$$coolqiService.class);
```

这里会从 Warehouse.groupsIndex 中，获取 group 对应的类，并创建实例，比如 `ARouter$$Group$$coolqiService`：

```java
public class ARouter$$Group$$coolqiService implements IRouteGroup {
  @Override
  public void loadInto(Map<String, RouteMeta> atlas) {
    atlas.put("/coolqiService/MySerializationService", RouteMeta.build(RouteType.PROVIDER, MySerializationService.class, 
                                                                       "/coolqiservice/myserializationservice", "coolqiservice", 
                                                                       null, -1, -2147483648));
  }
}
```

然后，调用其 loadInto 方法，就会将数据加入到 `Warehouse.routes` 中；

- **Warehouse.providers**

**上面涉及到了数据处理，我们先不看，后面再分析**；

## 4.2 WareHouse - 核心二号种子

WareHouse 是 ARouter 的数据仓库，存储跳转的信息！

### 4.2.1 成员变量

```java
//【1】保存动态生成的 ARouter$$Root$$moduleName.loadInto 方法加载的数据；
// 相当于，我们把 compiler 编译生成的数据保存到了这里；
static Map<String, Class<? extends IRouteGroup>> groupsIndex = new HashMap<>();
static Map<String, RouteMeta> routes = new HashMap<>(); // 上面数据的缓存；

//【2】保存动态生成的 ARouter$$Providers$$moduleName.loadInto 方法加载的数据；
// 相当于，我们把 compiler 编译生成的数据保存到了这里；
static Map<String, RouteMeta> providersIndex = new HashMap<>();
static Map<Class, IProvider> providers = new HashMap<>(); // 上面数据的缓存，key：类名，value：实例；

//【3】保存动态生成的 ARouter$$Interceptors$$moduleName.loadInto 方法加载的数据；【-->5.1】对于 interceptor，是通过 UniqueKeyTreeMap 来存放的！
// 相当于，我们把 compiler 编译生成的数据保存到了这里；
static Map<Integer, Class<? extends IInterceptor>> interceptorsIndex = new UniqueKeyTreeMap<>("More than one interceptors use same priority [%s]");
static List<IInterceptor> interceptors = new ArrayList<>(); // 上面数据的缓存
```

每次都会从 compiler 数据中获取数据保存到缓存数据中，然后删除  compiler 数据；



### 4.2.2 clear

清除内部数据：

```java
    static void clear() {
        routes.clear();
        groupsIndex.clear();
        providers.clear();
        providersIndex.clear();
        interceptors.clear();
        interceptorsIndex.clear();
    }
```

WareHouse 只有一个 clear 方法，用来清除数据；



## 4.3 InterceptorServiceImpl

InterceptorServiceImpl 他是 ARouter 内部实现的系统服务，是通过：`@Route(path = "/arouter/service/interceptor")` 注解处理的，它的作用是**用于处理拦截器**

```java
@Route(path = "/arouter/service/interceptor")
public class InterceptorServiceImpl implements InterceptorService {
    private static boolean interceptorHasInit;
    private static final Object interceptorInitLock = new Object();
    
}
```



### 4.3.1 init

初始化所有的 Interceptor：

```java
    @Override
    public void init(final Context context) {
        LogisticsCenter.executor.execute(new Runnable() {
            @Override
            public void run() {
                if (MapUtils.isNotEmpty(Warehouse.interceptorsIndex)) {
                    //【-->4.2.1】从 Warehouse.interceptorsIndex 获取所有注解生成的拦截器；
                    for (Map.Entry<Integer, Class<? extends IInterceptor>> entry : Warehouse.interceptorsIndex.entrySet()) {
                        Class<? extends IInterceptor> interceptorClass = entry.getValue();
                        try {
                            //【1】创建 interceptors 实例，并执行 init 初始化；
                            IInterceptor iInterceptor = interceptorClass.getConstructor().newInstance();
                            iInterceptor.init(context);
                            //【-->4.2.1】将其加入到缓存 Warehouse.interceptors
                            Warehouse.interceptors.add(iInterceptor);
                        } catch (Exception ex) {
                            throw new HandlerException(TAG + "ARouter init interceptor error! name = [" 
                                                       + interceptorClass.getName() + "], reason = [" + ex.getMessage() + "]");
                        }
                    }

                    interceptorHasInit = true;

                    logger.info(TAG, "ARouter interceptors init over.");

                    synchronized (interceptorInitLock) {
                        interceptorInitLock.notifyAll();
                    }
                }
            }
        });
    }
```

这里又涉及到缓存：**Warehouse.interceptorsIndex —>Warehouse.interceptors**;



#  5 base 包

## 5.1 UniqueKeyTreeMap

```java
public class UniqueKeyTreeMap<K, V> extends TreeMap<K, V> {
    private String tipText;

    public UniqueKeyTreeMap(String exceptionText) {
        super();

        tipText = exceptionText;
    }

    @Override
    public V put(K key, V value) {
        if (containsKey(key)) {
            throw new RuntimeException(String.format(tipText, key));
        } else {
            return super.put(key, value);
        }
    }
}
```

不多说了！



# 6 facade 包

## 6.1 PostCard

PostCard 继承了 RouteMeta，用于保存跳转的信息；

### 6.1.1 成员变量

```java
    private Uri uri;                // uri
    private Object tag;             // A tag prepare for some thing wrong.
    private Bundle mBundle;         // 数据 bundle；
    private int flags = -1;         // 跳转的 flags，用于 activity；
    private int timeout = 300;      // Navigation timeout, TimeUnit.Second
    private IProvider provider;     // 如果跳转的目标实例是 IProvider 的子类，那么该值不为 null；
    private boolean greenChannel;   // 是否跳过所有的拦截器；
    private SerializationService serializationService;

    private Bundle optionsCompat;    // 和动画相关的属性；
    private int enterAnim = -1;
    private int exitAnim = -1;
```

### 6.1.2 new Postcard

创建了一个 Postcard 实例！

```java
    public Postcard() {
        this(null, null);
    }

    public Postcard(String path, String group) {
        this(path, group, null, null);
    }
    //【1】最终会调用这个方法；
    public Postcard(String path, String group, Uri uri, Bundle bundle) {
        setPath(path); // 设置 path
        setGroup(group); // 设置 group
        setUri(uri); // 设置 uri
        this.mBundle = (null == bundle ? new Bundle() : bundle);
    }
```

不多说了！

在初始化过程中，我们传入的 path："/arouter/service/interceptor"

###  6.1.3 navigation

在初始化过程中，执行 navigation 方法，获取系统拦截器：**InterceptorServiceImpl** 

```java
    public Object navigation() {
        return navigation(null);
    }
    
    public Object navigation(Context context) {
        return navigation(context, null);
    }

    public Object navigation(Context context, NavigationCallback callback) {
        //【-->3.1.5】调用了 ARouter 的 navigation 方法！
        return ARouter.getInstance().navigation(context, this, -1, callback);
    }
```

这里涉及到了 **NavigationCallback callback** 的概念：**跳转回调**！ 

# 9 Utils 包

## 9.1 ClassUtils

### 9.1.1 成员变量

```java
    private static final String EXTRACTED_NAME_EXT = ".classes"; // 文件后缀
    private static final String EXTRACTED_SUFFIX = ".zip";

    private static final String SECONDARY_FOLDER_NAME = "code_cache" + File.separator + "secondary-dexes"; // 文件路径 /code_cache/secondary-dexes；

    private static final String PREFS_FILE = "multidex.version"; // 记录 mutilDex 信息的 sp name；
    private static final String KEY_DEX_NUMBER = "dex.number"; // sp key 值，记录 dex 的数量；

    private static final int VM_WITH_MULTIDEX_VERSION_MAJOR = 2; // VM 相关，用于判断 vm 是否支持 multiDex；
    private static final int VM_WITH_MULTIDEX_VERSION_MINOR = 1;
```



### 9.1.2 getFileNameByPackageName

通过指定包名，扫描 Apk 下面包含的所有类的 ClassName:

```java
    public static Set<String> getFileNameByPackageName(Context context, final String packageName) 
      																													throws PackageManager.NameNotFoundException, IOException, InterruptedException {
        final Set<String> classNames = new HashSet<>();
        //【-->9.1.3】获取 apk 源代码（dex）的路径；
        List<String> paths = getSourcePaths(context);
        final CountDownLatch parserCtl = new CountDownLatch(paths.size());

        for (final String path : paths) {
            DefaultPoolExecutor.getInstance().execute(new Runnable() {
                @Override
                public void run() {
                    DexFile dexfile = null;

                    try {
                        //【2】根据文件后缀的不同，会执行不通的操作，如果后缀是 .zip，那么回调用 DexFile.loadDex 方法；
                        if (path.endsWith(EXTRACTED_SUFFIX)) { 
                            dexfile = DexFile.loadDex(path, path + ".tmp", 0);
                        } else {
                            dexfile = new DexFile(path);
                        }
                        //【3】遍历 dex 文件，找到该 apk 的所有 class，并返回其 class name；
                        Enumeration<String> dexEntries = dexfile.entries();
                        while (dexEntries.hasMoreElements()) {
                            String className = dexEntries.nextElement();
                            if (className.startsWith(packageName)) {
                                classNames.add(className);
                            }
                        }
                    } catch (Throwable ignore) {
                        Log.e("ARouter", "Scan map file in dex files made error.", ignore);
                    } finally {
                        if (null != dexfile) {
                            try {
                                dexfile.close();
                            } catch (Throwable ignore) {
                            }
                        }
                        parserCtl.countDown();
                    }
                }
            });
        }

        parserCtl.await();

        Log.d(Consts.TAG, "Filter " + classNames.size() + " classes by packageName <" + packageName + ">");
        return classNames;
    }
```

### 9.1.3 getSourcePaths

获取源代码路径：

```java
    public static List<String> getSourcePaths(Context context) throws PackageManager.NameNotFoundException, IOException {
        ApplicationInfo applicationInfo = context.getPackageManager().getApplicationInfo(context.getPackageName(), 0);
        File sourceApk = new File(applicationInfo.sourceDir);

        List<String> sourcePaths = new ArrayList<>();
        sourcePaths.add(applicationInfo.sourceDir);//【1】添加默认的 apk 源路径；

        //【1】文件前缀：name.classes
        String extractedFilePrefix = sourceApk.getName() + EXTRACTED_NAME_EXT;

        //【-->9.1.3.1】判断 vm 是否支持 multiDex，如何已经支持了 muitiDex，那就不去 Secondary Folder 加载 Classesx.zip
        if (!isVMMultidexCapable()) {
            //【2】不支持 multiDex，那就要去加载 Classesx.zip；
            int totalDexNumber = getMultiDexPreferences(context).getInt(KEY_DEX_NUMBER, 1); //【-->9.1.3.2】获取 dex 的数量；
            File dexDir = new File(applicationInfo.dataDir, SECONDARY_FOLDER_NAME); // 获取存放其他 dex 的目录；
            //【3】收集 Secondary Folder 目录下的 dex 的路径；
            for (int secondaryNumber = 2; secondaryNumber <= totalDexNumber; secondaryNumber++) {
                //【3.1】每个 dex file 的文件名都是：name.classes.zip，添加到 sourcePaths 列表中；
                String fileName = extractedFilePrefix + secondaryNumber + EXTRACTED_SUFFIX;
                File extractedFile = new File(dexDir, fileName);
                if (extractedFile.isFile()) {
                    sourcePaths.add(extractedFile.getAbsolutePath());
                    //we ignore the verify zip part
                } else {
                    throw new IOException("Missing extracted secondary dex file '" + extractedFile.getPath() + "'");
                }
            }
        }

        if (ARouter.debuggable()) { // Search instant run support only debuggable
            sourcePaths.addAll(tryLoadInstantRunDexFile(applicationInfo));
        }
        //【4】返回收集的列表
        return sourcePaths;
    }
```

过程还是比较简单的；



#### 9.1.3.1 isVMMultidexCapable

判断 vm 是否支持 multiDex：

```java
   private static boolean isVMMultidexCapable() {
        boolean isMultidexCapable = false;
        String vmName = null;

        try {
            if (isYunOS()) { //【1】YunOS 需要特殊判断
                vmName = "'YunOS'";
                isMultidexCapable = Integer.valueOf(System.getProperty("ro.build.version.sdk")) >= 21;
            } else { //【2】非 YunOS 原生 Android
                vmName = "'Android'";
                String versionString = System.getProperty("java.vm.version");
                if (versionString != null) {
                    //【3】判断 java.vm.version 属性的 major 和 minor 的范围；
                    Matcher matcher = Pattern.compile("(\\d+)\\.(\\d+)(\\.\\d+)?").matcher(versionString);
                    if (matcher.matches()) {
                        try {
                            int major = Integer.parseInt(matcher.group(1));
                            int minor = Integer.parseInt(matcher.group(2));
                            isMultidexCapable = (major > VM_WITH_MULTIDEX_VERSION_MAJOR)
                                    || ((major == VM_WITH_MULTIDEX_VERSION_MAJOR)
                                    && (minor >= VM_WITH_MULTIDEX_VERSION_MINOR));
                        } catch (NumberFormatException ignore) {
                            // let isMultidexCapable be false
                        }
                    }
                }
            }
        } catch (Exception ignore) {

        }

        Log.i(Consts.TAG, "VM with name " + vmName + (isMultidexCapable ? " has multidex support" : " does not have multidex support"));
        return isMultidexCapable;
    }
```

不多说了 ；

#### 9.1.3.2 getMultiDexPreferences

获取记录 multi dex 信息 sp，ARouter 将 dex 的数量保存到内部 sp 中，name： "multidex.version"：

```java
   private static SharedPreferences getMultiDexPreferences(Context context) {
        return context.getSharedPreferences(PREFS_FILE, Build.VERSION.SDK_INT < Build.VERSION_CODES.HONEYCOMB 
                                            ? Context.MODE_PRIVATE : Context.MODE_PRIVATE | Context.MODE_MULTI_PROCESS);
    }
```

## 9.2 PackageUtils

### 9.2.1 成员变量

```java
    private static String NEW_VERSION_NAME;
    private static int NEW_VERSION_CODE;
```

用来缓存 apk 的版本号和版本名；

### 9.2.2 isNewVersion

判断 apk 是否是新的版本，包括第一次安装/更新安装；

```java
    public static boolean isNewVersion(Context context) {
        //【1】获得 apk 的信息；
        PackageInfo packageInfo = getPackageInfo(context);
        if (null != packageInfo) {
            //【2】获取 apk 的版本名和版本号；
            String versionName = packageInfo.versionName;
            int versionCode = packageInfo.versionCode;

            SharedPreferences sp = context.getSharedPreferences(AROUTER_SP_CACHE_KEY, Context.MODE_PRIVATE);
            //【3】如何和本地缓存的不一样，那就说明是新版本；
            if (!versionName.equals(sp.getString(LAST_VERSION_NAME, null)) || versionCode != sp.getInt(LAST_VERSION_CODE, -1)) {
                //【3.1】将新的 versionCode 和 VersionName 缓存下来；
                NEW_VERSION_NAME = versionName;
                NEW_VERSION_CODE = versionCode;

                return true;
            } else {
                return false;
            }
        } else {
            return true;
        }
    }
```

### 9.2.3 updateVersion

更新本地缓存：

```java
    public static void updateVersion(Context context) {
        if (!android.text.TextUtils.isEmpty(NEW_VERSION_NAME) && NEW_VERSION_CODE != 0) {
            SharedPreferences sp = context.getSharedPreferences(AROUTER_SP_CACHE_KEY, Context.MODE_PRIVATE);
            //【1】写入到本地 sp 中；
            sp.edit().putString(LAST_VERSION_NAME, NEW_VERSION_NAME).putInt(LAST_VERSION_CODE, NEW_VERSION_CODE).apply();
        }
    }
```

这里的 **LAST_VERSION_NAME**，**LAST_VERSION_CODE** 均定义在 Consts 中；

## 9.x Consts

常量类，保存了 arouter-api 中的常量关键字；

```java
public final class Consts {
    public static final String SDK_NAME = "ARouter"; // 这几个常量在 complier 中有见过，用于生成注解处理后的类的类名和包名；
    public static final String TAG = SDK_NAME + "::";
    public static final String SEPARATOR = "$$";
    public static final String SUFFIX_ROOT = "Root";
    public static final String SUFFIX_INTERCEPTORS = "Interceptors";
    public static final String SUFFIX_PROVIDERS = "Providers";
    public static final String SUFFIX_AUTOWIRED = SEPARATOR + SDK_NAME + SEPARATOR + "Autowired";
    public static final String DOT = ".";
    public static final String ROUTE_ROOT_PAKCAGE = "com.alibaba.android.arouter.routes";

    public static final String AROUTER_SP_CACHE_KEY = "SP_AROUTER_CACHE"; // 本地缓存 sp 的 name
    public static final String AROUTER_SP_KEY_MAP = "ROUTER_MAP";

    public static final String LAST_VERSION_NAME = "LAST_VERSION_NAME"; // 用于保存 apk 的版本号
    public static final String LAST_VERSION_CODE = "LAST_VERSION_CODE";
}
```





# 10 总结

我们分析路由初始化的整个过程，设计的 pkg 也很多，但是细心的观察，我们其实已经分析了一些路由跳转的逻辑，哈哈哈。

当然，还有下面的问题遗漏了：

- `PathReplaceService，PretreatmentService，DegradeService `：是如何获取的，作用又是什么呢？
- `facade.service` 下的这些 `Service` 都是如何工作的呢？
- `core`目录下的 `AutowiredServiceImpl` 和 `InterceptorServiceImpl`，又是如何工作的呢？
- ARouter 如何处理跳转回调的呢？

这些问题，我会在下篇：路由跳转中分析；

