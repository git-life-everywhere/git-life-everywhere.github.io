# ARouter 第三篇 - 注解解析 (arouter-compiler)

title: ARouter 第三篇 - 注解解析 (arouter-compiler)
date: 2019/04/19 20:46:25 
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

本篇博文主要分析 arouter-compiler 模块；

# 1 模块结构

下面我们来看看  arouter-compiler  的模块结构：

```xml
|____com
| |____alibaba
| | |____android
| | | |____arouter
| | | | |____compiler
| | | | | |____entity
| | | | | | |____RouteDoc.java
| | | | | |____processor
| | | | | | |____BaseProcessor.java
| | | | | | |____InterceptorProcessor.java
| | | | | | |____AutowiredProcessor.java
| | | | | | |____RouteProcessor.java
| | | | | |____utils
| | | | | | |____TypeUtils.java
| | | | | | |____Consts.java
| | | | | | |____Logger.java
```

可以看到，一共有三个 pacakge：

- entity：包含了实体数据类；
- processor：包含了所有的注解解释器类；
- utils：包含了一些工具类；

我们知道，在 Gradle 对 App 执行编译的时候，arouter-compiler 会对相关的注解进行解析，并动态生成所需的类；

arouter-compiler 模块还依赖了两个三方库：

```groovy
implementation 'com.google.auto.service:auto-service:1.0-rc3'
implementation 'com.squareup:javapoet:1.8.0'
```

*JavaPoet* 是 square 推出的开源 java 代码生成框架，提供 Java Api 生成 .java 源文件；

auto-service 是 google 提供的用于自动注册自定义注解处理器的三方库；

关于这两个库的源码，本系列文章不分析，后面单独分析；

# 2 源码分析

我们分别分析下三个 package 目录下的 class 的作用！



## 2.1 entity

该 package 下面只包含一个实体数据类：RouteDoc。

### 2.1.1 RouteDoc

RouteDoc 用于描述路由跳转的信息，用于生成路由表：

```java
public class RouteDoc {
    @JSONField(ordinal = 1)
    private String group;
    @JSONField(ordinal = 2)
    private String path;
    @JSONField(ordinal = 3)
    private String description;
    @JSONField(ordinal = 4)
    private String prototype;
    @JSONField(ordinal = 5)
    private String className;
    @JSONField(ordinal = 6)
    private String type;
    @JSONField(ordinal = 7)
    private int mark;
    @JSONField(ordinal = 8)
    private List<Param> params;

    ... ... ...// 省略掉 get/set 方法；

    public static class Param {
        @JSONField(ordinal = 1)
        private String key;
        @JSONField(ordinal = 2)
        private String type;
        @JSONField(ordinal = 3)
        private String description;
        @JSONField(ordinal = 4)
        private boolean required;

        ... ... ...// 省略掉 get/set 方法；
    }
}
```

当 processor 对注解进行解析的时候，它会把路由跳转相关的信息记录到 RouteDoc 中！

后面我们分析 processor 的时候就可以看到了！ 



## 2.2 processor - 注解解析

该 package 下面包含 ARouter 的核心类：processors，根据前面的注解，一共有三个 processor，我们分别来分析！

重点要关注他们是如何“**解析注解，并动态生成代码**的！

### 2.2.1 BaseProcessor

BaseProcessor 是其他三个 processor 的基类，定义了一些共有的属性和操作；

```java
public abstract class BaseProcessor extends AbstractProcessor {
    Filer mFiler;
    Logger logger;
    Types types;
    Elements elementUtils; // 元素工具类对象；
    TypeUtils typeUtils;
    //【1】模块的名称 name；
    String moduleName = null;
    //【2】是否要生成 route doc；
    boolean generateDoc;

    @Override
    public synchronized void init(ProcessingEnvironment processingEnv) {
        super.init(processingEnv);

        mFiler = processingEnv.getFiler();
        types = processingEnv.getTypeUtils();
        elementUtils = processingEnv.getElementUtils();
        //【*2.3.2】创建 TypeUtils 对象，用于对类型做处理；
        typeUtils = new TypeUtils(types, elementUtils);
        //【*2.3.1】创建 Logger 对象，用于打印过程信息；
        logger = new Logger(processingEnv.getMessager());

        //【3】获取 Processor 当前所在 moudle 的 name，判断是否生成路由文档；
        // 常量第一在 Consts 中；
        Map<String, String> options = processingEnv.getOptions();
        if (MapUtils.isNotEmpty(options)) {
            moduleName = options.get(KEY_MODULE_NAME);
            generateDoc = VALUE_ENABLE.equals(options.get(KEY_GENERATE_DOC_NAME));
        }
        
        if (StringUtils.isNotEmpty(moduleName)) { // 这部分是对 moduleName 进行检查；
            moduleName = moduleName.replaceAll("[^0-9a-zA-Z_]+", "");
            logger.info("The user has configuration the module name, it was [" + moduleName + "]");
        } else {
            logger.error(NO_MODULE_NAME_TIPS);
            throw new RuntimeException("ARouter::Compiler >>> No module name, for more information, look at gradle log.");
        }
    }

    ... ... ... // getSupportedSourceVersion /getSupportedOptions

}
```

上面省略掉了一些非核心方法，我们不关注它们；

```groovy
    android {
        defaultConfig {
            ...
            javaCompileOptions {
                // 这里是核心配置点；
                annotationProcessorOptions {
                    arguments = [AROUTER_MODULE_NAME: project.getName(), AROUTER_GENERATE_DOC: "enable"]
                }
            }
        }
    }
```

上面的 KEY_MODULE_NAME，KEY_GENERATE_DOC_NAME，对应的我们在 .gradle 中的配置，这些配置最终都会解析并保存到 ProcessingEnvironment 中；

**BaseProcessor** 主要作用就是创建 TypeUtils 对象和 Logger 对象，然后获得当前所在 module 的 gradle 配置！

### 2.2.2  RouteProcessor

核心解释器，用于处理 @Route 注解：

```java
@AutoService(Processor.class)
@SupportedAnnotationTypes({ANNOTATION_TYPE_ROUTE, ANNOTATION_TYPE_AUTOWIRED})
public class RouteProcessor extends BaseProcessor {
		... ... ...
}
```

我们从成员变量，初始化，注解处理三个方面来分析：

#### 2.2.2.1 Field

内部变量；

```java
// key 是所属的组 group，value 是该组下的所有跳转信息 RouteMeta 对象；
private Map<String, Set<RouteMeta>> groupMap = new HashMap<>();
private Map<String, String> rootMap = new TreeMap<>();  // Map of root metas, used for generate class file in order.

private TypeMirror iProvider = null; // .IProvider 的类型；
private Writer docWriter;       // 用于生成路由文档；
```



#### 2.2.2.2 Init

初始化 processor：

```java
    @Override
    public synchronized void init(ProcessingEnvironment processingEnv) {
        super.init(processingEnv);

        if (generateDoc) {
            try {
                docWriter = mFiler.createResource(
                        StandardLocation.SOURCE_OUTPUT,
                        PACKAGE_OF_GENERATE_DOCS,
                        "arouter-map-of-" + moduleName + ".json"
                ).openWriter();
            } catch (IOException e) {
                logger.error("Create doc writer failed, because " + e.getMessage());
            }
        }

        iProvider = elementUtils.getTypeElement(Consts.IPROVIDER).asType();

        logger.info(">>> RouteProcessor init. <<<");
    }
```



#### 2.2.2.3 Process - 处理 Route 注解

核心逻辑：注意，这里的参数 Set<? extends TypeElement> annotations ，表示的是要处理的注解，根据前面的内容：Route 和 AutoWired ！

```java
    @Override
    public boolean process(Set<? extends TypeElement> annotations, RoundEnvironment roundEnv) {
        if (CollectionUtils.isNotEmpty(annotations)) {
            //【1】这里返回了 @Route 处理的元素；
            Set<? extends Element> routeElements = roundEnv.getElementsAnnotatedWith(Route.class);
            try {
                logger.info(">>> Found routes, start... <<<");
                //【*2.2.2.3.1】开始处理注解修饰的元素；
                this.parseRoutes(routeElements);

            } catch (Exception e) {
                logger.error(e);
            }
            return true;
        }

        return false;
    }
```

我们看到，这里调用了 parseRoutes 方法：

##### 2.2.2.3.1 parseRoutes

核心的核心：

```java
    private void parseRoutes(Set<? extends Element> routeElements) throws IOException {
        if (CollectionUtils.isNotEmpty(routeElements)) {
            logger.info(">>> Found routes, size is " + routeElements.size() + " <<<");

            rootMap.clear();
            //【1】保存 activity，service，fragment 的元素类型；
            TypeMirror type_Activity = elementUtils.getTypeElement(ACTIVITY).asType();
            TypeMirror type_Service = elementUtils.getTypeElement(SERVICE).asType();
            TypeMirror fragmentTm = elementUtils.getTypeElement(FRAGMENT).asType();
            TypeMirror fragmentTmV4 = elementUtils.getTypeElement(Consts.FRAGMENT_V4).asType();

            //【2】获得 .IProvider/.IProviderGroup 对应的 TypeElement 对象；
            TypeElement type_IRouteGroup = elementUtils.getTypeElement(IROUTE_GROUP);
            TypeElement type_IProviderGroup = elementUtils.getTypeElement(IPROVIDER_GROUP);
          	//【3】获得 RouteMeta 和 RouteType 的类全限定名；
            ClassName routeMetaCn = ClassName.get(RouteMeta.class);
            ClassName routeTypeCn = ClassName.get(RouteType.class);

            //【4】准备动态生成 java 代码：
            //【4.1】生成方法参数类型：
            // Map<String, Class<? extends IRouteGroup>>
            ParameterizedTypeName inputMapTypeOfRoot = ParameterizedTypeName.get(
                    ClassName.get(Map.class),
                    ClassName.get(String.class),
                    ParameterizedTypeName.get(
                            ClassName.get(Class.class),
                            WildcardTypeName.subtypeOf(ClassName.get(type_IRouteGroup))
                    )
            );

            //【4.2】生成方法参数类型：Map<String, RouteMeta>
            ParameterizedTypeName inputMapTypeOfGroup = ParameterizedTypeName.get(
                    ClassName.get(Map.class),
                    ClassName.get(String.class),
                    ClassName.get(RouteMeta.class)
            );

            //【4.4】生成方法参数：
            // Map<String, Class<? extends IRouteGroup>> routes
            // Map<String, RouteMeta> atlas
            // Map<String, RouteMeta> providers
            ParameterSpec rootParamSpec = ParameterSpec.builder(inputMapTypeOfRoot, "routes").build();
            ParameterSpec groupParamSpec = ParameterSpec.builder(inputMapTypeOfGroup, "atlas").build();
            ParameterSpec providerParamSpec = ParameterSpec.builder(inputMapTypeOfGroup, "providers").build(); 

            //【4.5】生成方法签名：
            // @Override
            // public void loadInto(Map<String, Class<? extends IRouteGroup>> routes) 
            MethodSpec.Builder loadIntoMethodOfRootBuilder = MethodSpec.methodBuilder(METHOD_LOAD_INTO)
                    .addAnnotation(Override.class)
                    .addModifiers(PUBLIC)
                    .addParameter(rootParamSpec);

            //【5】处理 @Route 注解修饰的元素
            for (Element element : routeElements) {
                TypeMirror tm = element.asType(); // 获得 Route 注解的元素的类型信息；
                Route route = element.getAnnotation(Route.class); // 获得 Route 注解对象；
                RouteMeta routeMeta; // 用于封装跳转信息；

                if (types.isSubtype(tm, type_Activity)) {
                    //【5.1】注解的元素是 activity 的子类；
                    logger.info(">>> Found activity route: " + tm.toString() + " <<<");

                    Map<String, Integer> paramsType = new HashMap<>(); // 保存 fieldName/Autowired.name --> 类型枚举序号；
                    Map<String, Autowired> injectConfig = new HashMap<>(); // 保存 fieldName/Autowired.name --> Autowired 实例
                  
                    for (Element field : element.getEnclosedElements()) {
                        //【5.1.1】返回该元素直接包含的子元素（成员属性），处理内部哪些被 @Autowired 注解的成员属性（避开 IProvider 子类）；
                        if (field.getKind().isField() && field.getAnnotation(Autowired.class) != null && !types.isSubtype(field.asType(), iProvider)) {
                            // 必须是被 @Autowired 注解的属性，但是不能是 IProvider
                            Autowired paramConfig = field.getAnnotation(Autowired.class);
                            //【5.1.1.1】根据是否设置 Autowired.name 对属性进行 byName 或者 byType 处理；
                            String injectName = StringUtils.isEmpty(paramConfig.name()) ? field.getSimpleName().toString() : paramConfig.name();
                            //【5.1.1.2】加入到集合；
                            paramsType.put(injectName, typeUtils.typeExchange(field));
                            injectConfig.put(injectName, paramConfig);
                        }
                    }
                    //【5.1.2】创建跳转对象；
                    routeMeta = new RouteMeta(route, element, RouteType.ACTIVITY, paramsType);
                    routeMeta.setInjectConfig(injectConfig);
                  
                } else if (types.isSubtype(tm, iProvider)) {
                    //【5.2】注解的元素实现了 IProvider 接口，创建跳转对象
                    logger.info(">>> Found provider route: " + tm.toString() + " <<<");
                    routeMeta = new RouteMeta(route, element, RouteType.PROVIDER, null);
                  
                } else if (types.isSubtype(tm, type_Service)) {
                    //【5.3】注解的元素是 service 的子类，创建跳转对象
                    logger.info(">>> Found service route: " + tm.toString() + " <<<");
                    routeMeta = new RouteMeta(route, element, RouteType.parse(SERVICE), null);
                  
                } else if (types.isSubtype(tm, fragmentTm) || types.isSubtype(tm, fragmentTmV4)) 
                    //【5.4】注解的元素是 fragment 的子类，，创建跳转对象
                    logger.info(">>> Found fragment route: " + tm.toString() + " <<<");
                    routeMeta = new RouteMeta(route, element, RouteType.parse(FRAGMENT), null);
                  
                } else {
                    throw new RuntimeException("ARouter::Compiler >>> Found unsupported class type, type = [" + types.toString() + "].");
                }
                
                //【*2.2.2.3.1.1】对跳转对象进行分类；
                categories(routeMeta);
            }

            //【4.6】生成方法签名：
            // @Override
            // public void loadInto(Map<String, RouteMeta> providers) 
            MethodSpec.Builder loadIntoMethodOfProviderBuilder = MethodSpec.methodBuilder(METHOD_LOAD_INTO)
                    .addAnnotation(Override.class)
                    .addModifiers(PUBLIC)
                    .addParameter(providerParamSpec);

            Map<String, List<RouteDoc>> docSource = new HashMap<>(); // key：组名；value：每个组内的路由跳转文档；

            //【5】按照分组的方式，遍历 RouteMeta；
            for (Map.Entry<String, Set<RouteMeta>> entry : groupMap.entrySet()) {
                String groupName = entry.getKey();

                //【5.1】生成方法签名：
                // @Override
                // public void loadInto(Map<String, RouteMeta> atlas) 
                MethodSpec.Builder loadIntoMethodOfGroupBuilder = MethodSpec.methodBuilder(METHOD_LOAD_INTO)
                        .addAnnotation(Override.class)
                        .addModifiers(PUBLIC)
                        .addParameter(groupParamSpec);

                List<RouteDoc> routeDocList = new ArrayList<>(); // 用于保存路由跳转信息；

                //【5.2】获得属于该 group 下的所有 RouteMeta，并依次处理；
                Set<RouteMeta> groupData = entry.getValue();
                for (RouteMeta routeMeta : groupData) {
                    //【*2.2.2.3.1.2】根据跳转信息，生成文档对象；
                    RouteDoc routeDoc = extractDocInfo(routeMeta);

                    ClassName className = ClassName.get((TypeElement) routeMeta.getRawType()); // 目标类的全限定名

                    switch (routeMeta.getType()) { //【5.2.1】针对跳转类型为 PROVIDER 的情况，这里会将其父类的信息缓存下来；
                        case PROVIDER:
                            // 返回直接由此类实现或直接由此接口扩展的接口类型（目标类的负类）
                            List<? extends TypeMirror> interfaces = ((TypeElement) routeMeta.getRawType()).getInterfaces();
                            for (TypeMirror tm : interfaces) {
                                routeDoc.addPrototype(tm.toString());

                                if (types.isSameType(tm, iProvider)) { // 如果是 .IProvider 类型，说明目标类是直接实现的 .IProvider 接口； 
                                    //【5.2.2】生成方法体：
                                    // providers.put("目标类的全限定名", RouteMeta.build(RouteType.PROVIDER, 目标类的类名.class, 
                                    //           ${routeMeta.getPath()}, ${routeMeta.getGroup()}, null, ${routeMeta.getPriority()}, ${routeMeta.getExtra()}));
                                    loadIntoMethodOfProviderBuilder.addStatement(
                                            "providers.put($S, $T.build($T." + routeMeta.getType() + ", $T.class, $S, $S, null, " + routeMeta.getPriority() 
                                                                             + ", " + routeMeta.getExtra() + "))",
                                            (routeMeta.getRawType()).toString(), // routeMeta.getRawType() 返回的是 element;
                                            routeMetaCn,  // RouteMeta
                                            routeTypeCn,  // RouteType
                                            className,    // 类名；
                                            routeMeta.getPath(),  // Route 的 path 属性；
                                            routeMeta.getGroup()); // group 属性；
                                  
                                } else if (types.isSubtype(tm, iProvider)) { // 如果是 .IProvider 的字类型，说明目标类是继承了一个实现 .IProvider 的类；
                                    //【5.2.3】生成方法体：
                                    // providers.put("直接父类的全限定名", RouteMeta.build(RouteType.PROVIDER, 目标类的类名.class, 
                                    //            ${routeMeta.getPath()}, ${routeMeta.getGroup()}, null, ${routeMeta.getPriority()}, ${routeMeta.getExtra()}));
                                    loadIntoMethodOfProviderBuilder.addStatement(
                                            "providers.put($S, $T.build($T." + routeMeta.getType() + ", $T.class, $S, $S, null, " + routeMeta.getPriority() 
                                      																			 + ", " + routeMeta.getExtra() + "))",
                                            tm.toString(),
                                            routeMetaCn,
                                            routeTypeCn,
                                            className,
                                            routeMeta.getPath(),
                                            routeMeta.getGroup());
                                }
                            }
                            break;
                        default:
                            break;
                    }

                    //【5.3】用于继续生成 route doc, 和 Autowired 注解的参数 hashmap
                    StringBuilder mapBodyBuilder = new StringBuilder();
                    Map<String, Integer> paramsType = routeMeta.getParamsType();
                    Map<String, Autowired> injectConfigs = routeMeta.getInjectConfig();
                    if (MapUtils.isNotEmpty(paramsType)) {
                        List<RouteDoc.Param> paramList = new ArrayList<>();

                        for (Map.Entry<String, Integer> types : paramsType.entrySet()) {
                            // 创建 Autowired 注解的参数 hashmap；
                            mapBodyBuilder.append("put(\"").append(types.getKey()).append("\", ").append(types.getValue()).append("); ");

                            RouteDoc.Param param = new RouteDoc.Param();
                            Autowired injectConfig = injectConfigs.get(types.getKey());
                            param.setKey(types.getKey());
                            param.setType(TypeKind.values()[types.getValue()].name().toLowerCase());
                            param.setDescription(injectConfig.desc());
                            param.setRequired(injectConfig.required());

                            paramList.add(param);
                        }
                        // 将 @AutoWeird 修饰的变量信息保存到 routeDoc 中；
                        routeDoc.setParams(paramList);
                    }
                    String mapBody = mapBodyBuilder.toString();

                    //【5.4】生成方法体：：
                    // atlas.put(${path}, RouteMeta.build(RouteType.XXXX, ${className}.class, 
                    //           ${path}, ${group}, new java.util.HashMap<String, Integer>(){{put(${fieldName}/${AutoWired.Name}, ${TypeKind});}}, ${priority}, ${extra}));
                    loadIntoMethodOfGroupBuilder.addStatement(
                            "atlas.put($S, $T.build($T." + routeMeta.getType() + ", $T.class, $S, $S, " 
                      						+ (StringUtils.isEmpty(mapBody) ? null : ("new java.util.HashMap<String, Integer>(){{" + mapBodyBuilder.toString()
                                  + "}}")) + ", " + routeMeta.getPriority() + ", " + routeMeta.getExtra() + "))",
                            routeMeta.getPath(),
                            routeMetaCn,
                            routeTypeCn,
                            className,
                            routeMeta.getPath().toLowerCase(),
                            routeMeta.getGroup().toLowerCase());

                    routeDoc.setClassName(className.toString()); // 将 className 保存到 routeDoc 中；
                    routeDocList.add(routeDoc); // 将这个路由表加入到 routeDocList 中；
                }

                //【5.5】动态生成 java 文件：
                String groupFileName = NAME_OF_GROUP + groupName;
                JavaFile.builder(PACKAGE_OF_GENERATE_FILE,   // 包名；com.alibaba.android.arouter.routes
                        TypeSpec.classBuilder(groupFileName) // 类名 ARouter$$Group$$ + ${groupName}
                                .addJavadoc(WARNING_TIPS)
                                .addSuperinterface(ClassName.get(type_IRouteGroup)) // 实现 .IRouteGroup 接口；
                                .addModifiers(PUBLIC)
                                .addMethod(loadIntoMethodOfGroupBuilder.build())
                                .build()
                ).build().writeTo(mFiler);

                logger.info(">>> Generated group: " + groupName + "<<<");
                //【5.6】将 key：groupName ---> value：ARouter$$Group$$ + ${groupName} 保存到 rootMap 表中；
                rootMap.put(groupName, groupFileName);
                docSource.put(groupName, routeDocList); // 将当前组的所有路由表保存到 docSource 中；
            }

            if (MapUtils.isNotEmpty(rootMap)) {
                //【6】生成方法体：：
               	// routes.put("app", ARouter$$Group$$${$groupName}.class);
                for (Map.Entry<String, String> entry : rootMap.entrySet()) {
                    loadIntoMethodOfRootBuilder.addStatement("routes.put($S, $T.class)", 
                                                             entry.getKey(), ClassName.get(PACKAGE_OF_GENERATE_FILE, entry.getValue())); // 当然，这里是全限定名；
                }
            }

            //【7】如果 gradle 设置了生成路由表，那就将 docSource 以 json 的形式输出；
            if (generateDoc) {
                docWriter.append(JSON.toJSONString(docSource, SerializerFeature.PrettyFormat));
                docWriter.flush();
                docWriter.close();
            }

            //【8】动态生成 java 文件：
            String providerMapFileName = NAME_OF_PROVIDER + SEPARATOR + moduleName;
            JavaFile.builder(PACKAGE_OF_GENERATE_FILE, // 包名；com.alibaba.android.arouter.routes
                    TypeSpec.classBuilder(providerMapFileName) // 类名：ARouter$$Providers$$ + ${moduleName}
                            .addJavadoc(WARNING_TIPS)
                            .addSuperinterface(ClassName.get(type_IProviderGroup)) // 实现 .IProviderGroup 接口；
                            .addModifiers(PUBLIC)
                            .addMethod(loadIntoMethodOfProviderBuilder.build())
                            .build()
            ).build().writeTo(mFiler);

            logger.info(">>> Generated provider map, name is " + providerMapFileName + " <<<");

            //【9】动态生成 java 文件：
            String rootFileName = NAME_OF_ROOT + SEPARATOR + moduleName;
            JavaFile.builder(PACKAGE_OF_GENERATE_FILE,  // 包名；com.alibaba.android.arouter.routes
                    TypeSpec.classBuilder(rootFileName) // 包名；ARouter$$Root$$ + ${moduleName}
                            .addJavadoc(WARNING_TIPS)
                            .addSuperinterface(ClassName.get(elementUtils.getTypeElement(ITROUTE_ROOT))) // 实现 .IRouteRoot 接口；
                            .addModifiers(PUBLIC)
                            .addMethod(loadIntoMethodOfRootBuilder.build())
                            .build()
            ).build().writeTo(mFiler);

            logger.info(">>> Generated root, name is " + rootFileName + " <<<");
        }
    }
```

整个流程还是很简单清晰的，主要是代码生成过程用很多的占位符，为我们看源码产生了很多的阻碍；

- RouteProcessor 不仅会解析 @Route，还会解析 @AutoWired；
- 最终会生成三个 java 文件，具体的模版信息：

###### 2.2.2.3.1.1 categories

对跳转信息进行分类；

```java
    private void categories(RouteMeta routeMete) {
        //【*2.2.2.3.1.2】校验路由跳转信息！
        if (routeVerify(routeMete)) {
            logger.info(">>> Start categories, group = " + routeMete.getGroup() + ", path = " + routeMete.getPath() + " <<<");
            //【1】将校验通过的跳转 RouteMeta 加入到 groupMap 中；
            Set<RouteMeta> routeMetas = groupMap.get(routeMete.getGroup());
            if (CollectionUtils.isEmpty(routeMetas)) {
                //【2】如果是第一次添加，需要创建一个 Set<RouteMeta>，内部元素以 path 排序；
                Set<RouteMeta> routeMetaSet = new TreeSet<>(new Comparator<RouteMeta>() {
                    @Override
                    public int compare(RouteMeta r1, RouteMeta r2) {
                        try {
                            return r1.getPath().compareTo(r2.getPath());
                        } catch (NullPointerException npe) {
                            logger.error(npe.getMessage());
                            return 0;
                        }
                    }
                });
                //【3】加入到集合中；
                routeMetaSet.add(routeMete);
                groupMap.put(routeMete.getGroup(), routeMetaSet);
            } else {
                //【4】已经创建了 set，直接加入；
                routeMetas.add(routeMete);
            }
        } else {
            logger.warning(">>> Route meta verify error, group is " + routeMete.getGroup() + " <<<");
        }
    }

```

end～

###### 2.2.2.3.1.2 routeVerify

校验路由跳转信息；

```java
    private boolean routeVerify(RouteMeta meta) {
        String path = meta.getPath();
        //【1】path 必须要指定，并且以 "/" 开头；
        if (StringUtils.isEmpty(path) || !path.startsWith("/")) {   // The path must be start with '/' and not empty!
            return false;
        }
        //【2】如果 Route 没有指定 group 属性，那么就以 path 的一级目录为
        if (StringUtils.isEmpty(meta.getGroup())) { // Use default group(the first word in path)
            try {
                String defaultGroup = path.substring(1, path.indexOf("/", 1));
                if (StringUtils.isEmpty(defaultGroup)) {
                    return false;
                }
                //【2.1】设置组 group
                meta.setGroup(defaultGroup);
                return true;
            } catch (Exception e) {
                logger.error("Failed to extract default group! " + e.getMessage());
                return false;
            }
        }

        return true;
    }

```

不多说了～

###### 2.2.2.3.1.2 extractDocInfo

创建路由信息对象：

```java
    private RouteDoc extractDocInfo(RouteMeta routeMeta) {
        //【1】根据 RouteMeta 创建 RouteDoc 实例；
        RouteDoc routeDoc = new RouteDoc();
        routeDoc.setGroup(routeMeta.getGroup());
        routeDoc.setPath(routeMeta.getPath());
        routeDoc.setDescription(routeMeta.getName());
        routeDoc.setType(routeMeta.getType().name().toLowerCase());
        routeDoc.setMark(routeMeta.getExtra());

        return routeDoc;
    }

```



#### 2.2.5.4 动态生成类

##### 2.2.5.4.1 模版信息

我们来看看生成了哪几种模板类：

- `ARouter$$Providers$$${moduleName}.java`

这个模版类继承了 IProviderGroup，其实都可以猜到，用于添加属于同一组的 iprovider：

```java
package com.alibaba.android.arouter.routes;

import com.alibaba.android.arouter.facade.enums.RouteType;
import com.alibaba.android.arouter.facade.model.RouteMeta;
import com.alibaba.android.arouter.facade.template.IProviderGroup;
import java.lang.Override;
import java.lang.String;
import java.util.Map;
... ... ...

public class ARouter$$Providers$$${moduleName} implements IProviderGroup {
  @Override
  public void loadInto(Map<String, RouteMeta> providers) {
      providers.put("目标类的全限定名", RouteMeta.build(RouteType.PROVIDER, 目标类的类名.class, 
                                     ${routeMeta.getPath()}, ${routeMeta.getGroup()}, null, ${routeMeta.getPriority()}, ${routeMeta.getExtra()}));
    
      providers.put("父类的全限定名", RouteMeta.build(RouteType.PROVIDER, 目标类的类名.class, 
                                                ${routeMeta.getPath()}, ${routeMeta.getGroup()}, null, ${routeMeta.getPriority()}, ${routeMeta.getExtra()}));
  }
}
```

不多说了～

- `ARouter$$Group$$${groupName}.java`

这个模版类继承了 IRouteGroup，其实都可以猜到，用于添加属于同一组的所有被 @Route 注解的元素：

```java
package com.alibaba.android.arouter.routes;

import com.alibaba.android.arouter.facade.enums.RouteType;
import com.alibaba.android.arouter.facade.model.RouteMeta;
import com.alibaba.android.arouter.facade.template.IRouteGroup;
import java.lang.Override;
import java.lang.String;
import java.util.Map;
... ... ...

public class ARouter$$Group$$${groupName} implements IRouteGroup {
  @Override
  public void loadInto(Map<String, RouteMeta> atlas) {
    atlas.put(${path}, RouteMeta.build(RouteType.XXXX, ${className}.class, ${path}, ${group},
                                       new java.util.HashMap<String, Integer>(){{put(${fieldName}/${AutoWired.Name}, ${TypeKind});}}, ${priority}, ${extra}));
    atlas.put(${path}, RouteMeta.build(RouteType.XXXX, ${className}.class, ${path}, ${group}, null, ${priority}, ${extra}));
  }
}
```

不多说了～

- `ARouter$$Root$$${moduleName}.java`

这个模版类继承了 IRouteRoot，最为 root，用于添加和管理 group 和对应的 `ARouter$$Group$$${moduleName}.java` 的映射关系；

```java
package com.alibaba.android.arouter.routes;

import com.alibaba.android.arouter.facade.template.IRouteGroup;
import com.alibaba.android.arouter.facade.template.IRouteRoot;
import java.lang.Class;
import java.lang.Override;
import java.lang.String;
import java.util.Map;

public class ARouter$$Root$$${moduleName} implements IRouteRoot {
  @Override
  public void loadInto(Map<String, Class<? extends IRouteGroup>> routes) {
    routes.put(${groupName}, ARouter$$Group$$${groupName}.class);
  }
}
```

我们可以通过  `ARouter$$Root$$${groupName}.java` 知道该 module 一共包含多少个 group。每个组中的的元素，可以通过    `ARouter$$Group$$${moduleName}.java`  这个文档添加；

##### 2.2.5.4.2 举个栗子

我写了个 Demo 可以让大家更直观的看到模版对应的实际代码：

###### 2.2.5.4.2.1 实例代码

下面是一个简单的 Demo：

- **MyActivity.java**

MyActivity 的组是：**coolqiActivity**

```java
package com.lishuaiqi.test;

import android.app.Activity;
import android.os.Bundle;
import android.support.annotation.Nullable;
import com.alibaba.android.arouter.facade.annotation.Autowired;
import com.alibaba.android.arouter.facade.annotation.Route;
import com.alibaba.android.arouter.launcher.ARouter;

/**
 * Created by lishuaiqi
 */
@Route(path = "/coolqiActivity/MyActivity")
public class MyActivity extends Activity {
    @Autowired(name = "isOneAuto")
    public boolean isOne;

    @Autowired(name = "isTwoAuto")
    public int isTwo;

    @Override
    protected void onCreate(@Nullable Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        ARouter.getInstance().inject(this);
    }
}
```

- **MyIProvider.java**

MyIProvider 的组是：**coolqiProvider**

```java
package com.lishuaiqi.test;

import android.content.Context;
import com.alibaba.android.arouter.facade.annotation.Route;
import com.alibaba.android.arouter.facade.template.IProvider;

/**
 * Created by lishuaiqi
 */
@Route(path = "/coolqiProvider/MyIProvider")
public class MyProvider implements IProvider {
    @Override
    public void init(Context context) {
    }
}
```

- **MySerializationService.java**

MySerializationService 的组是：**coolqiService**

```java
@Route(path = "/coolqiService/MySerializationService")
public class MySerializationService implements SerializationService {
    @Override
    public <T> T json2Object(String input, Class<T> clazz) {
        return null;
    }

    @Override
    public String object2Json(Object instance) {
        return null;
    }

    @Override
    public <T> T parseObject(String input, Type clazz) {
        return null;
    }

    @Override
    public void init(Context context) {

    }
}
```

我是新建了一个 Module，名字叫：**Coolqi**

###### 2.2.5.4.2.2 动态代码

动态的代码如下所示：

- `ARouter$$Group$$coolqiActivity.java`， `ARouter$$Group$$coolqiProvider.java`， `ARouter$$Group$$coolqiService.java`

```java
package com.alibaba.android.arouter.routes;

import com.alibaba.android.arouter.facade.enums.RouteType;
import com.alibaba.android.arouter.facade.model.RouteMeta;
import com.alibaba.android.arouter.facade.template.IRouteGroup;
import com.lishuaiqi.test.MyActivity;
import com.lishuaiqi.test.MyPathReplaceService;
import com.lishuaiqi.test.MyProvider;
import com.lishuaiqi.MainActivity;
import java.lang.Override;
import java.lang.String;
import java.util.Map;

public class ARouter$$Group$$coolqiActivity implements IRouteGroup {
  @Override
  public void loadInto(Map<String, RouteMeta> atlas) {
    atlas.put("/coolqiActivity/MyActivity", RouteMeta.build(RouteType.ACTIVITY, MyActivity.class, "/coolqiactivity/myactivity", "coolqiactivity", null, -1, -2147483648));
  }
}

public class ARouter$$Group$$coolqiProvider implements IRouteGroup {
  @Override
  public void loadInto(Map<String, RouteMeta> atlas) {
    atlas.put("/coolqiProvider/MyIProvider", RouteMeta.build(RouteType.PROVIDER, MyIProvider.class, "/coolqiprovider/myiprovider", "coolqiprovider", null, -1, -2147483648));
  }
}

public class ARouter$$Group$$coolqiService implements IRouteGroup {
  @Override
  public void loadInto(Map<String, RouteMeta> atlas) {
    atlas.put("/coolqiService/MySerializationService", RouteMeta.build(RouteType.PROVIDER, MySerializationService.class, "/coolqiservice/myserializationservice", "coolqiservice", null, -1, -2147483648));
  }
}
```

其他的就不说了，反正就是看代码！

- `ARouter$$Providers$$app.java`

```java
package com.alibaba.android.arouter.routes;

import com.alibaba.android.arouter.facade.enums.RouteType;
import com.alibaba.android.arouter.facade.model.RouteMeta;
import com.alibaba.android.arouter.facade.template.IProviderGroup;
import com.pa.sales2.test.MyIProvider;
import com.pa.sales2.test.MySerializationService;
import java.lang.Override;
import java.lang.String;
import java.util.Map;

public class ARouter$$Providers$$Coolqi implements IProviderGroup {
  @Override
  public void loadInto(Map<String, RouteMeta> providers) {
    providers.put("com.alibaba.android.arouter.facade.service.SerializationService", RouteMeta.build(RouteType.PROVIDER, MySerializationService.class, 
                            "/coolqiService/MySerializationService", "coolqiService", null, -1, -2147483648));
    providers.put("com.pa.sales2.test.MyIProvider", RouteMeta.build(RouteType.PROVIDER, MyIProvider.class, 
                            "/coolqiProvider/MyIProvider", "coolqiProvider", null, -1, -2147483648));
  }
}
```

其他的就不说了，反正就是看代码！

- `ARouter$$Root$$app.java`

```java
package com.alibaba.android.arouter.routes;

import com.alibaba.android.arouter.facade.template.IRouteGroup;
import com.alibaba.android.arouter.facade.template.IRouteRoot;
import java.lang.Class;
import java.lang.Override;
import java.lang.String;
import java.util.Map;

public class ARouter$$Root$$Coolqi implements IRouteRoot {
  @Override
  public void loadInto(Map<String, Class<? extends IRouteGroup>> routes) {
    routes.put("coolqiActivity", ARouter$$Group$$coolqiActivity.class);
    routes.put("coolqiProvider", ARouter$$Group$$coolqiProvider.class);
    routes.put("coolqiService", ARouter$$Group$$coolqiService.class);
  }
}
```

其他的就不说了，反正就是看代码！

### 2.2.3 AutowiredProcessor

核心解释器，用于处理 @Autowired 注解：

```java
@AutoService(Processor.class)
@SupportedAnnotationTypes({ANNOTATION_TYPE_AUTOWIRED})
public class AutowiredProcessor extends BaseProcessor {
  	... ... ...
}
```

我们从成员变量，初始化，注解处理三个方面来分析：

#### 2.2.3.1 Field

```java
//【1】key 表示属性所属的类，value 是一个 list 列表，保存这个类被 Autowired 修饰的所有元素；
private Map<TypeElement, List<Element>> parentAndChild = new HashMap<>(); 
private static final ClassName ARouterClass = ClassName.get("com.alibaba.android.arouter.launcher", "ARouter");
private static final ClassName AndroidLog = ClassName.get("android.util", "Log");
```



#### 2.2.3.2 Init

init 方法很简单，没有太多代码：

```java
@Override
public synchronized void init(ProcessingEnvironment processingEnvironment) {
    super.init(processingEnvironment);
    logger.info(">>> AutowiredProcessor init. <<<");
}

```



#### 2.2.3.3 Process  - 处理 Autowired 注解

```java
    @Override
    public boolean process(Set<? extends TypeElement> set, RoundEnvironment roundEnvironment) {
        if (CollectionUtils.isNotEmpty(set)) {
            try {
                logger.info(">>> Found autowired field, start... <<<");
                //【*2.2.3.3.1】对变量进行归类，并找到其所属的类；
                categories(roundEnvironment.getElementsAnnotatedWith(Autowired.class));
                //【*2.2.3.3.2】动态生成 java 类！
                generateHelper();

            } catch (Exception e) {
                logger.error(e);
            }
            return true;
        }

        return false;
    }

```

##### 2.2.3.3.1 categories

对变量进行归类，并找到其所属的类；

```java
    private void categories(Set<? extends Element> elements) throws IllegalAccessException {
        if (CollectionUtils.isNotEmpty(elements)) {
            //【1】遍历所有被 @AutoWired 注解的元素；
            for (Element element : elements) {
                //【2】返回封装此元素（非严格意义上）的最里层元素，实际上就是其所属的类；
                TypeElement enclosingElement = (TypeElement) element.getEnclosingElement();
                //【3】如果此成员属性是 private 的，那就抛出异常！
                if (element.getModifiers().contains(Modifier.PRIVATE)) {
                    throw new IllegalAccessException("The inject fields CAN NOT BE 'private'!!! please check field ["
                            + element.getSimpleName() + "] in class [" + enclosingElement.getQualifiedName() + "]");
                }
                //【4】将成员属性 element 和所属类元素 enclosingElement 保存到 parentAndChild 中，分类完毕；
                if (parentAndChild.containsKey(enclosingElement 保存到 )) {
                    parentAndChild.get(enclosingElement).add(element);
                } else {
                    List<Element> childs = new ArrayList<>();
                    childs.add(element);
                    parentAndChild.put(enclosingElement, childs);
                }
            }

            logger.info("categories finished.");
        }
    }

```

可以看到，private 的元素不能用 Autowired 修饰；

##### 2.2.3.3.2 generateHelper

动态生成 java 类：

```java
    private void generateHelper() throws IOException, IllegalAccessException {
        //【1】获得 .ISyringe/.SerializationService 接口在编译时期的状态信息；
        TypeElement type_ISyringe = elementUtils.getTypeElement(ISYRINGE);
        TypeElement type_JsonService = elementUtils.getTypeElement(JSON_SERVICE);
        //【2】返回类型信息：类/接口
        TypeMirror iProvider = elementUtils.getTypeElement(Consts.IPROVIDER).asType();
        TypeMirror activityTm = elementUtils.getTypeElement(Consts.ACTIVITY).asType();
        TypeMirror fragmentTm = elementUtils.getTypeElement(Consts.FRAGMENT).asType();
        TypeMirror fragmentTmV4 = elementUtils.getTypeElement(Consts.FRAGMENT_V4).asType();

        //【3】开始动态生成类：
        //【3.1】生成 inject 方法的参数：Object target
        ParameterSpec objectParamSpec = ParameterSpec.builder(TypeName.OBJECT, "target").build();

        if (MapUtils.isNotEmpty(parentAndChild)) {
            // 遍历 parentAndChild 集合；
            for (Map.Entry<TypeElement, List<Element>> entry : parentAndChild.entrySet()) {
                //【3.2】生成 inject 方法签名；
                // 		 @Override 
                // 		 public void inject(Object target)
                MethodSpec.Builder injectMethodBuilder = MethodSpec.methodBuilder(METHOD_INJECT)
                        .addAnnotation(Override.class)
                        .addModifiers(PUBLIC)
                        .addParameter(objectParamSpec);

                TypeElement parent = entry.getKey();
                List<Element> childs = entry.getValue();
                //【3.3】获得所属类的全限定名，包名；
                String qualifiedName = parent.getQualifiedName().toString();
                String packageName = qualifiedName.substring(0, qualifiedName.lastIndexOf("."));
                //【3.4】获得所属类的类名，拼接 "$$ARouter$$Root$$Autowired" 作为动态生成类的类名；
                String fileName = parent.getSimpleName() + NAME_OF_AUTOWIRED;

                logger.info(">>> Start process " + childs.size() + " field in " + parent.getSimpleName() + " ... <<<");
                
                //【3.5】创建生成 java 类 helper 对象：指定类名(fileName)，实现的接口(.ISyringe), 修饰符(public)
                TypeSpec.Builder helper = TypeSpec.classBuilder(fileName)
                        .addJavadoc(WARNING_TIPS)
                        .addSuperinterface(ClassName.get(type_ISyringe))
                        .addModifiers(PUBLIC);
                //【3.6】创建动态类的成员变量：
                //     private com.alibaba.android.arouter.facade.service.SerializationService serializationService;
                FieldSpec jsonServiceField = FieldSpec.builder(TypeName.get(type_JsonService.asType()), 
                                                               "serializationService", Modifier.PRIVATE).build();
                helper.addField(jsonServiceField);
                
                //【3.5】给 inject 增加方法体：
                //      serializationService = com.alibaba.android.arouter.launcher.ARouter.getInstance()
                //                .navigation(com.alibaba.android.arouter.facade.service.SerializationService.class);
                //      parentClass substitute = (parentClass) target
                injectMethodBuilder.addStatement("serializationService = $T.getInstance().navigation($T.class)", 
                                                 ARouterClass, ClassName.get(type_JsonService));
                injectMethodBuilder.addStatement("$T substitute = ($T)target", ClassName.get(parent), ClassName.get(parent));

                //【3.6】继续给 inject 增加方法体：（处理成员变量）
                for (Element element : childs) {
                    Autowired fieldConfig = element.getAnnotation(Autowired.class);
                    //【3.6.1】获取变量的名称；
                    String fieldName = element.getSimpleName().toString();
                    //【3.6.2】如果实现了 .IProvider 接口，针对于是否设置了 name 属性，进行 byType/ byName 分类处理；
                    if (types.isSubtype(element.asType(), iProvider)) {  // It's provider
                        if ("".equals(fieldConfig.name())) { 
                            //【3.6.2.1】如果 Autowired.name 为空，生成如下代码：
                            // substitute.变量名 = com.alibaba.android.arouter.launcher.ARouter.getInstance()
                            //                          .navigation(变量类型的全限定名.class);
                            injectMethodBuilder.addStatement(
                                    "substitute." + fieldName + " = $T.getInstance().navigation($T.class)",
                                    ARouterClass,
                                    ClassName.get(element.asType())
                            );
                        } else { 
                            //【3.6.2.2】如果 Autowired.name 不为空，生成如下代码：
                            // substitute.变量名 = (变量类型的全限定名) com.alibaba.android.arouter.launcher.ARouter.getInstance()
                            //                          .build(Autowired().name).navigation();
                            injectMethodBuilder.addStatement(
                                    "substitute." + fieldName + " = ($T)$T.getInstance().build($S).navigation()",
                                    ClassName.get(element.asType()),
                                    ARouterClass,
                                    fieldConfig.name()
                            );
                        }

                        //【3.6.2.3】判断 Autowired 的 required 是否为 true，如果为 true，那就要禁止 null 的情况！
                        // 其实就是判断 substitute.变量 如果 null，抛出异常；
                        if (fieldConfig.required()) {
                            injectMethodBuilder.beginControlFlow("if (substitute." + fieldName + " == null)");
                            injectMethodBuilder.addStatement(
                                    "throw new RuntimeException(\"The field '" + fieldName + "' is null, in class '\" 
                                                     + $T.class.getName() + \"!\")", ClassName.get(parent));
                            injectMethodBuilder.endControlFlow();
                        }
                    } else {
                        //【3.6.3】对于一般的可通过 intent 传递的变量，进入这里；
                        String originalValue = "substitute." + fieldName;
                        // 用于拼接成员变量的生成方式："substitute.变量名 = substitute."；
                        //【*2.2.3.3.2.1】对于实现了 serializable 接口的变量, 则是："substitute.变量名 = (变量类型的全限定名) substitute."
                        String statement = "substitute." + fieldName + " = " + buildCastCode(element) + "substitute.";
                        boolean isActivity = false;
                        if (types.isSubtype(parent.asType(), activityTm)) { 
                            //【3.6.4.1】如果是 activity，那么拼接代码：getIntent().
                            isActivity = true;
                            statement += "getIntent().";
                        } else if (types.isSubtype(parent.asType(), fragmentTm) || types.isSubtype(parent.asType(), fragmentTmV4)) { 
                            //【3.6.4.2】如果是 fragment，那么拼接代码：getArguments().
                            statement += "getArguments()."; 
                        } else {
                            throw new IllegalAccessException("The field [" + fieldName + "] need autowired from intent, " 
                                                             + "its parent must be activity or fragment!");
                        }
                        //【*2.2.3.3.2.2】处理 getIntent()/getArguments() 的数据；
                        // typeUtils.typeExchange(element) 返回的是成员属性的枚举序号！
                        statement = buildStatement(originalValue, statement, typeUtils.typeExchange(element), isActivity);
                        if (statement.startsWith("serializationService.")) { 
                            //【3.6.5.1】处理 serializationService（自定义对象）的情况：
                            injectMethodBuilder.beginControlFlow("if (null != serializationService)");
                            //【3.6.5.2】生成方法体："substitute.fieldName = " + statement;
                            // $S 被替换为变量名/Autowired.name，$T 被替换为变量类型的全限定名；
                            injectMethodBuilder.addStatement(
                                    "substitute." + fieldName + " = " + statement,
                                    (StringUtils.isEmpty(fieldConfig.name()) ? fieldName : fieldConfig.name()),
                                    ClassName.get(element.asType())
                            );
                            injectMethodBuilder.nextControlFlow("else");
                            injectMethodBuilder.addStatement(
                                    "$T.e(\"" + Consts.TAG + "\", \"You want automatic inject the field '" + fieldName 
                                              + "' in class '$T' , then you should implement 'SerializationService'"
                                              + " to support object auto inject!\")", AndroidLog, ClassName.get(parent));
                            injectMethodBuilder.endControlFlow();
                        } else {
                            //【3.6.5.3】处理其他的情况，如果 Autowired.name 不为 null，那么 $S 替换为变量名，否则为 Autowired.name
                            // 将方法体写入 inject；
                            injectMethodBuilder.addStatement(statement,
                                           StringUtils.isEmpty(fieldConfig.name()) ? fieldName : fieldConfig.name());
                        }

                        // Autowired 的 required 为 true，且不是 private 的，非空判断；
                        if (fieldConfig.required() && !element.asType().getKind().isPrimitive()) { 
                            injectMethodBuilder.beginControlFlow("if (null == substitute." + fieldName + ")");
                            injectMethodBuilder.addStatement(
                                    "$T.e(\"" + Consts.TAG + "\", \"The field '" + fieldName 
                                    + "' is null, in class '\" + $T.class.getName() + \"!\")", AndroidLog, ClassName.get(parent));
                            injectMethodBuilder.endControlFlow(); // 闭合方法体；
                        }
                    }
                }

                helper.addMethod(injectMethodBuilder.build()); // 将方法 builder 加入到类 builder 中；

                //【4】动态创建 java 文件；
                JavaFile.builder(packageName, helper.build()).build().writeTo(mFiler);

                logger.info(">>> " + parent.getSimpleName() + " has been processed, " + fileName + " has been generated. <<<");
            }

            logger.info(">>> Autowired processor stop. <<<");
        }
    }

```

整个流程我们分析完成了，我们先不关注动态生成的类的作用，在后面分析 arouter-api 模块的时候，就会知道这些类的作用是什么了。

###### 2.2.3.3.2.1 buildCastCode

判断 element 的类型是否是 SERIALIZABLE 的，这里利用到了前面的枚举类 TypeKind 和工具类 typeUtils：

```java
    private String buildCastCode(Element element) {
        //【1】判断 element 的类型是否是 SERIALIZABLE 的！
        if (typeUtils.typeExchange(element) == TypeKind.SERIALIZABLE.ordinal()) {
            //【2】创建代码块：(变量类型的全限定名)
            return CodeBlock.builder().add("($T) ", ClassName.get(element.asType())).build().toString();
        }
        return "";
    }

```

这个主要是针对于实现了 serializable 接口的变量，比如一些集合等等；

###### 2.2.3.3.2.2 buildStatement

处理 getIntent()/getArguments() 的数据：

- 参数 originalValue 表示变量: "substitute.fieldName“，用于返回默认值；
- 参数 type 是成员属性对应的枚举序号：

```java
    private String buildStatement(String originalValue, String statement, int type, boolean isActivity) {
        switch (TypeKind.values()[type]) {
            case BOOLEAN:
                //【1】如果是 boolean，那么 activty 拼接：getBooleanExtra($S, 变量)，fragment 拼接：getBoolean($S)
                statement += (isActivity ? ("getBooleanExtra($S, " + originalValue + ")") : ("getBoolean($S)"));
                break;
            case BYTE:
                //【2】如果是 byte，那么 activty 拼接：getByteExtra($S, 变量)，fragment 拼接：getByte($S)
                statement += (isActivity ? ("getByteExtra($S, " + originalValue + ")") : ("getByte($S)"));
                break;
            case SHORT:
                //【3】如果是 short，那么 activty 拼接：getShortExtra($S, 变量)，fragment 拼接：getShort($S)
                statement += (isActivity ? ("getShortExtra($S, " + originalValue + ")") : ("getShort($S)"));
                break;	
            case INT:
                //【4】如果是 int，那么 activty 拼接：getIntExtra($S, 变量)，fragment 拼接：getInt($S)
                statement += (isActivity ? ("getIntExtra($S, " + originalValue + ")") : ("getInt($S)"));
                break;
            case LONG:
                //【5】如果是 long，那么 activty 拼接：getLongExtra($S, 变量)，fragment 拼接：getLong($S)
                statement += (isActivity ? ("getLongExtra($S, " + originalValue + ")") : ("getLong($S)"));
                break;
            case CHAR:
                //【6】如果是 char，那么 activty 拼接：getCharExtra($S, 变量)，fragment 拼接：getChar($S)
                statement += (isActivity ? ("getCharExtra($S, " + originalValue + ")") : ("getChar($S)"));
                break;
            case FLOAT:
                //【7】如果是 float，那么 activty 拼接：getFloatExtra($S, 变量)，fragment 拼接：getFloat($S)
                statement += (isActivity ? ("getFloatExtra($S, " + originalValue + ")") : ("getFloat($S)"));
                break;
            case DOUBLE:
                //【8】如果是 double，那么 activty 拼接：getDoubleExtra($S, 变量)，fragment 拼接：getDouble($S)
                statement += (isActivity ? ("getDoubleExtra($S, " + originalValue + ")") : ("getDouble($S)"));
                break;
            case STRING:
                //【9】如果是 string，那么 activty 拼接：getExtras() == null ? 变量 : substitute.getIntent().getExtras().getString($S, 变量)
                // fragment 拼接：getString($S)
                statement += (isActivity ? ("getExtras() == null ? " + originalValue + " : substitute.getIntent().getExtras().getString($S, " + originalValue + ")") : ("getString($S)"));
                break;
            case SERIALIZABLE:
                //【10】如果是 serializable，那么 activty 拼接：getSerializableExtra($S)
                // fragment 拼接：getSerializable($S)
                statement += (isActivity ? ("getSerializableExtra($S)") : ("getSerializable($S)"));
                break;
            case PARCELABLE:
                //【11】如果是 parcelable，那么 activty 拼接：getParcelableExtra($S)
                // fragment 拼接：getParcelable($S)
                statement += (isActivity ? ("getParcelableExtra($S)") : ("getParcelable($S)"));
                break;
            case OBJECT:
                //【12】如果是 object，那么 activity 返回：serializationService.parseObject(substitute.getIntent().getStringExtra($S), new com.alibaba.android.arouter.facade.model.TypeWrapper<$T>(){}.getType())
                // fragment 返回：serializationService.parseObject(substitute.getArguments().getString($S), new com.alibaba.android.arouter.facade.model.TypeWrapper<$T>(){}.getType())
                statement = "serializationService.parseObject(substitute." + (isActivity ? "getIntent()." : "getArguments().") + (isActivity ? "getStringExtra($S)" : "getString($S)") + ", new " + TYPE_WRAPPER + "<$T>(){}.getType())";
                break;
        }

        return statement;
    }

```

可以看到，buildStatement 会处理 getIntent()/getArguments() 的数据，在 statement 基础上拼接/修改：

- **Activity** - 生成的 statement

| **变量类型**     | **动态生成的代码块**                                         |
| ---------------- | ------------------------------------------------------------ |
| **boolean**      | `substitute.变量 = substitute.getIntent().getBooleanExtra($S, 变量)` |
| **byte**         | `substitute.变量 = substitute.getIntent().getByteExtra($S, 变量)` |
| **short**        | `substitute.变量 = substitute.getIntent().getShortExtra($S, 变量)` |
| **int**          | `substitute.变量 = substitute.getIntent().getIntExtra($S, 变量)` |
| **long**         | `substitute.变量 = substitute.getIntent().getLongExtra($S, 变量)` |
| **char**         | `substitute.变量 = substitute.getIntent().getCharExtra($S, 变量)` |
| **float**        | `substitute.变量 = substitute.getIntent().getFloatExtra($S, 变量)` |
| **double**       | `substitute.变量 = substitute.getIntent().getDoubleExtra($S, 变量)` |
| **string**       | `substitute.变量 = substitute.getIntent().getExtras() == null ? 变量 : substitute.getIntent().getExtras().getString($S, 变量)` |
| **serializable** | `substitute.变量 = (变量类型的全限定名) substitute.getIntent().getSerializableExtra($S)` |
| **parcelable**   | `substitute.变量 = substitute.getIntent().getParcelableExtra($S)` |
| **object**       | `serializationService.parseObject(substitute.getIntent().getStringExtra($S), new com.alibaba.android.arouter.facade.model.TypeWrapper<$T>(){}.getType())` |

这里的 `$S, $T`，依然是作为占位符，并没有被替换成实体的类型；

- **Fragment** - 生成的 statement

| **变量类型**     | **动态生成的代码块**                                         |
| ---------------- | ------------------------------------------------------------ |
| **boolean**      | `substitute.变量 = substitute.getArguments().getBoolean($S)` |
| **byte**         | `substitute.变量 = substitute.getArguments().getByte($S)`    |
| **short**        | `substitute.变量 = substitute.getArguments().getShort($S)`   |
| **int**          | `substitute.变量 = substitute.getArguments().getInt($S)`     |
| **long**         | `substitute.变量 = substitute.getArguments().getLong($S)`    |
| **char**         | `substitute.变量 = substitute.getArguments().getChar($S)`    |
| **float**        | `substitute.变量 = substitute.getArguments().getFloat($S)`   |
| **double**       | `substitute.变量 = substitute.getArguments().getDouble($S)`  |
| **string**       | `substitute.变量 = substitute.getArguments().getString($S)`  |
| **serializable** | `substitute.变量 = (变量类型的全限定名) substitute.getArguments().getSerializable($S)` |
| **parcelable**   | `substitute.变量 = substitute.getArguments().getParcelable($S)` |
| **object**       | `serializationService.parseObject(substitute.getArguments().getString($S), new com.alibaba.android.arouter.facade.model.TypeWrapper<$T>(){}.getType())` |

这里的 `$S, $T`，依然是作为占位符，并没有被替换成实体的类型；

返回的 statement 会继续被处理！

#### 2.2.3.4 动态生成类

##### 2.2.3.4.1 模版信息

我们来看一下，解析 AutoWired 后动态生成的 java 类的模版：

```java
package 所属类所在的包;
import com.alibaba.android.arouter.facade.template.ISyringe;
import com.alibaba.android.arouter.facade.service.SerializationService;
import com.alibaba.android.arouter.launcher.ARouter;
import ... ... ...// 省略掉其他的导入信息（变量类型等等）

class 所属类的类名$$ARouter$$Root$$Autowired implements ISyringe {
    private SerializationService serializationService;
  
    @Override
    public void inject(Object target) {
           serializationService = ARouter.getInstance().navigation(SerializationService.class);
           所属类的全限定名 substitute = (所属类的全限定名) target;
           // 实现了 .IProvider 的成员变量；
           substitute.变量 = ARouter.getInstance().navigation(变量类型的全限定名.class);
           substitute.变量 = (变量类型) ARouter.getInstance().build(Autowired.name).navigation();
           // Autowired 的 required 为 true 才有； 
           if (substitute.变量 == null) {
               throw new RuntimeException(...);
           }
           
           // activity 的成员；
           substitute.变量 = substitute.getIntent().getBooleanExtra(变量名/Autowired.name, substitute.变量);
           substitute.变量 = substitute.getIntent().getByteExtra(变量名/Autowired.name, substitute.变量);
           substitute.变量 = substitute.getIntent().getShortExtra(变量名/Autowired.name, substitute.变量);
           substitute.变量 = substitute.getIntent().getIntExtra(变量名/Autowired.name, substitute.变量);
           substitute.变量 = substitute.getIntent().getCharExtra(变量名/Autowired.name, substitute.变量);
           substitute.变量 = substitute.getIntent().getFloatExtra(变量名/Autowired.name, substitute.变量);
           substitute.变量 = substitute.getIntent().getDoubleExtra(变量名/Autowired.name, substitute.变量);
           substitute.变量 = substitute.getIntent().getExtras() == null ? substitute.变量 : substitute.getIntent().getExtras().getString(变量名/Autowired.name, substitute.变量);
           substitute.变量 = (变量类型) substitute.getIntent().getSerializableExtra(变量名/Autowired.name);
           substitute.变量 = substitute.getIntent().getParcelableExtra(变量名/Autowired.name);
           substitute.变量 = serializationService.parseObject(substitute.getIntent().getStringExtra(变量名/Autowired.name), new com.alibaba.android.arouter.facade.model.TypeWrapper<变量类型>(){}.getType());   
      
           // fragment 的成员；
           substitute.变量 = substitute.getArguments().getBoolean(变量名/Autowired.name);
           substitute.变量 = substitute.getArguments().getByte(变量名/Autowired.name);
           substitute.变量 = substitute.getArguments().getShort(变量名/Autowired.name);
           substitute.变量 = substitute.getArguments().getInt(变量名/Autowired.name);
           substitute.变量 = substitute.getArguments().getLong(变量名/Autowired.name);
           substitute.变量 = substitute.getArguments().getChar(变量名/Autowired.name);
           substitute.变量 = substitute.getArguments().getFloat(变量名/Autowired.name);
           substitute.变量 = substitute.getArguments().getDouble(变量名/Autowired.name);
           substitute.变量 = substitute.getArguments().getString(变量名/Autowired.name);
           substitute.变量 = (变量类型) substitute.getArguments().getSerializable(变量名/Autowired.name);
           substitute.变量 = substitute.getArguments().getParcelable(变量名/Autowired.name);
           serializationService.parseObject(substitute.getArguments().getString(变量名/Autowired.name), new com.alibaba.android.arouter.facade.model.TypeWrapper<变量类型>(){}.getType());
    }
}

```

其实大家可以看的出来，inject 方法就是用来自动给成员变量赋值的；

##### 2.2.3.4.2 举个栗子

###### 2.2.3.4.2.1 实例代码

以 activity 为例子；

```java
package com.lishuaiqi.test;

import android.app.Activity;
import android.os.Bundle;
import android.support.annotation.Nullable;
import com.alibaba.android.arouter.facade.annotation.Autowired;
import com.alibaba.android.arouter.facade.annotation.Route;
import com.alibaba.android.arouter.launcher.ARouter;

/**
 * Created by lishuaiqi
 */
@Route(path = "/app/MyActivity")
public class MyActivity extends Activity {
    @Autowired(name = "isOneAuto")
    public boolean isOne;

    @Autowired(name = "isTwoAuto")
    public int isTwo;

    @Override
    protected void onCreate(@Nullable Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        ARouter.getInstance().inject(this);
    }
}

```

###### 2.2.3.4.2.2 动态代码

如下是动态代码了，不多说了：

```java
package com.lishuaiqi.test;

import com.alibaba.android.arouter.facade.service.SerializationService;
import com.alibaba.android.arouter.facade.template.ISyringe;
import com.alibaba.android.arouter.launcher.ARouter;
import java.lang.Object;
import java.lang.Override;

/**
 * DO NOT EDIT THIS FILE!!! IT WAS GENERATED BY AROUTER. */
public class MyActivity$$ARouter$$Autowired implements ISyringe {
  private SerializationService serializationService;

  @Override
  public void inject(Object target) {
    serializationService = ARouter.getInstance().navigation(SerializationService.class);
    MyActivity substitute = (MyActivity)target;
    substitute.isOne = substitute.getIntent().getBooleanExtra("isOneAuto", substitute.isOne);
    substitute.isTwo = substitute.getIntent().getIntExtra("isTwoAuto", substitute.isTwo);
  }
}

```

### 2.2.4 InterceptorProcessor

核心解释器，用于处理 @Interceptor 注解：

```java
@AutoService(Processor.class)
@SupportedAnnotationTypes(ANNOTATION_TYPE_INTECEPTOR)
public class InterceptorProcessor extends BaseProcessor {
  	... ... ...
}

```

我们从成员变量，初始化，注解处理三个方面来分析：

#### 2.2.4.1 Field

成员变量有两个：

```java
//【1】用于保存 key: [priority 优先级] 和 value：[@Interceptor 修饰的元素 Element] 的映射关系，作为 cache； 
private Map<Integer, Element> interceptors = new TreeMap<>();
//【2】用于保存 [com.alibaba.android.arouter.facade.template.IInterceptor] 的类型信息
private TypeMirror iInterceptor = null;

```

#### 2.2.4.2 Init

初始化操作：

```java
@Override
public synchronized void init(ProcessingEnvironment processingEnv) {
    super.init(processingEnv);
    //【1】Elements.getTypeElement 会返回 .IInterceptor 接口对应的 TypeElement
    // TypeElement.sType() 会返回 .IInterceptor 的类型信息：接口
    iInterceptor = elementUtils.getTypeElement(Consts.IINTERCEPTOR).asType();
    logger.info(">>> InterceptorProcessor init. <<<");
}

```

这里的 Consts.IINTERCEPTOR 是 IInterceptor 接口的全限定名：

> **com.alibaba.android.arouter.facade.template.IInterceptor**

#### 2.2.4.3 Process - 处理 Interceptor 注解

```java
    @Override
    public boolean process(Set<? extends TypeElement> annotations, RoundEnvironment roundEnv) {
        if (CollectionUtils.isNotEmpty(annotations)) {
            //【1】获得 @Interceptor 修饰的元素，这里会返回多个 Element 组成的 set！！
            Set<? extends Element> elements = roundEnv.getElementsAnnotatedWith(Interceptor.class);
            try {
                //【*2.2.4.3.1】解析元素：
                parseInterceptors(elements);
            } catch (Exception e) {
                logger.error(e);
            }
            return true;
        }

        return false;
    }

```

核心逻辑在 parseInterceptors 中；

##### 2.2.4.3.1 parseInterceptors 

我们来看下如何解析元素：

```java
    private void parseInterceptors(Set<? extends Element> elements) throws IOException {
        if (CollectionUtils.isNotEmpty(elements)) {
            logger.info(">>> Found interceptors, size is " + elements.size() + " <<<");

            //【1】执行校验，并将元素缓存下来；
            for (Element element : elements) {
                //【*2.2.4.3.1.1】执行校验；
                if (verify(element)) {
                    logger.info("A interceptor verify over, its " + element.asType());
                    //【1.1】获得 Interceptor 对象的优先级，判断是否已经添加到 interceptors 哈希表中，已经添加，抛出异常；
                    Interceptor interceptor = element.getAnnotation(Interceptor.class);
                    Element lastInterceptor = interceptors.get(interceptor.priority());
                    if (null != lastInterceptor) { // Added, throw exceptions
                        throw new IllegalArgumentException(
                                String.format(Locale.getDefault(), "More than one interceptors use same" +  
                                              "priority [%d], They are [%s] and [%s].",
                                        interceptor.priority(),
                                        lastInterceptor.getSimpleName(),
                                        element.getSimpleName())
                        );
                    }
                    //【1.2】将 priority --> element 关系缓存到 interceptors 中；
                    interceptors.put(interceptor.priority(), element);
                } else {
                    logger.error("A interceptor verify failed, its " + element.asType());
                }
            }

            //【2】返回 ".IInterceptor/.IInterceptorGroup" 接口对应的 TypeElement，保存了接口在编译时期的状态信息；
            TypeElement type_ITollgate = elementUtils.getTypeElement(IINTERCEPTOR);
            TypeElement type_ITollgateGroup = elementUtils.getTypeElement(IINTERCEPTOR_GROUP);

            //【3】生成 loadInto 方法的参数类型："Map<Integer, Class<? extends ITollgate>>""
            ParameterizedTypeName inputMapTypeOfTollgate = ParameterizedTypeName.get(
                    ClassName.get(Map.class),
                    ClassName.get(Integer.class),
                    ParameterizedTypeName.get(
                            ClassName.get(Class.class),
                            WildcardTypeName.subtypeOf(ClassName.get(type_ITollgate))
                    )
            );

            //【4】生成 loadInto 方法的方法参数：“Map<Integer, Class<? extends ITollgate>> interceptors”
            ParameterSpec tollgateParamSpec = ParameterSpec.builder(inputMapTypeOfTollgate, "interceptors").build();

            //【5】生成 loadInto 方法声明：
            // @Override
            // public void loadInto(Map<Integer, Class<? extends ITollgate>> interceptors){...}
            MethodSpec.Builder loadIntoMethodOfTollgateBuilder = MethodSpec.methodBuilder(METHOD_LOAD_INTO)
                    .addAnnotation(Override.class)
                    .addModifiers(PUBLIC)
                    .addParameter(tollgateParamSpec);

            //【6】生成 loadInto 方法体:
            // @Override
            // public void loadInto(Map<Integer, Class<? extends ITollgate>> interceptors){
            //      interceptors.put(priority, $T.class);
            // }
            if (null != interceptors && interceptors.size() > 0) {
                // for 循环中 interceptors 是 InterceptorProcessor 的成员变量哦！用来保存所有的 interceptor；
                // $T 最终会被替换为自定义的 interceptor 的类全限定名；
                for (Map.Entry<Integer, Element> entry : interceptors.entrySet()) {
                    loadIntoMethodOfTollgateBuilder.addStatement("interceptors.put(" + entry.getKey() + ", $T.class)",
                                                                 ClassName.get((TypeElement) entry.getValue()));
                }
            }

            //【7】生成最终的类文件，指定了包名，类名，修饰符，实现的接口等等；
            // 常量均定义在 Consts 中，具体的生成的类见下面……
            JavaFile.builder(PACKAGE_OF_GENERATE_FILE,
                    TypeSpec.classBuilder(NAME_OF_INTERCEPTOR + SEPARATOR + moduleName)
                            .addModifiers(PUBLIC)
                            .addJavadoc(WARNING_TIPS)
                            .addMethod(loadIntoMethodOfTollgateBuilder.build())
                            .addSuperinterface(ClassName.get(type_ITollgateGroup))
                            .build()
            ).build().writeTo(mFiler);

            logger.info(">>> Interceptor group write over. <<<");
        }
    }

```

可以看到其是使用 javapoet 三方库来懂爱生成 .java 文件；

###### 2.2.4.3.1.1 verify 

校验元素和注解的正确性：

```java
    private boolean verify(Element element) {
        //【1】获得注解对象；
        Interceptor interceptor = element.getAnnotation(Interceptor.class);
        //【2】元素 Element 必须被 Interceptor 注解修饰，
        // 并且其实现了 com.alibaba.android.arouter.facade.template.IInterceptor 接口；
        return null != interceptor && ((TypeElement) element).getInterfaces().contains(iInterceptor);
    }

```

end～

#### 2.2.4.4 **动态生成类** 

##### 2.2.4.4.1 模版信息

最终生成的 java 文件名为：

```java
ARouter$$Interceptors$${moduleName}.java

```

最终生成的模版类信息为：

```java
package com.alibaba.android.arouter.routes;

import com.alibaba.android.arouter.facade.template.IInterceptor;
import com.alibaba.android.arouter.facade.template.IInterceptorGroup;
import java.lang.Class;
import java.lang.Integer;
import java.lang.Override;
import java.util.Map;
... ... ...

public class ARouter$$Interceptors$$${moduleName} implements IInterceptorGroup {
   @Override
   public void loadInto(Map<Integer, Class<? extends ITollgate>> interceptors){
     		interceptors.put(${priority}, ${InterceptorName}.class);
   }
}

```

可以看到，对于 Interceptor，ARouter 也是采取分组管理的方式：

- 以 module 为组，组名为 `ARouter$$Interceptors$${moduleName}`；

##### 2.2.4.4.2 举个栗子

###### 2.2.4.4.2.1 实例代码

我们自定义了一个拦截器：

```java
package com.lishuaiqi.test;

import android.content.Context;
import com.alibaba.android.arouter.facade.Postcard;
import com.alibaba.android.arouter.facade.annotation.Interceptor;
import com.alibaba.android.arouter.facade.callback.InterceptorCallback;
import com.alibaba.android.arouter.facade.template.IInterceptor;

/**
 * Created by lishuaiqi
 */
@Interceptor(priority = 8, name = "测试用拦截器")
public class TestInterceptor implements IInterceptor {
    @Override
    public void process(Postcard postcard, InterceptorCallback callback) {

    }

    @Override
    public void init(Context context) {

    }
}

```

###### 2.2.4.4.2.2 动态代码

看看最终的代码：

```java
package com.alibaba.android.arouter.routes;

import com.alibaba.android.arouter.facade.template.IInterceptor;
import com.alibaba.android.arouter.facade.template.IInterceptorGroup;
import com.lishuaiqi.test.TestInterceptor;
import java.lang.Class;
import java.lang.Integer;
import java.lang.Override;
import java.util.Map;

/**
 * DO NOT EDIT THIS FILE!!! IT WAS GENERATED BY AROUTER. */
public class ARouter$$Interceptors$$Coolqi implements IInterceptorGroup {
  @Override
  public void loadInto(Map<Integer, Class<? extends IInterceptor>> interceptors) {
    interceptors.put(8, TestInterceptor.class);
  }
}

```



## 2.3 utils

该 package 下包含了一些工具类：

### 2.3.1 Logger

用于打印 log 信息，调试使用；

```java
public class Logger {
    private Messager msg;

    public Logger(Messager messager) {
        msg = messager;
    }

    public void info(CharSequence info) {
        if (StringUtils.isNotEmpty(info)) {
            msg.printMessage(Diagnostic.Kind.NOTE, Consts.PREFIX_OF_LOGGER + info);
        }
    }

    public void error(CharSequence error) {
        if (StringUtils.isNotEmpty(error)) {
            msg.printMessage(Diagnostic.Kind.ERROR, Consts.PREFIX_OF_LOGGER + 
                             "An exception is encountered, [" + error + "]");
        }
    }

    public void error(Throwable error) {
        if (null != error) {
            msg.printMessage(Diagnostic.Kind.ERROR, Consts.PREFIX_OF_LOGGER + 
                             "An exception is encountered, [" + error.getMessage() + "]" + 
                             "\n" + formatStackTrace(error.getStackTrace()));
        }
    }

    public void warning(CharSequence warning) {
        if (StringUtils.isNotEmpty(warning)) {
            msg.printMessage(Diagnostic.Kind.WARNING, Consts.PREFIX_OF_LOGGER + warning);
        }
    }

    private String formatStackTrace(StackTraceElement[] stackTrace) {
        StringBuilder sb = new StringBuilder();
        for (StackTraceElement element : stackTrace) {
            sb.append("    at ").append(element.toString());
            sb.append("\n");
        }
        return sb.toString();
    }

```

方法都比较简单，就不多说了。



### 2.3.2 TypeUtils

该类是一个类型工具类，主要用于获取元素的类型，并对类型做一个转换；

```java
public class TypeUtils {

    private Types types;
    private TypeMirror parcelableType;
    private TypeMirror serializableType;

    public TypeUtils(Types types, Elements elements) {
        this.types = types;
        parcelableType = elements.getTypeElement(PARCELABLE).asType();
        serializableType = elements.getTypeElement(SERIALIZABLE).asType();
    }

    //【1】可以看到，这个方法用于返回枚举常量的序数。这里的枚举常量是前面分析的 TypeKind.XXX
    public int typeExchange(Element element) {
        TypeMirror typeMirror = element.asType();

        //【1】对于 private，类型这里直接处理；
        if (typeMirror.getKind().isPrimitive()) {
            return element.asType().getKind().ordinal();
        }
        //【2】对于 no private 的类型，返回变量的类型，通过 TypeKind 找到类型对应的序数（0，1，2...）
        switch (typeMirror.toString()) {
            case BYTE:
                return TypeKind.BYTE.ordinal();
            case SHORT:
                return TypeKind.SHORT.ordinal();
            case INTEGER:
                return TypeKind.INT.ordinal();
            case LONG:
                return TypeKind.LONG.ordinal();
            case FLOAT:
                return TypeKind.FLOAT.ordinal();
            case DOUBEL:
                return TypeKind.DOUBLE.ordinal();
            case BOOLEAN:
                return TypeKind.BOOLEAN.ordinal();
            case CHAR:
                return TypeKind.CHAR.ordinal();
            case STRING:
                return TypeKind.STRING.ordinal();
            default:
                //【3】处理 PARCELABLE，SERIALIZABLE 和 OBJECT 的情况；
                if (types.isSubtype(typeMirror, parcelableType)) {
                    // PARCELABLE
                    return TypeKind.PARCELABLE.ordinal();
                } else if (types.isSubtype(typeMirror, serializableType)) {
                    // SERIALIZABLE
                    return TypeKind.SERIALIZABLE.ordinal();
                } else {
                    return TypeKind.OBJECT.ordinal();
                }
        }
    }

```

TypeKind 前面有分析过，其是一个枚举类！

### 2.3.3 Consts

用于保存一些核心的常量，下面来看看核心的常量。

#### 2.3.3.1 Log 打印相关

这些是和 log 打印相关的，比较简单：

```java
public static final String PROJECT = "ARouter"; // 这个常量其他常量也会用到；
public static final String TAG = PROJECT + "::";
static final String PREFIX_OF_LOGGER = PROJECT + "::Compiler ";
public static final String NO_MODULE_NAME_TIPS = "These no module name, at 'build.gradle', like :\n" +
  "android {\n" +
  "    defaultConfig {\n" +
  "        ...\n" +
  "        javaCompileOptions {\n" +
  "            annotationProcessorOptions {\n" +
  "                arguments = [AROUTER_MODULE_NAME: project.getName()]\n" +
  "            }\n" +
  "        }\n" +
  "    }\n" +
  "}\n";

```

不多说！



#### 2.3.3.2 Gradle 配置相关

这些是和 gradle 配置相关的机制：

```java
public static final String KEY_MODULE_NAME = "AROUTER_MODULE_NAME";
public static final String KEY_GENERATE_DOC_NAME = "AROUTER_GENERATE_DOC";
public static final String VALUE_ENABLE = "enable";

```

这个前面有说过，通过 gradle 配置；



#### 2.3.3.3 系统核心类

这些是和 Android 系统的一些核心类有关，也是 ARouter 能够注解处理的类：

```java
public static final String ACTIVITY = "android.app.Activity";
public static final String FRAGMENT = "android.app.Fragment";
public static final String FRAGMENT_V4 = "android.support.v4.app.Fragment";
public static final String SERVICE = "android.app.Service";
public static final String PARCELABLE = "android.os.Parcelable";

```

可以看到，都是系统类的全限定名；

#### 2.3.3.4 注解类型

这些是和 ARouter 的注解相关的常量：

```java
private static final String FACADE_PACKAGE = "com.alibaba.android.arouter.facade";  // 这个常量其他常量也会用到；
public static final String ANNOTATION_TYPE_INTECEPTOR = FACADE_PACKAGE + ".annotation.Interceptor";
public static final String ANNOTATION_TYPE_ROUTE = FACADE_PACKAGE + ".annotation.Route";
public static final String ANNOTATION_TYPE_AUTOWIRED = FACADE_PACKAGE + ".annotation.Autowired";

```

可以看到，都是注解的全限定名；

#### 2.3.3.5 核心接口和类

这些是和 ARouter 提供的一些核心接口：

```java
//【1】用于指定不同的 package 目录，属于 arouter-api 模块；
private static final String FACADE_PACKAGE = "com.alibaba.android.arouter.facade";
private static final String TEMPLATE_PACKAGE = ".template";
private static final String SERVICE_PACKAGE = ".service";
private static final String MODEL_PACKAGE = ".model";
//【2】下面是 arouter-api 模块的 template 包下的接口的全限定名；
public static final String IPROVIDER = FACADE_PACKAGE + TEMPLATE_PACKAGE + ".IProvider";
public static final String IPROVIDER_GROUP = FACADE_PACKAGE + TEMPLATE_PACKAGE + ".IProviderGroup";
public static final String IINTERCEPTOR = FACADE_PACKAGE + TEMPLATE_PACKAGE + ".IInterceptor";
public static final String IINTERCEPTOR_GROUP = FACADE_PACKAGE + TEMPLATE_PACKAGE + ".IInterceptorGroup";
public static final String ITROUTE_ROOT = FACADE_PACKAGE + TEMPLATE_PACKAGE + ".IRouteRoot";
public static final String IROUTE_GROUP = FACADE_PACKAGE + TEMPLATE_PACKAGE + ".IRouteGroup";
public static final String ISYRINGE = FACADE_PACKAGE + TEMPLATE_PACKAGE + ".ISyringe";
//【3】下面是 arouter-api 模块的 service 包下的服务的全限定名；
public static final String JSON_SERVICE = FACADE_PACKAGE + SERVICE_PACKAGE + ".SerializationService";
//【4】下面是 arouter-annotation 模块的 model 包下的服务的全限定名；
public static final String TYPE_WRAPPER = FACADE_PACKAGE + MODEL_PACKAGE + ".TypeWrapper";

```

同样的，也是一些全限定名；

ARouter 的拦截器需要实现 IInterceptor 接口，服务需要实现 IProvider 接口；

同时，由于 ARouter 是分组管理的，所以拦截器和服务又会属于不同的组：拦截器组需要实现 IInterceptorGroup 接口，服务组需要实现 IProviderGroup 组；

对于跳转来说，也会有分组，跳转组需要实现 IRouteGroup，而所有的跳转组属于一个 root：IRouteRoot

#### 2.3.3.6 动态生成类

下面 这些是和动态生成的类相关的：

- 动态生成类的类名；

```java
public static final String SEPARATOR = "$$";
public static final String PROJECT = "ARouter";
public static final String METHOD_LOAD_INTO = "loadInto";
public static final String METHOD_INJECT = "inject";
public static final String NAME_OF_ROOT = PROJECT + SEPARATOR + "Root"; // ARouter$$Root
public static final String NAME_OF_PROVIDER = PROJECT + SEPARATOR + "Providers"; // ARouter$$Providers
public static final String NAME_OF_GROUP = PROJECT + SEPARATOR + "Group" + SEPARATOR; // ARouter$$Group$$
public static final String NAME_OF_INTERCEPTOR = PROJECT + SEPARATOR + "Interceptors"; // ARouter$$Interceptors
public static final String NAME_OF_AUTOWIRED = SEPARATOR + PROJECT + SEPARATOR + "Autowired"; //$$ARouter$$Root$$Autowired

```

动态生成类的类名是通过 "$$" 将关键字拼接起来！

- 动态生成类的所属包名；

```java
public static final String PACKAGE_OF_GENERATE_FILE = "com.alibaba.android.arouter.routes";
public static final String PACKAGE_OF_GENERATE_DOCS = "com.alibaba.android.arouter.docs";

```

ARouter 会通过 javapoet 来动态生成对应的类，我们在分析 processor 的过程中就会看到。

# 3 总结

本篇文章分析了 arouter-compiler 模块的架构，arouter 内置的三种注解处理器，以及 arouter 注解的处理，动态类的生成。

好累～

后续上流程图吧～～对于个人收获也是很大的～～至少会自定义注解～～至少会动态生成代码了～～

