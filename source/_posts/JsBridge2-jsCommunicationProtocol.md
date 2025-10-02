# JsBridge 第二篇 - js 通信协议分析
title: JsBridge 第二篇 - js 通信协议分析
date: 2019/05/21 20:46:25
categories: 
- 开源库源码分析
- JsBridge
tags: JsBridge
---


本系列文章主要分析 JsBridge 框架的原理，学习 H5 和原生 WebView 的交互方式，框架选自 GitHub 上的很火的 H5 + WebView 三方库：lzyzsd/JsBridge，作者是大鬼头；


# 1 初步分析

下面分析下 jsBridge 框架的通信协议，他是实际上是一个 js 文件，位于 assets 目录下：

```java
WebViewJavascriptBridge.js
```

这个 js 文件作为协议，决定了 H5 和 Native 代码通信方式和通信数据！

<br/> 

这里就有一个问题了，他是如何被加载并生效的呢，有两种方式：

- 第一种方式：通过 H5 直接加载；
- 第二种方式：通过动态注入的方式：

```java
BridgeUtil.webViewLoadLocalJs(view, BridgeWebView.toLoadJs);
```
我们的 jsBridge 框架也是用的第二种方式，具体的逻辑我们后面再分析；


# 2 协议代码分析

下面我们分析下 js 协议代码的逻辑：

```javascript
(function() {
    //【1】判断变量 WebViewJavascriptBridge 是否初始化过了；
    if (window.WebViewJavascriptBridge) {
        return;
    }
    ... ... ...
})();
```

这里来看的话，其实他是一个 js function，当我们将 js 动态注入到 H5 中时，这么这个 function 就会执行；


## 2.1 内部关键变量

js 文件中定义了一些关键的变量：

```javascript
    var messagingIframe; // 这两个变量用与 android 获取 js 的数据；
    var bizMessagingIframe;
    var sendMessageQueue = [];  // 发送的消息队列，H5 传递给 Native
    var receiveMessageQueue = []; // 接受的消息队列，Native 传递给 H5
    var messageHandlers = {}; // js 处理 native 消息的 handler 数组！

    var CUSTOM_PROTOCOL_SCHEME = 'yy'; // 消息的 scheme，类似与 file，content；
    var QUEUE_HAS_MESSAGE = '__QUEUE_MESSAGE__/';

    var responseCallbacks = {}; // js 端的回调数组；
    var uniqueId = 1;
    
    ... ... ...
    
    // 这个很重要，window.WebViewJavascriptBridge 是 H5 和 Native 通信的
    // 关键点！
    var WebViewJavascriptBridge = window.WebViewJavascriptBridge = {
        init: init,
        send: send,
        registerHandler: registerHandler,
        callHandler: callHandler,
        _fetchQueue: _fetchQueue,
        _handleMessageFromNative: _handleMessageFromNative
    };
```

上面最关键的一个对象就是 WebViewJavascriptBridge，H5 和 Native 都会通过它。

这个 window.WebViewJavascriptBridge 内部包含了一些函数对象，这些 function 都定义在 js 内部！


## 2.2 动态注入初始化

这里是很关键的地方：

```javascript
    var doc = document;
    //【1】创建消息队列，一个是 index，一个是消息体；
    _createQueueReadyIframe(doc);
    _createQueueReadyIframe4biz(doc);
    //【2】创建一个 event，类型为 'WebViewJavascriptBridgeReady'
    // 然后分发 event；
    var readyEvent = doc.createEvent('Events');
    readyEvent.initEvent('WebViewJavascriptBridgeReady');
    readyEvent.bridge = WebViewJavascriptBridge;
    doc.dispatchEvent(readyEvent); //【*2.2.1】关键点！！
```
在动态注入的时候，会执行初始化的操作：

- 创建了一个 event；
- 初始化 event，事件类型为 'WebViewJavascriptBridgeReady'；
- readyEvent.bridge 设置为我们上面创建的 'WebViewJavascriptBridgeReady' 对象；
- doc.dispatchEvent 分发 event；

<br/>

**这个 event 是在哪里做响应**的呢？

是在 H5 里面，这个 H5 在加载时候，会执行内部 js 脚本，并通过 document.addEventListener 方法设置该 event 的监听器；

### 2.2.1 H5 加载启动 event 监听

H5 的页面里面，是有下面的一段 js 脚本，在 webview.loadUrl 后会直接加载该 js：

```javascript
<script>
... ... ...
function connectWebViewJavascriptBridge(callback) {
    if (window.WebViewJavascriptBridge) {
        //【2】如果 window.WebViewJavascriptBridge 已经存在
        // 直接执行函数闭包；
        callback(WebViewJavascriptBridge)
    } else {
        //【3】否则我们就注册一个 EventListener，监听 WebViewJavascriptBridgeReady 事件；
        document.addEventListener(
            'WebViewJavascriptBridgeReady'
            , function() {
                // 事件出发后，执行函数闭包；
                callback(WebViewJavascriptBridge)
            },
            false
        );
    }
}
//【1】执行 connectWebViewJavascriptBridge 方法，传入了一个 js 闭包;
connectWebViewJavascriptBridge(function(bridge) {
     //【*2.2.2】下一步初始化！
     ... ... ... ...
})
</script>
```
默认情况下，window.WebViewJavascriptBridge 不存在，那么会注册一个 EventListener！

等待 event 触发后，执行 callback！


### 2.2.2 event 出发点后下一步初始化

callback 实际上就是闭包，参数 bridge 就是 js 协议中创建的 var WebViewJavascriptBridge：

```javascript
    //【*2.3.1】执行 WebViewJavascriptBridge 对象的 init 方法，
    // 传入一个函数闭包！
    bridge.init(function(message, responseCallback) {
        console.log('JS got a message', message);
        var data = {
            'Javascript Responds': '测试中文!'
        };

        if (responseCallback) {
            console.log('JS responding with', data);
            //【1】函数闭包出发后，会回调 responseCallback
            responseCallback(data);
        }
    });

    //【*2.3.1】执行 WebViewJavascriptBridge 对象的 registerHandler 方法，
    // 传入一个函数闭包！
    bridge.registerHandler("functionInJs", function(data, responseCallback) {
        document.getElementById("show").innerHTML = ("data from Java: = " + data);
        if (responseCallback) {
            var responseData = "Javascript Says Right back aka!";
            //【2】函数闭包出发后，会回调 responseCallback，通知 native；
            responseCallback(responseData);
        }
    });
```
关于 init 和 registerHandler 我们会在下面分析：

## 2.3 核心函数

下面来分析下关键的协议函数：

### 2.3.1 init 

init 方法用于设置 **js 处理 native 消息的默认 handler**：

同时也会**分发已经被添加到 receiveMessageQueue 接受队列中的 native 的消息**：

```javascript
    function init(messageHandler) {
        if (WebViewJavascriptBridge._messageHandler) {
            throw new Error('WebViewJavascriptBridge.init called twice');
        }
        //【1】设置 js 用于处理 native 消息的 handler
        // 实际上就是【*2.2.2】中的函数闭包；
        WebViewJavascriptBridge._messageHandler = messageHandler;
        //【2】分发已经被添加到 receiveMessageQueue 接受队列中的 native 的消息
        var receivedMessages = receiveMessageQueue;
        receiveMessageQueue = null;
        for (var i = 0; i < receivedMessages.length; i++) {
            //【*2.3.3】分发来自 native 的消息；
            _dispatchMessageFromNative(receivedMessages[i]);
        }
    }
```
参数 messageHandler 就是【*2.2.2】中的函数闭包；

### 2.3.2 registerHandler 

**注册特定的消息处理 handler**：

```javascript
    function registerHandler(handlerName, handler) {
        //【1】实际上就是向数组中放值；
        messageHandlers[handlerName] = handler;
    }
```
messageHandlers 之前有说过，是 js 处理 native 消息的 handler 数组！

- index 是 handler 的名称，根据前面代码，名称是 "functionInJs"；
- value 是一个函数闭包；

### 2.3.3 _dispatchMessageFromNative 

这个方法是 **js 层**调用的，**分发来自 native 的消息**：

```javascript
function _dispatchMessageFromNative(messageJSON) {
    setTimeout(function() {
        //【2】JSON 字符串转化 JSON 对象 message；
        var message = JSON.parse(messageJSON);
        var responseCallback;
        //【2】这里我们知道 native 发送消息完成，接下来 js 会处理消息，并将结果
        // 通过 callback 传递给 native 层；
        if (message.responseId) {
            //【2.1】如果 native 指定了消息的 responseId，这种情况对应的情况是：
            // js 发送消息给 native，此时 native 发送回调消息给 js；
            // 那么我们就要在 responseCallback 数组中找到对应的 responseCallback
            responseCallback = responseCallbacks[message.responseId];
            if (!responseCallback) {
                return;
            }
            //【2.2】然后执行 js 的 callback；
            responseCallback(message.responseData);
            //【2.3】删掉该 callback
            delete responseCallbacks[message.responseId];
        } else {
            //【2.4】没有指定 responseId，但是指定了 callbackId，这种情况对应的是：
            // native 发送消息给 js，此时 js 发送回调消息给 native；
            if (message.callbackId) {
                //【2.4.1】获得 callbackId，并创建一个 responseCallback
                // 实际上就是一个函数闭包，该闭包会执行 _doSend 方法！
                var callbackResponseId = message.callbackId;
                responseCallback = function(responseData) {
                    //【*2.3.4】发送回调给 native，但是此时是不触发的，出发的点在下面；
                    _doSend({
                        responseId: callbackResponseId,
                        responseData: responseData
                    });
                };
            }
            //【2.5】找到处理 native 消息的 handler，如果没有指定 handlerName
            // 那么就是 init 方法注册的默认 handler；否则就是特定的 handler
            // 其实就是前面 "functionInJs" 对应的 handler；
            var handler = WebViewJavascriptBridge._messageHandler;
            if (message.handlerName) {
                handler = messageHandlers[message.handlerName];
            }
            //【2.6】这个 handler 其实就是一个函数闭包，见【*2.2.2】，最后会回调
            // responseCallback 接口，就是上面的 function；
            try {
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
到这里看起来，似乎很清晰呢；

### 2.3.4 _doSend

这个方法是 **js 层**调用，用于**发送消息给 native 端**：

```javascript
    function _doSend(message, responseCallback) {
        if (responseCallback) {
            //【1】计算回调 id；
            var callbackId = 'cb_' + (uniqueId++) + '_' + new Date().getTime();
            //【2】以 index 为回调 id，value 为具体的回调接口的形式，保存到 responseCallbacks 数组重；
            responseCallbacks[callbackId] = responseCallback;
            //【3】message.callbackId 为计算出的回调 id；
            message.callbackId = callbackId;
        }
        //【4】将该 message 加入到 sendMessageQueue 队列中，要发送给 native；
        sendMessageQueue.push(message);
        //【6】这个地方会通过 messagingIframe.src 生成一个 Url，这会被 Webview.shouldOverrideUrlLoading 拦截到；
        messagingIframe.src = CUSTOM_PROTOCOL_SCHEME + '://' + QUEUE_HAS_MESSAGE;
    }
```
这里要注意第二个参数 responseCallback：

- 如果 responseCallback 不为 null，说明本次消息需要回调通知；
- 如果 responseCallback 为 null，说明不需要回调通知；

该方法创建了一个动态的 url，这会被 Webview.shouldOverrideUrlLoading 拦截到，这是该库 android 获得 js 数据的方式；

但是这里并不是真正获取数据的地方，该 url 会触发一次 Webview.shouldOverrideUrlLoading；

然后 android 又会调用 js 的 _fetchQueue 方法，这时，又会生成一个 url，这个 url 才会保存了要传递给 android 的消息；

具体可以看 2.3.8 的 _fetchQueue 方法；


### 2.3.5 callHandler

这个方法是 **js 层**调用的，**通过这个接口来调用 native 方法**：

- **handlerName**：js 处理消息的 handler 名称，这个 handler 是 **native 层注册到 js 的**；
- **data**：native 层传递的数据；
- **responseCallback**：接受回调的接口，native 层处理完数据会回调；

```javascript
    function callHandler(handlerName, data, responseCallback) {
        //【*2.3.4】调用 _doSend 方法发送消息给 js，注意这里第二个参数
        // 不为 null，因为 js 短需要收到回调；
        _doSend({
            handlerName: handlerName,
            data: data
        }, responseCallback);
    }
```
这里第二个参数不为 null，因为 js 短需要收到回调；

该方法设置 handlerName，所以 native 会使用指定 handlerName 的 handler 去处理；


### 2.3.6 send

这个方法也是 **js 层**调用的，**通过这个接口来调用 native 方法**：

```javascript
    // 发送
    function send(data, responseCallback) {
        _doSend({
            data: data
        }, responseCallback);
    }
```
这里我们看到，他并没有设置 handlerName，所以 native 会使用默认的 handler 去处理；


### 2.3.7 _handleMessageFromNative

这个方法是 **native 层**调用的，**以 json string 的形式发送数据给 js**：

```javascript
function _handleMessageFromNative(messageJSON) {
    console.log(messageJSON);
    //【1】如果 receiveMessageQueue 不为 null，那就直接添加到 receiveMessageQueue 队列中去；
    if (receiveMessageQueue) {
        receiveMessageQueue.push(messageJSON);
    }
    //【*2.3.3】处理 native 传递的数据；
    _dispatchMessageFromNative(messageJSON);
}
```
这里很简单，就不多说了；

### 2.3.8 _fetchQueue

这个方法是 **native 层**调用的，**用于获取 sendMessageQueue 队列中的消息**：

```javascript
function _fetchQueue() {
    //【1】这里是统一处理要发给 native 的所有消息，将队列转为 string
    var messageQueueString = JSON.stringify(sendMessageQueue);
    sendMessageQueue = [];
    //【2】这里又再在页面生成 url，和之前的区别是包含数据，然后通过 shouldOverrideUrlLoading 方法拦截，捕获 url 中的数据；
    if (messageQueueString !== '[]') {
        bizMessagingIframe.src = CUSTOM_PROTOCOL_SCHEME + '://return/_fetchQueue/' + encodeURIComponent(messageQueueString);
    }
}
```
逻辑很简单，不多说了，关于 H5 和 Native 通信的流程，后续再分析！

# 3 总结

关于 js 通信协议的相关分析到这里就结束了。

这里我自己也有点疑惑，对于 android 获取 js 数据的方式，该库并没有使用 @JavascriptInterface 注解，通过如下方式实现：

```java
WebView.addJavascriptInterface(new WebData(), "webdata");
```
通过查阅相关资料，可能有如下的原因：

- **安全隐患**：这是因为同源规则 (SOP) 不适用与该方法，加上第三方 JavaScript 库或来自一个陌生域名的 iframe 可能在 Java 层访问这些被暴露的方法。因此，攻击者可通过一个 XSS 漏洞执行原生代码或者注入病毒代码到应用程序中。
- **兼容性**：JavaScript 层中暴露的 Java 对象的所有公有方法在 Android 版本低于 JerryBean MRI(API Level 17) 以下时可访问。而在 Google API 17 （4.２）以上，暴露的函数必须通过 @JavaScriptInterface 注释来防止方法的暴露

