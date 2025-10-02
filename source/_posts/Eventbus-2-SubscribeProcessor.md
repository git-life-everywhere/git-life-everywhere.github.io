# EventBus 第二篇 - Subscribe 注解处理
title: EventBus 第二篇 - Subscribe 注解处理
date: 2019/08/27 20:46:25 
catalog: true
categories: 

- 开源库源码分析 
- EventBus
tags: EventBus
copyright: true

---

本系列文章主要分析 EventBus 框架的架构和原理，，基于最新的 **3.1.0** 版本。

> 这是 EventBus 开源库的地址，大家可以直接访问
> https://github.com/greenrobot/EventBus

本篇文章是 EventBus 的第二篇，主要分析 Subscribe 注解的处理；

Eventbus 翻译过来就是事件总线，用于简化组件和组件，线程和线程之间的消息通信，可以捆成是 Handler + Thread 的替代品。

# 1 回顾

我们在使用的过程中，需要设置接收消息的方法：

```java
@Subscribe(threadMode = ThreadMode.MAIN)
public void onEventMainThread(Object object) {
   ... ... ...
}
```

注解  Subscribe 可以说是 EventBus 的核心了，我们知道，3.x 版本之前，EventBus 使用的是运行时注解，其实就是 Java 的反射机制，但是这带来了性能的损耗！

因此，从 3.x 开始，Eventbus 引入了编译时注解处理的特性，核心类就是 EventBusAnnotationProcessor！ 

我们来看看注解的定义：

```java
@Documented
@Retention(RetentionPolicy.RUNTIME)
@Target({ElementType.METHOD})
public @interface Subscribe {
    ThreadMode threadMode() default ThreadMode.POSTING; // 线程模型：默认 POSTING

    boolean sticky() default false; // 默认非粘性；

    int priority() default 0; // 优先级为 0；
}
```

可以看到，Subscribe 用于修饰方法，并且可以保留到运行时，这是因为默认情况下，EventBus 是通过运行时注解，反射加载方法的，除非开启编译时注解处理机制；

# 2 EventBusAnnotationProcessor - Subscribe 处理

浏览 EventBus 的源码目录，我们能看到处理 Subscribe 注解的是 EventBusAnnotationProcessor 类，依然是编译时注解，动态生成代码：

```java
@SupportedAnnotationTypes("org.greenrobot.eventbus.Subscribe")
@SupportedOptions(value = {"eventBusIndex", "verbose"})
public class EventBusAnnotationProcessor extends AbstractProcessor {
```

可以看到，它支持的只有一个注解：Subscribe

同时他支持两个配置属性：

- eventBusIndex：是否开启编译时注解处理，这个特性是 3.x 版本新增的，也就是将运行时的处理放到了编译时注解处理，动态生成 java 代码，用于提升框架的性能；
- verbose：用于控制 log，调试使用；

这两个属性是在 gradle 中配置的；

```groovy
android {
    defaultConfig {
        javaCompileOptions {
            annotationProcessorOptions {
                arguments = [eventBusIndex:'com.monster.android.wild.MyEventBusIndex', verbose : "true"]
            }
        }
    }
}

dependencies {
    api 'org.greenrobot:eventbus:3.1.0'
    annotationProcessor 'org.greenrobot:eventbus-annotation-processor:3.0.1'
} 
```

我们在下面的分析中就能看到，注解处理器对于这几个参数的处理：



## 2.1 成员变量

EventBusAnnotationProcessor 内部如下的变量和常量：

```java
public static final String OPTION_EVENT_BUS_INDEX = "eventBusIndex";
public static final String OPTION_VERBOSE = "verbose";
```

上面的变量用于获取 gradle 的环境变量；

```java
// 保存订阅者（类）和其订阅方法（注解修饰的方法）的映射关系；
private final ListMap<TypeElement, ExecutableElement> methodsByClass = new ListMap<>(); 
// 保存需要跳过的订阅者
private final Set<TypeElement> classesToSkip = new HashSet<>();

private boolean writerRoundDone;
private int round;
private boolean verbose;
```



## 2.2 process

依然是核心方法：

```java
@Override
public boolean process(Set<? extends TypeElement> annotations, RoundEnvironment env) {
  Messager messager = processingEnv.getMessager();
  try {
    //【1】获取 eventBusIndex 编译属性，并判断是否有设置这个，没有的话就不处理；
    String index = processingEnv.getOptions().get(OPTION_EVENT_BUS_INDEX);
    if (index == null) {
      messager.printMessage(Diagnostic.Kind.ERROR, "No option " + OPTION_EVENT_BUS_INDEX +
                            " passed to annotation processor");
      return false;
    }
    //【2】判断是否要输出 log；
    verbose = Boolean.parseBoolean(processingEnv.getOptions().get(OPTION_VERBOSE));
    //【3】获取动态创建的 java 类的类名；
    int lastPeriod = index.lastIndexOf('.');
    String indexPackage = lastPeriod != -1 ? index.substring(0, lastPeriod) : null;

    round++;
    if (verbose) {
      messager.printMessage(Diagnostic.Kind.NOTE, "Processing round " + round + ", new annotations: " +
                            !annotations.isEmpty() + ", processingOver: " + env.processingOver());
    }
    if (env.processingOver()) {
      if (!annotations.isEmpty()) {
        messager.printMessage(Diagnostic.Kind.ERROR,
                              "Unexpected processing state: annotations still available after processing over");
        return false;
      }
    }
    if (annotations.isEmpty()) {
      return false;
    }

    if (writerRoundDone) {
      messager.printMessage(Diagnostic.Kind.ERROR,
                            "Unexpected processing state: annotations still available after writing.");
    }
    //【-->2.2.1】收集注解 Subscribe 修饰的元素；
    collectSubscribers(annotations, env, messager);
    //【-->2.2.2】检查某些注解是否要忽略；
    checkForSubscribersToSkip(messager, indexPackage);

    if (!methodsByClass.isEmpty()) {
      //【-->2.2.3】如果收集到了被注解修饰的方法，那么就动态创建 java 类；
      createInfoIndexFile(index);
    } else {
      messager.printMessage(Diagnostic.Kind.WARNING, "No @Subscribe annotations found");
    }
    writerRoundDone = true;
  } catch (RuntimeException e) {
    e.printStackTrace();
    messager.printMessage(Diagnostic.Kind.ERROR, "Unexpected error in EventBusAnnotationProcessor: " + e);
  }
  return true;
}
```

整个流程很简单，不多 say；

### 2.2.1 collectSubscribers - 收集

收集注解 Subscribe 修饰的元素；

```java
    private void collectSubscribers(Set<? extends TypeElement> annotations, RoundEnvironment env, Messager messager) {
        for (TypeElement annotation : annotations) {
            Set<? extends Element> elements = env.getElementsAnnotatedWith(annotation); // 获得 Subscribe 修饰的所有元素；
            for (Element element : elements) {
                if (element instanceof ExecutableElement) {
                    ExecutableElement method = (ExecutableElement) element;
                    //【-->2.2.1.1】检查注解修饰的方法是否满足条件；
                    if (checkHasNoErrors(method, messager)) {
                        //【1】获得方法所属的类元素，加入到 methodsByClass 哈希表中；
                        TypeElement classElement = (TypeElement) method.getEnclosingElement();
                        methodsByClass.putElement(classElement, method);
                    }
                } else {
                    messager.printMessage(Diagnostic.Kind.ERROR, "@Subscribe is only valid for methods", element);
                }
            }
        }
    }
```

#### 2.2.1.1 checkHasNoErrors

检查是否有错误：

```java
    private boolean checkHasNoErrors(ExecutableElement element, Messager messager) {
        //【1】方法不能是 static 的；
        if (element.getModifiers().contains(Modifier.STATIC)) {
            messager.printMessage(Diagnostic.Kind.ERROR, "Subscriber method must not be static", element);
            return false;
        }
        //【2】方法必须是 public 的；
        if (!element.getModifiers().contains(Modifier.PUBLIC)) {
            messager.printMessage(Diagnostic.Kind.ERROR, "Subscriber method must be public", element);
            return false;
        }
        //【3】方法必须至少有一个 param；
        List<? extends VariableElement> parameters = ((ExecutableElement) element).getParameters();
        if (parameters.size() != 1) {
            messager.printMessage(Diagnostic.Kind.ERROR, "Subscriber method must have exactly 1 parameter", element);
            return false;
        }
        return true;
    }
```

不多说；



### 2.2.2 checkForSubscribersToSkip

跳过一些订阅者：

```java
    private void checkForSubscribersToSkip(Messager messager, String myPackage) {
        //【1】遍历收集到所有的订阅者；
        for (TypeElement skipCandidate : methodsByClass.keySet()) {
            TypeElement subscriberClass = skipCandidate;
            while (subscriberClass != null) {
                //【-->2.2.2.1】判断下注解方法所属的类是否满足条件，如果不满足，加入到 classesToSkip 跳过；
                if (!isVisible(myPackage, subscriberClass)) {
                    //【2】将要跳过的 class 加入到 classesToSkip 集合中；
                    boolean added = classesToSkip.add(skipCandidate);
                    if (added) {
                        String msg;
                        if (subscriberClass.equals(skipCandidate)) { // 是当前类，还是父类呢？
                            msg = "Falling back to reflection because class is not public";
                        } else {
                            msg = "Falling back to reflection because " + skipCandidate +
                                    " has a non-public super class";
                        }
                        messager.printMessage(Diagnostic.Kind.NOTE, msg, subscriberClass);
                    }
                    break;
                }
                //【3】处理类（包括父类）所有的注解方法：
                List<ExecutableElement> methods = methodsByClass.get(subscriberClass);
                if (methods != null) {
                    //【3.1】遍历该类的注解方法，处理方法的参数；
                    for (ExecutableElement method : methods) {
                        String skipReason = null;
                        //【3.2】获取注解方法的第一个参数（意味着第一个参数必须是消息对象）
                        VariableElement param = method.getParameters().get(0);
                        //【-->2.2.2.2】获取被注解方法的参数类型；
                        TypeMirror typeMirror = getParamTypeMirror(param, messager);
                        //【3.3】如果参数的类型不是类/接口，那么就跳过该类；
                        if (!(typeMirror instanceof DeclaredType) ||
                                !(((DeclaredType) typeMirror).asElement() instanceof TypeElement)) {
                            skipReason = "event type cannot be processed";
                        }
                        //【3.4】如果上面满足条件，那就判断下参数是否可以访问；
                        if (skipReason == null) {
                            TypeElement eventTypeElement = (TypeElement) ((DeclaredType) typeMirror).asElement();
                            //【-->2.2.2.1】判断下注解方法所属的类是否满足条件，如果不满足，加入到 classesToSkip 跳过；
                            if (!isVisible(myPackage, eventTypeElement)) {
                                skipReason = "event type is not public";
                            }
                        }
                        if (skipReason != null) {
                             //【3.2】将要跳过的 class 加入到 classesToSkip 集合中；
                            boolean added = classesToSkip.add(skipCandidate);
                            if (added) {
                                String msg = "Falling back to reflection because " + skipReason;
                                if (!subscriberClass.equals(skipCandidate)) {
                                    msg += " (found in super class for " + skipCandidate + ")";
                                }
                                messager.printMessage(Diagnostic.Kind.NOTE, msg, param);
                            }
                            break;
                        }
                    }
                }
                //【-->2.2.2.3】获取其父类对应的元素，然后 while 循环继续处理其 super class；
                subscriberClass = getSuperclass(subscriberClass); 
            }
        }
    }
```

可以看到，默认我们获得的是注解方法所在的当前类，但是 while 循环还会继续处理其父类；

- 先处理子类，再处理父类；
- 被注解的方法的第一个参数必须是要处理的消息；
- 消息类型必须是类/接口的实现；



#### 2.2.2.1 isVisible

用来判断注解方法所属的类是否满足条件：

```java
    private boolean isVisible(String myPackage, TypeElement typeElement) {
        Set<Modifier> modifiers = typeElement.getModifiers();
        boolean visible;
        //【1】该类必须是 public 的；
        if (modifiers.contains(Modifier.PUBLIC)) {
            visible = true;
        //【2】该类不能是 private/protected 的；
        } else if (modifiers.contains(Modifier.PRIVATE) || modifiers.contains(Modifier.PROTECTED)) {
            visible = false;
        } else {
            //【3】默认访问权限；
            //【-->2.2.2.1.1】获得注解方法所属类的包元素名（包名）；
            String subscriberPackage = getPackageElement(typeElement).getQualifiedName().toString();
            if (myPackage == null) {
                visible = subscriberPackage.length() == 0;
            } else {
                // 正常情况进入这里：
                //【3】就是说注解方法所说的类必须和 eventBusIndex 指定的要动态生成的 java 类属于同一个包下；
                visible = myPackage.equals(subscriberPackage); 
            }
        }
        return visible;
    }
```

**参数 myPackage** 是我们 eventBusIndex 指定的要动态生成的 java 类的 class Name；

**参数 TypeElement typeElement** 则是注解方法所在的类元素；

可以看到，注解方法所属的类必须要满足一下的条件：

- 如果是 **public**，那就是可见的 visible 为 true；
- 如果是 **private/protected**，那就是不可见的；
- 如果是 **default**，那么 必须和 **eventBusIndex** 指定的要动态生成的 **java** 类属于同一个包下；



##### 2.2.2.1.1 getPackageElement

获得注解方法所属类的包元素：

```java
    private PackageElement getPackageElement(TypeElement subscriberClass) {
        Element candidate = subscriberClass.getEnclosingElement();
        //【1】这里的 while 不断循环处理，直到 candidate 是一个包元素；
        while (!(candidate instanceof PackageElement)) {
            candidate = candidate.getEnclosingElement();
        }
        return (PackageElement) candidate;
    }
```

这里用到了 **TypeElement. getenclosingelement()** 的方法：


> 返回封装此元素（非严格意义上）的最里层元素。
>
> 如果此元素的声明在词法上直接封装在另一个元素的声明中，则返回那个封装元素。
> 如果此元素是顶层类型，则返回它的包。
> 如果此元素是一个包，则返回 null。
> 如果此元素是一个类型参数，则返回 null。



#### 2.2.2.2 getParamTypeMirror

获取注解方法的参数的类型：

```java
    private TypeMirror getParamTypeMirror(VariableElement param, Messager messager) {
        //【1】获取参数的类型；
        TypeMirror typeMirror = param.asType();
        // Check for generic type
        if (typeMirror instanceof TypeVariable) {
            //【1.1】判断参数类型是否有上边界，如果有的话，那就使用上边界为参数类型；
            TypeMirror upperBound = ((TypeVariable) typeMirror).getUpperBound();
            if (upperBound instanceof DeclaredType) { // 上边界是类或接口类型；
                if (messager != null) {
                    messager.printMessage(Diagnostic.Kind.NOTE, "Using upper bound type " + upperBound +
                            " for generic parameter", param);
                }
                typeMirror = upperBound; // 就将上边界类型作为参数类型：
            }
        }
        return typeMirror;
    }
```

这里我们用到了这个方法 **TypeVariable.getUpperBound()**

> 返回：此类型变量的上边界
>
> 如果此类型变量被声明为没有明确上边界，则结果为 `java.lang.object`。
> 如果此类型变量被声明为有多个上边界，则结果是一个交集类型（建模为 `declaredtype`）。
> 通过检查结果的超类型，可以发现个别边界。



这个是什么意思呢？举个简单的栗子，下面是我们的消息类：

```java
class Message<T> extends BaseMessage {} // 返回上边界 BaseMessage，为了使用多态；
class Message<T> {}      // 返回 Message 对应的类型；
```

这样解释就简单了吧！

#### 2.2.2.3 getSuperclass

获取当前类的父类元素：

```java
    private TypeElement getSuperclass(TypeElement type) {
        //【1】如果当前元素类型是类或者接口，才会获取父类；
        if (type.getSuperclass().getKind() == TypeKind.DECLARED) {
            //【1.1】获取其直接父类；
            TypeElement superclass = (TypeElement) processingEnv.getTypeUtils().asElement(type.getSuperclass());
            String name = superclass.getQualifiedName().toString();
            if (name.startsWith("java.") || name.startsWith("javax.") || name.startsWith("android.")) {
                //【1.1.1】过滤掉 java/javax/android 系统类；
                return null;
            } else {
                return superclass;
            }
        } else {
            return null;
        }
    }
```

TypeKind 是枚举，保存了 Java 定义的所有的类型数据！



### 2.2.3 createInfoIndexFile

动态生成 java 类：

```java
    private void createInfoIndexFile(String index) {
        BufferedWriter writer = null;
        try {
            //【1】创建 JavaFileObject 对象，用于生成 java 类；
            JavaFileObject sourceFile = processingEnv.getFiler().createSourceFile(index);
            //【2】生成 java 包名（eventBusIndex 最后一个 . 前面的字符串）和 java 类名（eventBusIndex 最后一个 . 后面的字符串）；
            int period = index.lastIndexOf('.');
            String myPackage = period > 0 ? index.substring(0, period) : null;
            String clazz = index.substring(period + 1);
            writer = new BufferedWriter(sourceFile.openWriter());
            if (myPackage != null) {
                writer.write("package " + myPackage + ";\n\n"); // 动态 java 类的包名；
            }
            writer.write("import org.greenrobot.eventbus.meta.SimpleSubscriberInfo;\n");
            writer.write("import org.greenrobot.eventbus.meta.SubscriberMethodInfo;\n");
            writer.write("import org.greenrobot.eventbus.meta.SubscriberInfo;\n");
            writer.write("import org.greenrobot.eventbus.meta.SubscriberInfoIndex;\n\n");
            writer.write("import org.greenrobot.eventbus.ThreadMode;\n\n");
            writer.write("import java.util.HashMap;\n");
            writer.write("import java.util.Map;\n\n");
            writer.write("/** This class is generated by EventBus, do not edit. */\n");
            writer.write("public class " + clazz + " implements SubscriberInfoIndex {\n"); // 动态 java 类的类名
            writer.write("    private static final Map<Class<?>, SubscriberInfo> SUBSCRIBER_INDEX;\n\n");
            writer.write("    static {\n");
            writer.write("        SUBSCRIBER_INDEX = new HashMap<Class<?>, SubscriberInfo>();\n\n");
            //【-->2.2.3.1】写入注解生成的信息；
            writeIndexLines(writer, myPackage);
            writer.write("    }\n\n");
            writer.write("    private static void putIndex(SubscriberInfo info) {\n"); // 写入内部的 putIndex 方法；
            writer.write("        SUBSCRIBER_INDEX.put(info.getSubscriberClass(), info);\n");
            writer.write("    }\n\n");
            writer.write("    @Override\n");
            writer.write("    public SubscriberInfo getSubscriberInfo(Class<?> subscriberClass) {\n"); // 写入的 getSubscriberInfo 方法；
            writer.write("        SubscriberInfo info = SUBSCRIBER_INDEX.get(subscriberClass);\n");
            writer.write("        if (info != null) {\n");
            writer.write("            return info;\n");
            writer.write("        } else {\n");
            writer.write("            return null;\n");
            writer.write("        }\n");
            writer.write("    }\n");
            writer.write("}\n");
        } catch (IOException e) {
            throw new RuntimeException("Could not write source for " + index, e);
        } finally {
            if (writer != null) {
                try {
                    writer.close();
                } catch (IOException e) {
                    //Silent
                }
            }
        }
    }
```

这里的代码有些 low，竟然是硬编码写进去的；

生成了的代码会涉及到如下的类：

```java
import org.greenrobot.eventbus.meta.SimpleSubscriberInfo;
import org.greenrobot.eventbus.meta.SubscriberMethodInfo;
import org.greenrobot.eventbus.meta.SubscriberInfo;
import org.greenrobot.eventbus.meta.SubscriberInfoIndex;
import org.greenrobot.eventbus.ThreadMode;
```

可以看到，这几个类定义在 eventbus 模块里，简单的说下：

- **SimpleSubscriberInfo**：表示一个订阅者，就是 Subscribe 注解所在的类；
- **SubscriberMethodInfo**：表示一个订阅方法，就是 Subscribe 注解的方法；
- **SubscriberInfo**：接口，SimpleSubscriberInfo 继承了 AbstractSubscriberInfo，而 AbstractSubscriberInfo 实现了  SubscriberInfo 接口，**适配器模式**；
- **SubscriberInfoIndex**：接口，我们动态生成的 Java 类，实现了该接口；
- **ThreadMode**：枚举类型，表示线程类型；

这里我们不多关注；

#### 2.2.3.1 writeIndexLines

这里就是将 methodsByClass 中收集到的信息写入到动态 java 类中；

```java
    private void writeIndexLines(BufferedWriter writer, String myPackage) throws IOException {
        //【1】遍历 methodsByClass 哈希表，跳过 classesToSkip 中的元素；
        for (TypeElement subscriberTypeElement : methodsByClass.keySet()) {
            if (classesToSkip.contains(subscriberTypeElement)) {
                continue;
            }
            //【-->2.2.3.1.1】获得注解方法所在的类名；
            String subscriberClass = getClassString(subscriberTypeElement, myPackage);
            //【-->2.2.2.1】判断下动态 java 类所在的包是否可以访问注解所在类，可以的话，才写入！ 
            if (isVisible(myPackage, subscriberTypeElement)) {
                writeLine(writer, 2,
                        "putIndex(new SimpleSubscriberInfo(" + subscriberClass + ".class,",
                        "true,", "new SubscriberMethodInfo[] {"); // 这个我就不分析了，一行一行的写入呗；
                //【2】获取注解的方法；
                List<ExecutableElement> methods = methodsByClass.get(subscriberTypeElement);
                //【-->2.2.3.1.2】将方法信息写入到 java 类中；
                writeCreateSubscriberMethods(writer, methods, "new SubscriberMethodInfo", myPackage);
                writer.write("        }));\n\n");
            } else {
                writer.write("        // Subscriber not visible to index: " + subscriberClass + "\n");
            }
        }
    }
```

第二个参数表示的是否检查父类：**shouldCheckSuperclass**，传入的是 true；

##### 2.2.3.1.1 getClassString

获取类名；

```java
    private String getClassString(TypeElement typeElement, String myPackage) {
        //【-->2.2.2.1.1】获取注解所在的类的包元素；
        PackageElement packageElement = getPackageElement(typeElement);
        //【1】获取所在包名；
        String packageString = packageElement.getQualifiedName().toString();
        //【2】获取类的全限定名；
        String className = typeElement.getQualifiedName().toString();
        if (packageString != null && !packageString.isEmpty()) {
            if (packageString.equals(myPackage)) {
                //【3】如果注解所在的类和动态生成的 java 类的包名一样；就截掉全限定名的包名部分（因为在同一个包嘛）
                className = cutPackage(myPackage, className);
            } else if (packageString.equals("java.lang")) {
                className = typeElement.getSimpleName().toString();
            }
        }
        return className;
    }
```

这里调用内部的 cutPackage 去截取类名！

代码简单，就 String 的基本操作。。。



##### 2.2.3.1.2 writeCreateSubscriberMethods

将方法信息写入到 java 类中，参数 **String callPrefix** 的值："new SubscriberMethodInfo"

```java
    private void writeCreateSubscriberMethods(BufferedWriter writer, List<ExecutableElement> methods,
                                              String callPrefix, String myPackage) throws IOException {
        //【1】遍历方法 list；
        for (ExecutableElement method : methods) {
            List<? extends VariableElement> parameters = method.getParameters();
            TypeMirror paramType = getParamTypeMirror(parameters.get(0), null); //【-->2.2.2.2】注解方法的参数的类型;
            TypeElement paramElement = (TypeElement) processingEnv.getTypeUtils().asElement(paramType);
            //【1.1】获取方法名；
            String methodName = method.getSimpleName().toString();
            //【-->2.2.3.1.2】获取方法参数（事件）的类名；
            String eventClass = getClassString(paramElement, myPackage) + ".class";

            //【1.3】获取 Subscribe 注解对象；
            Subscribe subscribe = method.getAnnotation(Subscribe.class);
            List<String> parts = new ArrayList<>();
            parts.add(callPrefix + "(\"" + methodName + "\","); //【1.4】第一个参数：methodName；
            String lineEnd = "),";
            //【1.5】处理注解的 priority、sticky、threadMode 属性；
            if (subscribe.priority() == 0 && !subscribe.sticky()) { // 如果优先级为 0（默认）并且不是 sticky 事件，那么会进入 if；
                // 如果是默认类型的线程池，只要写入事件的类名；
                // 不是默认线程，那么还要写入线程枚举类型；
                if (subscribe.threadMode() == ThreadMode.POSTING) {
                    parts.add(eventClass + lineEnd);
                } else {
                    parts.add(eventClass + ",");
                    parts.add("ThreadMode." + subscribe.threadMode().name() + lineEnd); // 处理线程类型；
                }
            } else { // 如果指定了优先级，或者是粘性事件，这里会写入事件的类名，线程枚举类型，优先级，粘性状态；
                parts.add(eventClass + ",");
                parts.add("ThreadMode." + subscribe.threadMode().name() + ",");
                parts.add(subscribe.priority() + ",");
                parts.add(subscribe.sticky() + lineEnd);
            }
            writeLine(writer, 3, parts.toArray(new String[parts.size()]));

            if (verbose) {
                processingEnv.getMessager().printMessage(Diagnostic.Kind.NOTE, "Indexed @Subscribe at " +
                        method.getEnclosingElement().getSimpleName() + "." + methodName +
                        "(" + paramElement.getSimpleName() + ")");
            }

        }
    }
```

这个过程是处理注解方法和注解参数的过程；

# 3 动态 Java 类实例

我们可以看下动态生成的 Java 类实例：

```java
package com.coolqi.top;

import org.greenrobot.eventbus.meta.SimpleSubscriberInfo;
import org.greenrobot.eventbus.meta.SubscriberMethodInfo;
import org.greenrobot.eventbus.meta.SubscriberInfo;
import org.greenrobot.eventbus.meta.SubscriberInfoIndex;

import org.greenrobot.eventbus.ThreadMode;

import java.util.HashMap;
import java.util.Map;

/** This class is generated by EventBus, do not edit. */
public class moduleAppIndex implements SubscriberInfoIndex {
    private static final Map<Class<?>, SubscriberInfo> SUBSCRIBER_INDEX;

    static {
        SUBSCRIBER_INDEX = new HashMap<Class<?>, SubscriberInfo>();
      
        putIndex(new SimpleSubscriberInfo(com.coolqi.ui.EditPicActivity.class, true, new SubscriberMethodInfo[] {
            new SubscriberMethodInfo("onEventMainThread", com.coolqi.common.beans.MessageEvent.class),
            new SubscriberMethodInfo("onEventMainThread2", com.coolqi.common.beans.MessageEvent.class,
                    ThreadMode.ASYNC, 1, true),
        }));

        putIndex(new SimpleSubscriberInfo(com.coolqi.ui.ChangeDateActivity.class, true, new SubscriberMethodInfo[] {
            new SubscriberMethodInfo("onEventMainThread", com.coolqi.common.beans.MessageEvent.class),
        }));

        putIndex(new SimpleSubscriberInfo(com.coolqi.ui.normal.ExhibitionWebFragment.class, true,
                new SubscriberMethodInfo[] {
            new SubscriberMethodInfo("onShowMessageChatNumberEvent", com.gensee.kzkt_res.bean.MessageChatNumber.class),
            new SubscriberMethodInfo("onEventMainThread", com.coolqi.common.beans.MessageEvent.class),
        }));
    }

    private static void putIndex(SubscriberInfo info) {
        SUBSCRIBER_INDEX.put(info.getSubscriberClass(), info);
    }

    @Override
    public SubscriberInfo getSubscriberInfo(Class<?> subscriberClass) {
        SubscriberInfo info = SUBSCRIBER_INDEX.get(subscriberClass);
        if (info != null) {
            return info;
        } else {
            return null;
        }
    }
```

下面我们简单的看下涉及到的类和接口，这些类都位于 eventbus 模块中！

后面再分析的时候，我们就不再过多关注这些类了！



## 3.1 SubscriberInfoIndex

接口，动态生成的类实现该接口：

```java
public interface SubscriberInfoIndex {
    //【1】用于获取订阅信息；
    SubscriberInfo getSubscriberInfo(Class<?> subscriberClass);
}
```

## 3.2 SubscriberInfo

接口，订阅者类实现该接口：

```java
public interface SubscriberInfo {
    Class<?> getSubscriberClass(); // 获取订阅者对应的类；

    SubscriberMethod[] getSubscriberMethods(); // 获取订阅方法；

    SubscriberInfo getSuperSubscriberInfo(); // 获取父类订阅者；

    boolean shouldCheckSuperclass(); // 是否检查父类，动态生成时，传入的是 true；
}
```



## 3.3 AbstractSubscriberInfo

抽象类，实现了 **SubscriberInfo** 接口，并实现了其部分接口，**适配器模式**：

```java
public abstract class AbstractSubscriberInfo implements SubscriberInfo {
    private final Class subscriberClass;
    private final Class<? extends SubscriberInfo> superSubscriberInfoClass;
    private final boolean shouldCheckSuperclass;

    protected AbstractSubscriberInfo(Class subscriberClass, Class<? extends SubscriberInfo> superSubscriberInfoClass,
                                     boolean shouldCheckSuperclass) {
        this.subscriberClass = subscriberClass; // 订阅者类；
        this.superSubscriberInfoClass = superSubscriberInfoClass; // 订阅者的父类订阅者，processor 动态生成时传入的是 null；
        this.shouldCheckSuperclass = shouldCheckSuperclass; //  是否检查父类，processor 动态生成时传入的是 true；
    }

    @Override
    public Class getSubscriberClass() {
        return subscriberClass;
    }

    @Override
    public SubscriberInfo getSuperSubscriberInfo() {
        if(superSubscriberInfoClass == null) {
            return null;
        }
        try {
            return superSubscriberInfoClass.newInstance(); // 返回父类订阅者的实例；
        } catch (InstantiationException | IllegalAccessException e) {
            throw new RuntimeException(e);
        }
    }

    @Override
    public boolean shouldCheckSuperclass() {
        return shouldCheckSuperclass;
    }

    // 下面是创建订阅方法；
    protected SubscriberMethod createSubscriberMethod(String methodName, Class<?> eventType) {
        return createSubscriberMethod(methodName, eventType, ThreadMode.POSTING, 0, false);
    }

    protected SubscriberMethod createSubscriberMethod(String methodName, Class<?> eventType, ThreadMode threadMode) {
        return createSubscriberMethod(methodName, eventType, threadMode, 0, false);
    }

    protected SubscriberMethod createSubscriberMethod(String methodName, Class<?> eventType, ThreadMode threadMode,
                                                      int priority, boolean sticky) {
        try {
            // 显然这里是通过反射的方式来创建！
            Method method = subscriberClass.getDeclaredMethod(methodName, eventType);
            return new SubscriberMethod(method, eventType, threadMode, priority, sticky);
        } catch (NoSuchMethodException e) {
            throw new EventBusException("Could not find subscriber method in " + subscriberClass +
                    ". Maybe a missing ProGuard rule?", e);
        }
    }

}
```



## 3.4 SimpleSubscriberInfo

订阅类，继承了 AbstractSubscriberInfo 类，**适配器模式**：

```java
public class SimpleSubscriberInfo extends AbstractSubscriberInfo {

    private final SubscriberMethodInfo[] methodInfos; // 保存的是订阅方法；

    public SimpleSubscriberInfo(Class subscriberClass, boolean shouldCheckSuperclass, SubscriberMethodInfo[] methodInfos) {
        super(subscriberClass, null, shouldCheckSuperclass); //【-->3.3】抽象类的方法；
        this.methodInfos = methodInfos;
    }

    @Override
    public synchronized SubscriberMethod[] getSubscriberMethods() { // 返回所有的订阅方法；
        int length = methodInfos.length;
        SubscriberMethod[] methods = new SubscriberMethod[length];
        for (int i = 0; i < length; i++) {
            SubscriberMethodInfo info = methodInfos[i];
            //【-->3.3】注意并不是直接返回，而是返回了一份拷贝，防止修改；
            methods[i] = createSubscriberMethod(info.methodName, info.eventType, info.threadMode,
                    info.priority, info.sticky);
        }
        return methods;
    }
}
```

不多说了！

# 4 总结

本篇文章，分析了 eventbus 的注解是如何处理的，生成了哪些类，类的关系如何（适配器模式）；

下篇文章，分析 register 的过程；