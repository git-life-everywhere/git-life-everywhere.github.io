#  JsBridge 第四篇 - H5 调用 Native 代码
title:  JsBridge 第四篇 - H5 调用 Native 代码
date: 2019/05/23 20:46:25
categories: 
- 开源库源码分析
- JsBridge
tags: JsBridge
---

本系列文章主要分析 JsBridge 框架的原理，学习 H5 和原生 WebView 的交互方式，框架选自 GitHub 上的很火的 H5 + WebView 三方库：lzyzsd/JsBridge，作者是大鬼头；

# 1 调用接口

在 js 中，我们通过如下方式，使用 jsBridge 框架来和 Native 通信：

```javascript
var data = {id: 1, content: "这是一个图片 <img src=\"a.png\"/> test\r\nhahaha"};
//【1】通过 js 通信协议 send 方法；
window.WebViewJavascriptBridge.send(
    data
    , function(responseData) {
        document.getElementById("show").innerHTML = "repsonseData from java, data = " + responseData
    }
);


//【2】通过 js 通信协议 callHandler 方法；
window.WebViewJavascriptBridge.callHandler(
    'submitFromWeb'
    , {'param': '中文测试'}
    , function(responseData) {
        document.getElementById("show").innerHTML = "send get responseData from java, data = " + responseData
    }
);
```
- **send 方法**；用客户端默认的 handler 处理；
- **callHandler 方法**：用指定的 handler 处理；

下面我们来分析下 callHandler 和 send 方法！

# 2 WebViewJavascriptBridge

接下来进入了 js 通信协议文件中：

## 2.1 send

用客户端默认的 handler 处理

```javascript
function send(data, responseCallback) {
    //【-->*2.3】调用 _doSend 方法；
    _doSend({
        data: data
    }, responseCallback);
}
```


## 2.2 callHandler

用指定的 handler 处理

```javascript
function callHandler(handlerName, data, responseCallback) {
    //【-->*2.3】调用 _doSend 方法；
    _doSend({
        handlerName: handlerName,
        data: data
    }, responseCallback);
}
```

## 2.3 _doSend

最后都调用了 _doSend 的方法：

```javascript
//sendMessage add message, 触发native处理 sendMessage
function _doSend(message, responseCallback) {   
    if (responseCallback) {  
        //【1】创建了一个 calbackId，并将 id 和 callback 的映射关系保存到 responseCallbacks 中；
        // 将 callbackId 保存到 message 中！
        var callbackId = 'cb_' + (uniqueId++) + '_' + new Date().getTime();
        responseCallbacks[callbackId] = responseCallback;
        message.callbackId = callbackId;
    }
    //【2】将消息保存到 sendMessageQueue 中，创建动态 url，通知 native；
    sendMessageQueue.push(message);
    //【3】yy://__QUEUE_MESSAGE__，最终会触发如下方法
    //【-->*3.1】BridgeWebView.flushMessageQueue
    messagingIframe.src = CUSTOM_PROTOCOL_SCHEME + '://' + QUEUE_HAS_MESSAGE;
}
```
这里在前面分析过了，和前面的类似。

messagingIframe.src 最终会触发如下方法调用链：

```xml
BridgeWebViewClient.shouldOverrideUrlLoading ---> BridgeWebView.flushMessageQueue
```


## 2.4 _fetchQueue

从 sendMessageQueue 队列中获取 message，发送给 native：
```javascript
function _fetchQueue() {
    //【1】这里是统一处理要发给 native 的所有消息，将队列转为 string
    var messageQueueString = JSON.stringify(sendMessageQueue);
    sendMessageQueue = [];
    //【2】这里又再在页面生成 url，和之前的区别是包含数据，然后再次通过 shouldOverrideUrlLoading 方法拦截，
    // 捕获 url 中的数据；
    if (messageQueueString !== '[]') {
        //【-->*3.3】这一次，生成的 url 将真正带有回调数据；
        bizMessagingIframe.src = CUSTOM_PROTOCOL_SCHEME + '://return/_fetchQueue/' + encodeURIComponent(messageQueueString);
    }
}
```

这里生成了一个新的 url：

> yy://://return/_fetchQueue/[{"responseId":"xxxxxxx","responseData":"xxxxxxxxxx"}]

messagingIframe.src 最终会触发如下方法调用链：

```xml
BridgeWebViewClient.shouldOverrideUrlLoading ---> BridgeWebView.handlerReturnData
```

## 2.5 _handleMessageFromNative

js 代码中会处理 native 发送的 message json：

```javascript
function _handleMessageFromNative(messageJSON) {
    console.log(messageJSON);
    //【1】如果 receiveMessageQueue 不为 null，那么会讲她加入到
    // receiveMessageQueue 队列中，它是用来保存 native 发送的消息的；
    if (receiveMessageQueue) {
        receiveMessageQueue.push(messageJSON);
    }
    //【-->*2.6】分发来自 native 的消息；
    _dispatchMessageFromNative(messageJSON);
}
```
在第二篇 js 协议中有讲过：

当在动态注入 js 脚本时，会执行 init 方法，那里会将 receiveMessageQueue 置为 null，同时处理已经包含的 native 消息；

所以这里就直接 _dispatchMessageFromNative 了；

## 2.6 _dispatchMessageFromNative

js 处理 native 层的回调消息：

```javascript
    function _dispatchMessageFromNative(messageJSON) {
        setTimeout(function() {
            var message = JSON.parse(messageJSON);
            var responseCallback;
            //【1】这里 js 处理消息回调，不多说了！
            if (message.responseId) {
                responseCallback = responseCallbacks[message.responseId];
                if (!responseCallback) {
                    return;
                }
                responseCallback(message.responseData);
                delete responseCallbacks[message.responseId];
            } else {
                ... ... ...// 这里是处理 java 回调的，之前分析过；
            }
        });
    }
```

# 3 BridgeWebView

## 3.1 flushMessageQueue

native 读取 js 的命令：

```java
void flushMessageQueue() {
    //【1】必须在主线程（loadUrl）
	if (Thread.currentThread() == Looper.getMainLooper().getThread()) {
	    //【--->*3.2】执行 js 脚本
		loadUrl(BridgeUtil.JS_FETCH_QUEUE_FROM_JAVA, new CallBackFunction() {

            //【*3.1.1】这个回调是用来分发 url 对应的数据给相应的回调！
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
					//【3】如果 Message.responseId 不为 null，说明这是 js 反馈给 native 的回调数据。
					// 此时 responseId 表示 native 回调函数的 id！
					if (!TextUtils.isEmpty(responseId)) {
					    ... ... ... ...// 这里前面有分析过；

					} else {
					    //【4】这里是我们要关注的地方：
					    // 这种情况是属于 js 主动向 Handler 发送消息的时候，callbackId 显然是 js 的回调函数 id；
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
									
									//【-->*3.4】将消息加入 list，等待处理；
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
						//【4.4】处理 js 的 message，并发送回调信息给 js。
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
这里可以看大了，给 js 反馈回调的时候：

```java
Message responseMsg = new Message();
responseMsg.setResponseId(callbackId);
responseMsg.setResponseData(data);
```
js 传入的 callbackId 被设置到了 responseId 上了；

## 3.2 loadUrl

参数 jsUrl 是 javascript:WebViewJavascriptBridge._fetchQueue();

```java
public void loadUrl(String jsUrl, CallBackFunction returnCallback) {
    //【-->*2.4】执行 jsUrl 命令；
	this.loadUrl(jsUrl);
    //【2】同时将 CallBackFunction 放入到 responseCallbacks 中；
	responseCallbacks.put(BridgeUtil.parseFunctionName(jsUrl), returnCallback);
}
```
这里有分析过，对 js 命令做了处理，生成了 key：

> javascript:WebViewJavascriptBridge._fetchQueue(); –> _fetchQueue

用于保存回调；

## 3.3 handlerReturnData

```java
void handlerReturnData(String url) {
    //【-->3*7.2】再次解析 url，获得回调的 key：functionName
	String functionName = BridgeUtil.getFunctionFromReturnUrl(url);
	//【1】functionName 就是 _fetchQueue，这里我们获得了【*3.3】注册的 callback
	CallBackFunction f = responseCallbacks.get(functionName);
	//【-->3*7.2】获得 js 的回调返回数据；
	String data = BridgeUtil.getDataFromReturnUrl(url);
	if (f != null) {
	    //【-->*3.1.1】执行回调，处理数据！
		f.onCallBack(data);
		//【2】删除该 _fetchQueue 对应的回调（我觉得可以不删除的）
		responseCallbacks.remove(functionName);
		return;
	}
}
```
触发前面的 _fetchQueue 对应的回调；


## 3.4 queueMessage

加入 message list：

```java
	private void queueMessage(Message m) {
		if (startupMessage != null) {
			startupMessage.add(m);
		} else {
		    //【-->*3.5】分发 message 给 js；
			dispatchMessage(m);
		}
	}
```

## 3.5 dispatchMessage

native 给 js 发送消息的关键点，参数 message 是一个消息对象！

```java
void dispatchMessage(Message m) {
    //【1】将 message 转为 json
    String messageJson = m.toJson();
    //【2】为 message json 字符串转义特殊字符；
    messageJson = messageJson.replaceAll("(\\\\)([^utrn])", "\\\\\\\\$1$2");
    messageJson = messageJson.replaceAll("(?<=[^\\\\])(\")", "\\\\\"");
	messageJson = messageJson.replaceAll("(?<=[^\\\\])(\')", "\\\\\'");
	messageJson = messageJson.replaceAll("%7B", URLEncoder.encode("%7B"));
	messageJson = messageJson.replaceAll("%7D", URLEncoder.encode("%7D"));
	messageJson = messageJson.replaceAll("%22", URLEncoder.encode("%22"));
	//【3】创建要执行的 js 代码，用于和 H5 通信；
    String javascriptCommand = String.format(BridgeUtil.JS_HANDLE_MESSAGE_FROM_JAVA, messageJson);
    //【4】必须要找主线程才会将数据传递出去 --- 划重点
    if (Thread.currentThread() == Looper.getMainLooper().getThread()) {
        【-->*2.5】执行 js 代码；
        this.loadUrl(javascriptCommand);
    }
}
```

BridgeUtil 是一个工具类，里面主要是一些通信协议码，以及一些工具方法，native 和 H5 通信的时候，本质上是执行 js 代码：

> final static String JS_HANDLE_MESSAGE_FROM_JAVA = 
"javascript:WebViewJavascriptBridge._handleMessageFromNative('%s');";

可以看到，执行的 js 代码如下：

> javascript:WebViewJavascriptBridge._handleMessageFromNative(JsonString of Message);

我相信大家知道，这个方法将进入通信协议 js 文件了！






