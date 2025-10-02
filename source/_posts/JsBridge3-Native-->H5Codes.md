# JsBridge 第三篇 - Native 调用 H5 代码
title: JsBridge 第三篇 - Native 调用 H5 代码
date: 2019/05/23 20:46:25
categories: 
- 开源库源码分析
- JsBridge
tags: JsBridge
---

本系列文章主要分析 JsBridge 框架的原理，学习 H5 和原生 WebView 的交互方式，框架选自 GitHub 上的很火的 H5 + WebView 三方库：lzyzsd/JsBridge，作者是大鬼头；

# 1 调用接口

在 android 中，我们通过如下方式，使用 jsBridge 框架来和 H5 通信：

```java
//【1】封装 Java 层的 bean 数据；
User user = new User();
Location location = new Location();
location.address = "SDU";
user.location = location;
user.name = "大头鬼";

//【2】发送数据，并注册回调函数 CallBackFunction：
webView.callHandler("functionInJs", new Gson().toJson(user), new CallBackFunction() {
    @Override
    public void onCallBack(String data) {
    }
});
//【3】这个是不需要回调的，直接发送数据给 H5；
webView.send("hello");
```
这里涉及到 2 个交互接口：

- webView.callHandler：有回调；
- webView.send：没有回调；

下面我会来分析下这两个方法的调用链，这会涉及到 jsBridge 中其他的类：

```java
|____Message.java
|____WebViewJavascriptBridge.java
|____DefaultHandler.java
|____BridgeWebView.java
|____BridgeWebViewClient.java
|____CallBackFunction.java
|____BridgeHandler.java
|____BridgeUtil.java
```
在分析交互流程的时候，我们会详细分析每个类的内部逻辑！

# 2 BridgeWebView

我们从 BridgeWebView 开始，先看看内部的一些重要成员变量：
```java
//【1】native 的回调函数 map，key 是 id，value 是具体的回调对象；
Map<String, CallBackFunction> responseCallbacks = new HashMap<String, CallBackFunction>();
//【2】native 处理 js 消息的 handler map，key 是 handler name，value 是具体的 handler
Map<String, BridgeHandler> messageHandlers = new HashMap<String, BridgeHandler>();
//【3】默认的 handler，默认是它处理 js 的消息；
BridgeHandler defaultHandler = new DefaultHandler();
//【4】native 发送给 js 的 message 列表；
private List<Message> startupMessage = new ArrayList<Message>();
```
不多说了。

## 2.1 callHandler

我们先分析有回调的接口的交互流程：

```java
//【1】发送数据，并注册回调函数 CallBackFunction：
webView.callHandler("functionInJs", new Gson().toJson(user), new CallBackFunction() {
    @Override
    public void onCallBack(String data) {
    }
});
```

callHandler 方法指定了 js 使用哪个 Handler 处理 native 的消息！

这里指定了 js 用于处理的 handler name 是 "functionInJs"！

```java
public void callHandler(String handlerName, String data, CallBackFunction callBack) {
    //【*2.2】调用另外一个方法：
    doSend(handlerName, data, callBack);
}
```
我们去看看 doSend 方法：

## 2.2 doSend

继续分析 doSend 方法：

```java
private void doSend(String handlerName, String data, CallBackFunction responseCallback) {
   //【1】创建一个消息；
	Message m = new Message();
	if (!TextUtils.isEmpty(data)) {
	    //【2】设置 data 数据；
		m.setData(data);
	}
	//【3】如果需要回调，那么会创建回调 id（String）
	if (responseCallback != null) {
		String callbackStr = String.format(BridgeUtil.CALLBACK_ID_FORMAT, ++uniqueId + (BridgeUtil.UNDERLINE_STR + SystemClock.currentThreadTimeMillis()));
		responseCallbacks.put(callbackStr, responseCallback);
		m.setCallbackId(callbackStr);
	}
	//【4】如果指定了 handler，那么设置 handlerName；
	if (!TextUtils.isEmpty(handlerName)) {
		m.setHandlerName(handlerName);
	}
	//【*2.3】将消息送入队列；
	queueMessage(m);
}
```

这里我们看到，会创建一个 Message 对象，封装要发给 js 的消息；

同时注意到，native 的回调并没有传递给 js，而是保存在了内部的一个 responseCallbacks 哈希表中；

实际传递给 js 的是 callbackId；

```java
Message.data  // native 发送的数据
Message.callbackId  // native 回调函数的 id
Message.handlerName // js 处理数据的 handlerName；
```

最后就是把 message 放入到 message list；

## 2.3 queueMessage

将 message 放入到 message list；

```java
private void queueMessage(Message m) {
	if (startupMessage != null) {
	    //【1】将消息加入到 message list 中；
		startupMessage.add(m);
	} else {
	    //【*2.4】特殊情况，直接发送 message！
		dispatchMessage(m);
	}
}
```
可以看到，这里默认是会将 message 添加到 startupMessage 消息列表中，然后 webview 会处理 message list！

那么在哪里会处理呢？

前面我们分析过，在网页加载好后，会出发 BridgeWebViewClient.onPageFinished 方法，就会启动 native 的消息处理循环！

**见 【3.1】 节**；

## 2.4 dispatchMessage

native 给 js 发送消息的关键点，参数 message 是一个消息对象！

```java
void dispatchMessage(Message m) {
    //【*4.2】将 message 转为 json
    String messageJson = m.toJson();
    //【1】为 message json 字符串转义特殊字符；
    messageJson = messageJson.replaceAll("(\\\\)([^utrn])", "\\\\\\\\$1$2");
    messageJson = messageJson.replaceAll("(?<=[^\\\\])(\")", "\\\\\"");
	messageJson = messageJson.replaceAll("(?<=[^\\\\])(\')", "\\\\\'");
	messageJson = messageJson.replaceAll("%7B", URLEncoder.encode("%7B"));
	messageJson = messageJson.replaceAll("%7D", URLEncoder.encode("%7D"));
	messageJson = messageJson.replaceAll("%22", URLEncoder.encode("%22"));
	//【2】创建要执行的 js 代码，用于和 H5 通信；
    String javascriptCommand = String.format(BridgeUtil.JS_HANDLE_MESSAGE_FROM_JAVA, messageJson);
    //【3】必须要找主线程才会将数据传递出去 --- 划重点
    if (Thread.currentThread() == Looper.getMainLooper().getThread()) {
        【*5.1】执行 js 代码；
        this.loadUrl(javascriptCommand);
    }
}
```

BridgeUtil 是一个工具类，里面主要是一些通信协议码，以及一些工具方法，native 和 H5 通信的时候，本质上是执行 js 代码：

```java
final static String JS_HANDLE_MESSAGE_FROM_JAVA = "javascript:WebViewJavascriptBridge._handleMessageFromNative('%s');";
```

可以看到，执行的 js 代码如下：

```java
javascript:WebViewJavascriptBridge._handleMessageFromNative(JsonString of Message);
```

我相信大家知道，这个方法将进入通信协议 js 文件了！

## 2.5 handlerReturnData

拦截 url 并处理信息

```java
	void handlerReturnData(String url) {
		String functionName = BridgeUtil.getFunctionFromReturnUrl(url);
		CallBackFunction f = responseCallbacks.get(functionName);
		String data = BridgeUtil.getDataFromReturnUrl(url);
		if (f != null) {
			f.onCallBack(data);
			responseCallbacks.remove(functionName);
			return;
		}
	}
```

# 3 BridgeWebViewClient

WebViewClient 是用于处理各种事件的回调。

## 3.1 onPageFinished

当 H5 页面加载完成后，会 WebViewClient 方法会处罚；

```java
@Override
public void onPageFinished(WebView view, String url) {
    super.onPageFinished(view, url);
    //【1】动态注入 js 协议脚本，这个我们之前有讲过；
    if (BridgeWebView.toLoadJs != null) {
        BridgeUtil.webViewLoadLocalJs(view, BridgeWebView.toLoadJs);
    }

    //【*2.5】这里会遍历 BridgeWebView.startupMessage 分发 native 消息；
    if (webView.getStartupMessage() != null) {
        for (Message m : webView.getStartupMessage()) {
            //【*2.5】分发 native 消息；
            webView.dispatchMessage(m);
        }
        //【*2.3】注意：这里将 BridgeWebView.startupMessage 设置为 null 了
        // 那么下次就不用将消息加入 list 了，而是直接 dispatch 了！
        webView.setStartupMessage(null);
    }

    //【2】调用其他函数处理 url！
    onCustomPageFinishd(view,url);
}
```
看起来最终调用了 webView.dispatchMessage 方法！

## 3.2 shouldOverrideUrlLoading

我们来看看：

```java
    @Override
    public boolean shouldOverrideUrlLoading(WebView view, String url) {
        try {
            //【1】获得动态创建的 url
            url = URLDecoder.decode(url, "UTF-8");
        } catch (UnsupportedEncodingException e) {
            e.printStackTrace();
        }
        
        //【*3.2.2】此时是返回数据，url 携带数据；
        if (url.startsWith(BridgeUtil.YY_RETURN_DATA)) {
            //【*3.5】native 解析 js 返回的数据；
            webView.handlerReturnData(url);
            return true;
        
        //【*3.2.1】此时是提醒 native，js 有数据返回；
        } else if (url.startsWith(BridgeUtil.YY_OVERRIDE_SCHEMA)) { 
            //【*3.3】native 再次和 js 通信，获取数据；
            webView.flushMessageQueue();
            return true;

        } 
        ... ... ...
    }
```

BridgeUtil 定义了如下的 url 前缀规则：
```java
	final static String YY_OVERRIDE_SCHEMA = "yy://";
	//【1】表示 js 有数据返回，提醒 native 去读取数据；
	final static String YY_RETURN_DATA = YY_OVERRIDE_SCHEMA + "return/";
	//【2】该 url 会携带 js 返回的数据；
	final static String YY_FETCH_QUEUE = YY_RETURN_DATA + "_fetchQueue/";
```

## 3.3 flushMessageQueue

核心方法，从 js 的队列里获取要发送给 native 的 message：

```java
	void flushMessageQueue() {
	    //【1】必须在主线程（loadUrl）
		if (Thread.currentThread() == Looper.getMainLooper().getThread()) {
		    //【--->*3.4】执行 js 脚本
			loadUrl(BridgeUtil.JS_FETCH_QUEUE_FROM_JAVA, new CallBackFunction() {

                //【*3.3.1】这个回调是用来分发 url 对应的数据给相应的回调！
				@Override
				public void onCallBack(String data) {
					//【1】用于存储所有的消息；
					List<Message> list = null;
					try {
						list = Message.toArrayList(data);
					} catch (Exception e) {
                        e.printStackTrace();
						return;
					}
					if (list == null || list.size() == 0) {
						return;
					}
					for (int i = 0; i < list.size(); i++) {
					    //【2】遍历处理下每一个 Message。
						Message m = list.get(i);
						String responseId = m.getResponseId();
						//【3】如果 Message.responseId 不为 null，说明这是 js 反馈给 native 的回调数据。此时 responseId 表示 native 回调函数的 id！
						if (!TextUtils.isEmpty(responseId)) {
						    //【3.1】获得 native 在 callHandler 是设定的回调函数！
							CallBackFunction function = responseCallbacks.get(responseId);
							//【3.2】获得回调数据；
							String responseData = m.getResponseData();
							//【-->*2.1】native 处理数据，这里又回到了 callHandler 哪里！
							function.onCallBack(responseData);
							//【3.3】移除 native 注册的回调函数！
							responseCallbacks.remove(responseId);

						} else {
						    //【4】这种情况是属于，js 主动向 Handler 发送消息的时候，callbackId 显然是 js 的回调函数 id；
							CallBackFunction responseFunction = null;
							final String callbackId = m.getCallbackId();
							if (!TextUtils.isEmpty(callbackId)) {
							    //【4.1】有 js 回调的情况，进入这里；
								responseFunction = new CallBackFunction() {
									@Override
									public void onCallBack(String data) {
									    //【4.1.1】可以看到，这里创建了一个 Message，作为给 js 的回调信息；
										Message responseMsg = new Message();
										responseMsg.setResponseId(callbackId);
										responseMsg.setResponseData(data);
										//【-->*2.4】将消息加入 list，等待处理；
										queueMessage(responseMsg);
									}
								};
							} else {
							    //【4.2】无 js 回调的情况，进入这里；
								responseFunction = new CallBackFunction() {
									@Override
									public void onCallBack(String data) {
										// do nothing
									}
								};
							}
							//【4.3】js 指定了 native 处理数据的 handler！
							BridgeHandler handler;
							if (!TextUtils.isEmpty(m.getHandlerName())) {
								handler = messageHandlers.get(m.getHandlerName());
							} else {
								handler = defaultHandler;
							}
							//【4.4】处理 js 的message，并发送回调信息给 js。
							if (handler != null){
								handler.handler(m.getData(), responseFunction);
							}
						}
					}
				}
			});
		}
	}
```
下面部分的代码（【4】)），是和 H5 调用 Native 代码相关的，我们不过多关注！


BridgeUtil 定义了指定 js 协议的 _fetchQueue 方法的命令：
```java
final static String JS_FETCH_QUEUE_FROM_JAVA = "javascript:WebViewJavascriptBridge._fetchQueue();";
```
下面去看一下 loadUrl：

## 3.4 loadUrl

参数 jsUrl 是 **javascript:WebViewJavascriptBridge._fetchQueue();**

```java
public void loadUrl(String jsUrl, CallBackFunction returnCallback) {
    //【*5.4】执行 jsUrl 命令；
	this.loadUrl(jsUrl);
    //【2】同时将 CallBackFunction 放入到 responseCallbacks 中；
	responseCallbacks.put(BridgeUtil.parseFunctionName(jsUrl), returnCallback);
}
```
这里调用了【*7.1】BridgeUtil.parseFunctionName 对 jsUrl 做了处理，以结果作为 key！

处理入下：

> javascript:WebViewJavascriptBridge._fetchQueue(); --> _fetchQueue

这样是有好处了，因为 returnCallback 根据协议是可以复用的，所以这里也保存在了 responseCallbacks 中！！

<br/>

responseCallbacks 之前我们有分析过！**此时 responseCallbacks 放入了 2 个 native 的回调！**

## 3.5 handlerReturnData

改方法用于处理 js 返回给 native 的回调数据：

```java
	void handlerReturnData(String url) {
	    //【*7.2】再次解析 url，获得回调的 key：functionName
		String functionName = BridgeUtil.getFunctionFromReturnUrl(url);
		//【1】functionName 就是 _fetchQueue，这里我们获得了【*3.3】注册的 callback
		CallBackFunction f = responseCallbacks.get(functionName);
		//【*7.3」获得 js 的回调返回数据；
		String data = BridgeUtil.getDataFromReturnUrl(url);
		if (f != null) {
		    //【*3.3.1】执行回调，处理数据！
			f.onCallBack(data);
			//【2】删除该 _fetchQueue 对应的回调（我觉得可以不删除的）
			responseCallbacks.remove(functionName);
			return;
		}
	}
```
这里调用了【*7.2】BridgeUtil.getFunctionFromReturnUrl 对 url 再次做了处理，前面在 【3.3】 flushMessageQueue，我们将另一个解析回调以 _fetchQueue 为 key，保存到了 responseCallbacks 中，这里是触发他的时候了！


# 4 Message

该对象用于封装 native 和 js 交互的信息：

## 4.1 属性

我们来看看他的基本属性：

```java
    // native --> js: native 回调的 id，用于处理 js 的返回信息；
    // js --> native: js 回调的 id，用于处理 native 的返回信息；
	private String callbackId;

    // js --> native: native 回调的 id，用于处理 js 的返回信息；
	private String responseId;
	
	// js --> native: js 发送给 native 的信息：json，string；
	private String responseData; // js 的回调信息，json string；
	
	// native --> js: native 发送给 js 的信息：json，string；
	private String data; 
	
	// native --> js: 处理 native 信息的 js handler name；
	private String handlerName;
```
不多说了！

## 4.2 toJson

将 Message 转为 json string！

```java
public String toJson() {
    JSONObject jsonObject= new JSONObject();
    try {
        jsonObject.put(CALLBACK_ID_STR, getCallbackId()); // callbackId
        jsonObject.put(DATA_STR, getData()); // data
        jsonObject.put(HANDLER_NAME_STR, getHandlerName()); // handlerName
        String data = getResponseData();
        //【3】这个地方我有些疑问，不知道作者为啥这样写
        // 值永远会被第三个覆盖掉；
        if (TextUtils.isEmpty(data)) {
          jsonObject.put(RESPONSE_DATA_STR, data);
        } else {
          jsonObject.put(RESPONSE_DATA_STR, new JSONTokener(data).nextValue());
        }
        jsonObject.put(RESPONSE_DATA_STR, getResponseData()); // responseData
        jsonObject.put(RESPONSE_ID_STR, getResponseId()); // responseId
        return jsonObject.toString();
    } catch (JSONException e) {
        e.printStackTrace();
    }
    return null;
}
```
具体的参数我就不说了，很简单！


# 5 WebViewJavascriptBridge

最后进入了通信协议 js 脚本：

## 5.1 _handleMessageFromNative

js 代码中会处理 native 发送的 message json：

```javascript
    function _handleMessageFromNative(messageJSON) {
        console.log(messageJSON);
        //【1】如果 receiveMessageQueue 不为 null，那么会讲她加入到
        // receiveMessageQueue 队列中，它是用来保存 native 发送的消息的；
        if (receiveMessageQueue) {
            receiveMessageQueue.push(messageJSON);
        }
        //【*5.2】分发来自 native 的消息；
        _dispatchMessageFromNative(messageJSON);
    }
```
在第二篇 js 协议中有讲过：

当在动态注入 js 脚本时，会执行 init 方法，那里会将 receiveMessageQueue 置为 null，同时处理已经包含的 native 消息；

所以这里就直接 _dispatchMessageFromNative 了；

## 5.2 _dispatchMessageFromNative

js 处理 native 层的消息：

```javascript
function _dispatchMessageFromNative(messageJSON) {
    setTimeout(function() {
        //【1】获得 message json 对象；
        var message = JSON.parse(messageJSON);
        var responseCallback;
        if (message.responseId) {
            ... ... ...// 这个地方是 js 回调的地方，我们先不看；
        } else {
            //【2】很显然，此时会进入这里，因为我们设置了 callbackId！
            if (message.callbackId) {
                //【3】获得 callbackId！
                var callbackResponseId = message.callbackId;
                //【*5.2.1】创建 js 回调函数，当回调触发后，会执行 doSend 方法！
                responseCallback = function(responseData) {
                    //【*5.3】将结果以回调形式发送给 native！
                    _doSend({
                        //【4】注意这里，Message.callbackId 的值赋给了 Message.responseId
                        // Message.responseData 用于保存回调数据；
                        responseId: callbackResponseId,
                        responseData: responseData
                    });
                };
            }
            //【5】选择合适的 handler 去处理 native message。
            // 没有指定 handler，就用默认的！
            var handler = WebViewJavascriptBridge._messageHandler;
            if (message.handlerName) {
                handler = messageHandlers[message.handlerName];
            }
            try {
                //【*6.1】handler 其实就是一个函数，这个在通信协议 js 有分析过！
                handler(message.data, responseCallback);
            } catch (exception) {
                if (typeof console != 'undefined') {
                    console.log("WebViewJavascriptBridge: WARNING: javascript handler threw.", message, exception);
                }
            }
        }
    });
}
```
最后会选择合适的 handler，将 native message 和 js 回调函数交给 handler 处理！


## 5.3 _doSend

将结果以回调形式发送给 native！

```javascript
function _doSend(message, responseCallback) {
    //【1】responseCallback 不为 null，说明 native 需要回调通知，这里将 responseCallback
    // 保存到 responseCallbacks 的意义是：可以建立双向通信！
    if (responseCallback) {
        //【2】为该 responseCallback 创建 id，并将 id：responseCallback 的映射关系
        // 保存到 responseCallbacks 数组中！
        var callbackId = 'cb_' + (uniqueId++) + '_' + new Date().getTime();
        responseCallbacks[callbackId] = responseCallback;
        //【3】将 id 保存到 message.callbackId 中；！
        // 因为此时 message 用于保存 js 发给 native 的数据，所以 message.callbackId 保存了 js 的回调函数，
        // 这样 native 就可以和 js 双向通信了！
        message.callbackId = callbackId;
    }
    
    //【4】将消息保存到 sendMessageQueue 中，然后创建 url，
    //【*3.2】这样 BridgeWebViewClient.shouldOverrideUrlLoading 就能拦截这个 url 了；
    sendMessageQueue.push(message);
    messagingIframe.src = CUSTOM_PROTOCOL_SCHEME + '://' + QUEUE_HAS_MESSAGE;
}
```

responseCallback 实际上是 js 处理 native 消息的回调函数，这里给 responseCallback 也分配了一个 id，并保存到 responseCallbacks 中！

目的很明显，是为了建立双向通信！

<br/>

到这里 Message 中的数据发生了变化：
```java
Message.responseId  // native 的回调函数 id
Message.callbackId  // js 的回调函数 id
Message.responseData // js 发送的回调数据；
```

<br/>

这里创建了一个 url：

```java
yy://__QUEUE_MESSAGE__/
```
这个方法会导致 BridgeWebViewClient.shouldOverrideUrlLoading 触发！

## 5.4 _fetchQueue

从 sendMessageQueue 队列中获取 message，发送给 native：

```javascript
function _fetchQueue() {
    //【1】这里是统一处理要发给 native 的所有消息，将队列转为 string
    var messageQueueString = JSON.stringify(sendMessageQueue);
    sendMessageQueue = [];
    //【2】这里又再在页面生成 url，和之前的区别是包含数据，然后再次通过 shouldOverrideUrlLoading 方法拦截，
    // 捕获 url 中的数据；
    if (messageQueueString !== '[]') {
        //【*3.2.2】这一次，生成的 url 将真正带有回调数据；
        bizMessagingIframe.src = CUSTOM_PROTOCOL_SCHEME + '://return/_fetchQueue/' + encodeURIComponent(messageQueueString);
    }
}
```
这里生成了一个新的 url：

```javascript
yy://://return/_fetchQueue/[{"responseId":"xxxxxxx","responseData":"xxxxxxxxxx"}]
```
再次回到了 shouldOverrideUrlLoading：

# 6 H5 页面初始化 js 脚本

在 jsBridge 框架中，当 js 协议脚本被动态注入到 H5 中时，会触发 H5 页面中的初始化 js 脚本，该脚本会初始化 js 的 handler：

## 6.1 connectWebViewJavascriptBridge

可以看到，H5 页面注册的 js handler 的名字就是 "functionInJs" 这个和前面 callHandler 相符合了！

```javascript
        connectWebViewJavascriptBridge(function(bridge) {
            ... ... ...
            //【1】后面的 function 就是我们的 handler。
            bridge.registerHandler("functionInJs", function(data, responseCallback) {
                document.getElementById("show").innerHTML = ("data from Java: = " + data);
                if (responseCallback) {
                    var responseData = "Javascript Says Right back aka!";
                    //【*5.2.1】js 处理完 message 后，回调 responseCallback 接口！
                    // 其实就是【*5.2】创建的回调，会触发 _doSend 方法！
                    responseCallback(responseData);
                }
            });
        })
```
这个在前面的 js 通信协议中有分析过，不多说了！




# 7 BridgeUtil

工具类，包含一些解析方法和协议头常量：

## 7.1 parseFunctionName

从 url 中解析 funtion name：

```java
// url 的一个例子：javascript:WebViewJavascriptBridge._fetchQueue();
public static String parseFunctionName(String jsUrl){
    //【1】返回_fetchQueue
	return jsUrl.replace("javascript:WebViewJavascriptBridge.", "").replaceAll("\\(.*\\);", "");
}
```

该方法是在 js 创建 url，通知 native 有回调消息后调用的！

## 7.2 getFunctionFromReturnUrl

从 url 中解析 funtion name：

```java
	// 下面是 url 的一个例子；
	// url = yy://return/_fetchQueue/[{"responseId":"xxxxx","responseData":"xxxxx"}]
	public static String getFunctionFromReturnUrl(String url) {
	    //【1】去掉 "yy://return/";
		String temp = url.replace(YY_RETURN_DATA, EMPTY_STR);
		//【2】去掉 "/[{"responseId":"xxxxx","responseData":"xxxxx"}]"
		String[] functionAndData = temp.split(SPLIT_MARK);
		if(functionAndData.length >= 1){
			//【3】我们得到了 key，也就是 functionName；
			return functionAndData[0];
		}
		return null;
	}
```
该方法是在 native 获取到 js 消息后调用的！


## 7.3 getDataFromReturnUrl

```java
// 下面是 url 的一个例子；
// url = yy://return/_fetchQueue/[{"responseId":"JAVA_CB_2_3957","responseData":"xxxxx"}]
public static String getDataFromReturnUrl(String url) {
	if(url.startsWith(YY_FETCH_QUEUE)) {
		//【1】返回了 [{"responseId":"JAVA_CB_2_3957","responseData":"xxxxx"}]
		return url.replace(YY_FETCH_QUEUE, EMPTY_STR);
	}

	// temp = _fetchQueue/[{"responseId":"JAVA_CB_2_3957","responseData":"Javascript Says Right back aka!"}]
	//【2】对另外一种情况的处理
	String temp = url.replace(YY_RETURN_DATA, EMPTY_STR);
	String[] functionAndData = temp.split(SPLIT_MARK);

	if(functionAndData.length >= 2) {
		StringBuilder sb = new StringBuilder();
		for (int i = 1; i < functionAndData.length; i++) {
			sb.append(functionAndData[i]);
		}
        //【3】返回结果是一样的！
		return sb.toString();
	}
	return null;
}
```

该方法是在 native 获取到 js 消息后调用的，并且在【7.2】调用以后才调用！


