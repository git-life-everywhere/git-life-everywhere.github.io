# JsBridge 第一篇 - 框架整体结构和基本使用
title: JsBridge 第一篇 - 框架整体结构和基本使用
date: 2019/05/20 20:46:25
categories: 
- 开源库源码分析
- JsBridge
tags: JsBridge
---

本系列文章主要分析 JsBridge 框架的原理，学习 H5 和原生 WebView 的交互方式，框架选自 GitHub 上的很火的 H5 + WebView 三方库：lzyzsd/JsBridge，作者是大鬼头；

# 1 工程结构

我们先来看看 lib 的主要文件结构：

```java
|____src
| |____main
| | |____java
| | | |____com
| | | | |____github
| | | | | |____lzyzsd
| | | | | | |____jsbridge
| | | | | | | |____Message.java
| | | | | | | |____WebViewJavascriptBridge.java
| | | | | | | |____DefaultHandler.java
| | | | | | | |____BridgeWebView.java
| | | | | | | |____BridgeWebViewClient.java
| | | | | | | |____CallBackFunction.java
| | | | | | | |____BridgeHandler.java
| | | | | | | |____BridgeUtil.java
| | |____assets
| | | |____WebViewJavascriptBridge.js
```

可以看到，核心的代码在 asserts 和 jsbridge 目录下。

- js 文件是是通信的协议，H5 会出发 js 脚本中的语句，然后把通信的数据传递给 WebView；
- jsbridge 目录下的 .java 文件则是和 Native 层的逻辑相关；

这里先不详细分析每个文件的具体逻辑，我们后面分析交互流程的时候会讲；

# 2 基本使用

具体的使用主要分为如下几个部分，我会将 Demo 中的一些重点代码块，用注视标注出来：

## 2.1 初始化 BridgeWebView：

```java
webView = (BridgeWebView) findViewById(R.id.webView);
button = (Button) findViewById(R.id.button);
button.setOnClickListener(this);

//【1】设置默认的消息处理回调；
webView.setDefaultHandler(new DefaultHandler());
//【2】设置 WebChromeClient 对象，无关不分析；
webView.setWebChromeClient(new WebChromeClient() {
    ... ... ...
});
```
当然，这里的 WebChromeClient 其实没有太大的用处，而真正有用的是其内部的：WebViewClient 对象，这个我们后面单独去分析 BridgeWebView 的时候，就知道 WebViewClient 的具体逻辑了；

## 2.2 注册回调 Handler 到 webview 中：

```java
//【1】加载 H5 页面；
webView.loadUrl("file:///android_asset/demo.html");
//【2】注册了一个 BridgeHandler 回调对象，用于处理 js 的消息并回调通知；
webView.registerHandler("submitFromWeb", new BridgeHandler() {
	@Override
	public void handler(String data, CallBackFunction function) {
		Log.i(TAG, "handler = submitFromWeb, data from web = " + data);
        function.onCallBack("submitFromWeb exe, response data 中文 from Java");
	}
});
```
BridgeHandler 是一个接口，面向接口编程，前面的 DefaultHandler 实现了这个接口！

可以看到 BridgeHandler 是用于处理 H5 发送给 Native 的消息的；

而 CallBackFunction 则是用于回调结果给 H5；

## 2.3 Native 向 H5 发送消息，并接受回调

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
可以看到，上面给出了有回调和没有回调的两种通信方式；

具体的调用逻辑，我们后面再分析！

## 2.4 H5 向 Native 发送消息，并接受回调

这个地方就比较复杂了，我们要从 H5 中看起；

- H5 触发 js 的函数，指定具体的 handler 处理：

```javascript
function testClick1() {
    var str1 = document.getElementById("text1").value;
    var str2 = document.getElementById("text2").value;

    //【1】调用本地方法，特定 handler 处理！
    window.WebViewJavascriptBridge.callHandler(
        'submitFromWeb'
        , {'param': '中文测试'}
        , function(responseData) {
            document.getElementById("show").innerHTML = "send get responseData from java, data = " + responseData
        }
    );
}
```
这里看到了 'submitFromWeb'，这和前面的 registerHandler 相呼应了！

- H5 触发 js 的函数，默认 handler 处理：

```javascript
function testClick() {
    var str1 = document.getElementById("text1").value;
    var str2 = document.getElementById("text2").value;

    //【1】调用本地方法，默认 handler 处理！
    var data = {id: 1, content: "这是一个图片 <img src=\"a.png\"/> test\r\nhahaha"};
    window.WebViewJavascriptBridge.send(
        data
        , function(responseData) {
            document.getElementById("show").innerHTML = "repsonseData from java, data = " + responseData
        }
    );
}
```
这和前面的 DefaultHandler 相呼应了！
