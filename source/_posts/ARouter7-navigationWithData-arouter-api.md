# ARouter 第七篇 - 路由跳转/数据传递 (arouter-api)

title: ARouter 第七篇 - 路由跳转/数据传递 (arouter-api)
date: 2019/05/09 20:46:25 
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

本篇博文主要分析 arouter-api 模块的路由跳转的过程，以及变量/数据的传递，这篇文章将是本系列的最后一篇（后续会抽时间写其他的）！

在阅读过程中，涉及到方法跳转的时候，注释上有 `-->`的标志，这样的好处是，以类为单位，一次性分析其所有的方法：

#  1 路由跳转

我们来看看

## 1.1 跳转方式

ARouter 支持两种方式来跳转：

- path 跳转

```java
ARouter.getInstance().build("/home/main")  // 指定 path
            .navigation();
ARouter.getInstance().build("/home/main", "ap").navigation(); // 显示指定分组
```

- uri 跳转

```java
Uri uri;
ARouter.getInstance().build(uri).navigation(); // 指定 uri
```

- 我们可以设置跳转请求码和跳转回调

这种调用方式相当于原生的 startActivityForResult：

```java
ARouter.getInstance().build("/home/main", "ap").navigation(this, 5);
```
同时我们也可以指定跳转回调：**NavigationCallback** 

```java
ARouter.getInstance().build("/test/activity").navigation(this, new NavigationCallback() {
  @Override
  public void onFound(Postcard postcard) {
  }

  @Override
  public void onLost(Postcard postcard) {
  }

  @Override
  public void onArrival(Postcard postcard) {
  }

  @Override
  public void onInterrupt(Postcard postcard) {
  }
});
```

处理跳转的结果；

- 我们也可以设置跳过所有的拦截器

我们知道 actiivty 的跳转是收到拦截器的限制的，但是 PostCard 提供了接口，能够跳过所有的拦截器：

```java
// 使用绿色通道(跳过所有的拦截器)
ARouter.getInstance().build("/home/main").greenChannel().navigation();
```

这里的 greenChannel 方法我们前面有分析过，不多说了！

### 1.1.1 Uri 跳转的特殊性

这里要单独讲下 uri 跳转的特殊性，ARouter 通过新建一个没有 UI 的界面作为跳板来统一处理，scheme 是 arouter 的跳转请求！

- 需要新建一个 activity 来接收 uri，没有 ui 界面，这是关键点！

```java
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

- AndroidManifest.xml 中要指定该 activity 监听的 schame 事件是：**arouter://m.aliyun.com**

```xml
<activity android:name=".activity.SchameFilterActivity">
    <!-- Schame -->
    <intent-filter>
        <data
            android:host="m.aliyun.com"
            android:scheme="arouter"/>
        <action android:name="android.intent.action.VIEW"/>
        <category android:name="android.intent.category.DEFAULT"/>
        <category android:name="android.intent.category.BROWSABLE"/>   <!-- 让浏览器可以打这个 Actvity，当然要符合 host 和 scheme -->
    </intent-filter>
    <!-- App Links -->
    <intent-filter android:autoVerify="true">
        <action android:name="android.intent.action.VIEW"/>
        <category android:name="android.intent.category.DEFAULT"/>
        <category android:name="android.intent.category.BROWSABLE"/>   <!-- 让浏览器可以打这个 Actvity，当然要符合 host 和 scheme -->
        <data
            android:host="m.aliyun.com"
            android:scheme="http"/>
        <data
            android:host="m.aliyun.com"
            android:scheme="https"/>
    </intent-filter>
</activity>
```

相当于这个 activity 作为外界的统一入口，H5 或者 native 通过 intent 匹配，将数据传递给这个 activity，然后这个 activity 解析数据，将 uri 叫给 ARouter 最终实现跳转！

详细分析：Uri 的组成为：**scheme://host:port/path?query**，我们通过中转 activity 匹配 **scheme://host:port** 部分，截获 Uri，然后通过 getPath 就可以回去到 Path，这个 Path 需要和 @Route 的 path 匹配，这样就可以实现跳转了！

可以看出这个过程就是 **intent 的匹配！**

## 1.2 数据传递

以上的两种跳转方式，都可以传递数据，我们来看下数据传递的方式：

- 直接传递 Bundle

```java
Bundle params = new Bundle();
ARouter.getInstance()
    .build("/home/main")
    .with(params)
    .navigation();
```

注意：这个方法会覆盖掉 PostCast 内部默认创建的 Bundle；

- 指定启动 Flag

```java
ARouter.getInstance()
    .build("/home/main")
    .withFlags();
    .navigation();
```

- 支持传递各种基本类型数据、对象、数组、List、可序列化数据：

```java
ARouter.getInstance()
    .build("/home/main").withAction(..)
    .withBoolean(String key, boolean value)
    .withBundle(String key, Bundle value)
    .withByte(String key, byte value)
    .withCharSequenceArrayList(String key, ArrayList<CharSequence> value)
    .withParcelable(String key, Parcelable value)
    .withSerializable(String key, Serializable value)
    .withStringArrayList(String key, ArrayList<String> value)
    .withObject(@Nullable String key, @Nullable Object value)
    ... ... ... // 接口太多了，省略下，其实这些接口对应的是 Bundle 中的方法！
    .navigation();
```

这些数据都会被加入到 PostCard 内部的默认创建的 Bundle 中，其实这些方法对应的就是 Bundle 中的方法！

- 支持设置转场动画

```java
// 转场动画(常规方式)
ARouter.getInstance()
    .build("/test/activity2")
    .withTransition(R.anim.slide_in_bottom, R.anim.slide_out_bottom)
    .navigation(this);

// 转场动画(API16+)
ActivityOptionsCompat compat = ActivityOptionsCompat.
    makeScaleUpAnimation(v, v.getWidth() / 2, v.getHeight() / 2, 0, 0);
ARouter.getInstance()
    .build("/test/activity2")
    .withOptionsCompat(compat)
    .navigation();
```



### 1.2.1 对象传递的特殊

对于自定义的对象，不能确保它可序列化，所以这里通过 SerializationService 将其转为了 jsonstring：

```java
    public Postcard withObject(@Nullable String key, @Nullable Object value) {
        serializationService = ARouter.getInstance().navigation(SerializationService.class);
        mBundle.putString(key, serializationService.object2Json(value));
        return this;
    }
```

# 2 跳转流程

下面，我们重点分析路由跳转的流程，和数据传递的流程，忽略掉一些之前已经见过的流程！

```java
ARouter.getInstance().build(...); --> _ARouter.getInstance().build(...);
```

无论是 path 跳转，还是 uri 跳转，ARouter 都会调用 _ARouter 的方法！

## 2.1 _ARouter.build

无论是 build(path)，还是  build(uri)，最终创建的 PostCard 都是一样的！

```java
    protected Postcard build(String path) {
        if (TextUtils.isEmpty(path)) {
            throw new HandlerException(Consts.TAG + "Parameter is invalid!");
        } else {
            PathReplaceService pService = ARouter.getInstance().navigation(PathReplaceService.class);
            if (null != pService) {
                path = pService.forString(path);
            }
            return build(path, extractGroup(path));  // 通过 path 创建！
        }
    }

    protected Postcard build(Uri uri) {
        if (null == uri || TextUtils.isEmpty(uri.toString())) {
            throw new HandlerException(Consts.TAG + "Parameter invalid!");
        } else {
            PathReplaceService pService = ARouter.getInstance().navigation(PathReplaceService.class);
            if (null != pService) {
                uri = pService.forUri(uri);
            }
            return new Postcard(uri.getPath(), extractGroup(uri.getPath()), uri, null); // 通过 uri 创建！
        }
    }
```



## 2.2 Postcard



### 2.2.1 new Postcard

创建一个新的 Postcard：

```java
    public Postcard(String path, String group) {
        this(path, group, null, null);
    }

    public Postcard(String path, String group, Uri uri, Bundle bundle) {
        setPath(path);
        setGroup(group);
        setUri(uri);
        this.mBundle = (null == bundle ? new Bundle() : bundle);
    }
```

可以看到，通过 Uri 创建的话，会多设置一个 Uri 的属性；



### 2.2.2 navigation

最核心的就后面的两个方法，支持传入 requestCode 和 NavigationCallback 实例：

```java
    public Object navigation() {
        return navigation(null);
    }

    public Object navigation(Context context) {
        return navigation(context, null);
    }

    public void navigation(Activity mContext, int requestCode) {
        navigation(mContext, requestCode, null);
    }

    public Object navigation(Context context, NavigationCallback callback) {
        return ARouter.getInstance().navigation(context, this, -1, callback);
    }

    public void navigation(Activity mContext, int requestCode, NavigationCallback callback) {
        ARouter.getInstance().navigation(mContext, this, requestCode, callback);
    }
```

对于 ARouter.getInstance().navigation，我们知道最后会调用 _ARouter.getInstance().navigation

```java
ARouter.getInstance().navigation(...) --> _ARouter.getInstance().navigation(...);
```

## 2.3 _ARouter.navigation

这里我们可以看到回调的处理：

```java
protected Object navigation(final Context context, final Postcard postcard, final int requestCode, final NavigationCallback callback) {
    //【1】这里的获取方式是一样的；
    PretreatmentService pretreatmentService = ARouter.getInstance().navigation(PretreatmentService.class);
    if (null != pretreatmentService && !pretreatmentService.onPretreatment(context, postcard)) {
        return null;
    }
    try {
        //【-->2.3.1】完善跳转信息；
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
            //【-->2.4.2】完善跳转信息失败后会调用，通过 NavigationCallback 通知；
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
        //【-->2.4.1】完善跳转信息成功后会调用，通过 NavigationCallback 通知；
        callback.onFound(postcard);
    }
    //【3】对于 Service/Fragment 是会跳过拦截器的，对于 activity 默认是不会跳过的，当然了可动态设置；
    if (!postcard.isGreenChannel()) {
        interceptorService.doInterceptions(postcard, new InterceptorCallback() {
            @Override
            public void onContinue(Postcard postcard) {
                //【-->2.3.2】最终的处理；
                _navigation(context, postcard, requestCode, callback);
            }
            @Override
            public void onInterrupt(Throwable exception) {
                if (null != callback) {
                    //【-->2.4.4】拦截器对跳转进行了拦截后会调用，通过 NavigationCallback 通知；
                    callback.onInterrupt(postcard);
                }
                logger.info(Consts.TAG, "Navigation failed, termination by interceptor : " + exception.getMessage());
            }
        });
    } else {
        //【-->2.3.2】最终的处理；
        return _navigation(context, postcard, requestCode, callback);
    }
    return null;
}
```





### 2.3.1 LogisticsCenter.completion

完善登陆信息，这里前面有说过：

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
        if (null == groupMeta) { //【2.1.1】找不到抛出异常；
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
                    logger.debug(TAG, String.format(Locale.getDefault(), "The group [%s] has already been loaded, trigger by [%s]",
                                                    postcard.getGroup(), postcard.getPath()));
                }
            } catch (Exception e) {
                throw new HandlerException(TAG + "Fatal exception when loading group meta. [" + e.getMessage() + "]");
            }
            //【-->2.3.1】重新加载；
            completion(postcard);
        }
    } else {
        //【-->2.3.1.1】这里是通过 RouteMeta 来设置 Postcard 对象，我们先不看；
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
        //【4】这里是关键点，判断类型，可以看到 activity 这里是不处理的，主要是 IProvider 类型；
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

这里是

####  2.3.1.1  数据传递 - important

这里我们要重点看下 PostCard 中的数据是如何处理的：

```java
postcard.setDestination(routeMeta.getDestination());
postcard.setType(routeMeta.getType()); 
postcard.setPriority(routeMeta.getPriority());
postcard.setExtra(routeMeta.getExtra());

//【1】注意这里是处理 uri 的数据
Uri rawUri = postcard.getUri();
if (null != rawUri) { 
  //【-->2.3.1.2】获得 uri 的数据；
  Map<String, String> resultMap = TextUtils.splitQueryParameters(rawUri);
  //【2】RouteMeta 的 paramsType 保存了被 @AutoWired 注解的参数和类型的枚举序号的映射关系；
  Map<String, Integer> paramsType = routeMeta.getParamsType(); 
  if (MapUtils.isNotEmpty(paramsType)) {
    for (Map.Entry<String, Integer> params : paramsType.entrySet()) {
      //【-->2.3.1.3】将传递的数据设置到 Postcard 中，调用 Postcard.withXXX 方法；
      setValue(postcard,
               params.getValue(), // 变量类型的枚举序号；
               params.getKey(), // 变量名/注解name值
               resultMap.get(params.getKey())); // uri 写到的变量的值；
    }
    //【3】将被 @AutoWired 注解的变量的变量名/注解name值以 String[] 形式保存到 PostCard.Bundle 中；
    postcard.getExtras().putStringArray(ARouter.AUTO_INJECT, paramsType.keySet().toArray(new String[]{}));
  }
  //【4】将 uri 保存到 PostCard.Bundle 中，
  postcard.withString(ARouter.RAW_URI, rawUri.toString());
}
```

上面的两个常量的如下：


```java
public static final String RAW_URI = "NTeRQWvye18AkPd6G";
public static final String AUTO_INJECT = "wmHzgD4lOj5o4241";
```



#### 2.3.1.2 TextUtils.splitQueryParameters

将 uri 后缀保存的 key-value 的键值对转为 Map<String, String>：

```java
    public static Map<String, String> splitQueryParameters(Uri rawUri) {
        String query = rawUri.getEncodedQuery();

        if (query == null) {
            return Collections.emptyMap();
        }

        Map<String, String> paramMap = new LinkedHashMap<>();
        int start = 0;
        do {
            int next = query.indexOf('&', start);
            int end = (next == -1) ? query.length() : next;

            int separator = query.indexOf('=', start);
            if (separator > end || separator == -1) {
                separator = end;
            }

            String name = query.substring(start, separator);

            if (!android.text.TextUtils.isEmpty(name)) {
                String value = (separator == end ? "" : query.substring(separator + 1, end));
                paramMap.put(Uri.decode(name), Uri.decode(value));
            }

            // Move start to end of name.
            start = end + 1;
        } while (start < query.length());

        return Collections.unmodifiableMap(paramMap);
    }
```



#### 2.3.1.3 LogisticsCenter.setValue

将传递的数据设置进入 Postcard 中！

```java
    private static void setValue(Postcard postcard, Integer typeDef, String key, String value) {
        if (TextUtils.isEmpty(key) || TextUtils.isEmpty(value)) {
            return;
        }

        try {
            if (null != typeDef) {
                //【1】根据类型的枚举序号，匹配对应的 withXXX 方法，将值设置到内部的 Bundle 中！
                if (typeDef == TypeKind.BOOLEAN.ordinal()) {
                    postcard.withBoolean(key, Boolean.parseBoolean(value));
                } else if (typeDef == TypeKind.BYTE.ordinal()) {
                    postcard.withByte(key, Byte.valueOf(value));
                } else if (typeDef == TypeKind.SHORT.ordinal()) {
                    postcard.withShort(key, Short.valueOf(value));
                } else if (typeDef == TypeKind.INT.ordinal()) {
                    postcard.withInt(key, Integer.valueOf(value));
                } else if (typeDef == TypeKind.LONG.ordinal()) {
                    postcard.withLong(key, Long.valueOf(value));
                } else if (typeDef == TypeKind.FLOAT.ordinal()) {
                    postcard.withFloat(key, Float.valueOf(value));
                } else if (typeDef == TypeKind.DOUBLE.ordinal()) {
                    postcard.withDouble(key, Double.valueOf(value));
                } else if (typeDef == TypeKind.STRING.ordinal()) {
                    postcard.withString(key, value);
                } else if (typeDef == TypeKind.PARCELABLE.ordinal()) {
                    // TODO : How to description parcelable value with string?
                } else if (typeDef == TypeKind.OBJECT.ordinal()) {
                    postcard.withString(key, value);
                } else { 
                    // Compatible compiler sdk 1.0.3, in that version, the string type = 18
                    postcard.withString(key, value);
                }
            } else {
                postcard.withString(key, value);
            }
        } catch (Throwable ex) {
            logger.warning(Consts.TAG, "LogisticsCenter setValue failed! " + ex.getMessage());
        }
    }
```

方法很简单，不多说了！



### 2.3.2 \_ARouter.\_navigation

可以看到，启动过的过程就是将 Postcard 中的数据设置到 intent 中：

```java
    private Object _navigation(final Context context, final Postcard postcard, final int requestCode, final NavigationCallback callback) {
        final Context currentContext = null == context ? mContext : context;
        //【1】根据跳转类型处理不同的目标；
        switch (postcard.getType()) {
            case ACTIVITY:
                //【1.1】创建 activity；
                final Intent intent = new Intent(currentContext, postcard.getDestination());
                intent.putExtras(postcard.getExtras());

                //【1.2】设置目标启动的 flags
                int flags = postcard.getFlags();
                if (-1 != flags) {
                    intent.setFlags(flags);
                } else if (!(currentContext instanceof Activity)) {    // Non activity, need less one flag.
                    intent.setFlags(Intent.FLAG_ACTIVITY_NEW_TASK);
                }

                //【1.3】设置 action！
                String action = postcard.getAction();
                if (!TextUtils.isEmpty(action)) {
                    intent.setAction(action);
                }

                //【-->2.3.2.1】执行启动；
                runInMainThread(new Runnable() {
                    @Override
                    public void run() {
                        startActivity(requestCode, currentContext, intent, postcard, callback);
                    }
                });

                break;
            case PROVIDER:
                return postcard.getProvider(); // 这个是针对 Provider 的；
            case BOARDCAST:
            case CONTENT_PROVIDER:
            case FRAGMENT:
                //【1.4】针对于 brocastreceiver，contenprovider，fragment，会拿到其实例！
                Class fragmentMeta = postcard.getDestination();
                try {
                    Object instance = fragmentMeta.getConstructor().newInstance();
                    //【1.4.1】针对于 fragment，还会设置 Arguments；
                    if (instance instanceof Fragment) {
                        ((Fragment) instance).setArguments(postcard.getExtras());
                    } else if (instance instanceof android.support.v4.app.Fragment) {
                        ((android.support.v4.app.Fragment) instance).setArguments(postcard.getExtras());
                    }

                    return instance;
                } catch (Exception ex) {
                    logger.error(Consts.TAG, "Fetch fragment instance error, " + TextUtils.formatStackTrace(ex.getStackTrace()));
                }
            case METHOD: // 其他情况没有；
            case SERVICE:
            default:
                return null;
        }

        return null;
    }
```

针对于 brocastreceiver，contenprovider，fragment，会拿到其实例，针对于 fragment，还会设置 Arguments！

#### 2.3.2.1 _ARouter.startActivity

这就是最后启动过程了，其实很简单：

```java
    private void startActivity(int requestCode, Context currentContext, Intent intent, Postcard postcard, NavigationCallback callback) {
        //【1】这里根据 requestCode 有不同的调用方式；
        if (requestCode >= 0) {  // Need start for result
            if (currentContext instanceof Activity) {
                ActivityCompat.startActivityForResult((Activity) currentContext, intent, requestCode, postcard.getOptionsBundle());
            } else {
                logger.warning(Consts.TAG, "Must use [navigation(activity, ...)] to support [startActivityForResult]");
            }
        } else {
            ActivityCompat.startActivity(currentContext, intent, postcard.getOptionsBundle());
        }

        if ((-1 != postcard.getEnterAnim() && -1 != postcard.getExitAnim()) && currentContext instanceof Activity) {    // Old version.
            ((Activity) currentContext).overridePendingTransition(postcard.getEnterAnim(), postcard.getExitAnim());
        }

        if (null != callback) {
            //【-->2.4.3】启动完成，回调 NavigationCallback
            callback.onArrival(postcard);
        }
    }
```

不多说了！

这里的 postcard.getOptionsBundle() 会返回一个 Bundle 是用来保存额外的启动参数，比如动画等等；

```java
    private Bundle optionsCompat;     
		public Bundle getOptionsBundle() {
        return optionsCompat;
    }
```



## 2.4 跳转回调

我们看看 NavigationCallback 的相关接口：

```java
public interface NavigationCallback {
    //【-->2.4.1】完善跳转信息成功后会调用；
    void onFound(Postcard postcard);

    //【-->2.4.2】完善跳转信息失败后会调用；
    void onLost(Postcard postcard);

    //【-->2.4.3】跳转成功后会回调；
    void onArrival(Postcard postcard);

    //【-->2.4.4】拦截器对跳转进行了拦截后会调用；
    void onInterrupt(Postcard postcard);
}
```

# 3 总结

到这里 ARouter 分析就暂告一段落了；