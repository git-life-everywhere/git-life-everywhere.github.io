# Shadow 原理分析

title: Shadow 原理分析
date: 2022/10/13 20:46:25 
categories: 

- 插件化
- Shadow
tags: Shadow
copyright: true
---

上周在团队内部做了一次技术分享，关于 shadow 核心技术原理，下面是这次分享的内容：

我这里也提供了 pdf 的百度云地址，微信扫一扫就好～～

![](https://coolqifiles.oss-cn-hangzhou.aliyuncs.com/uPic2/2022/10/24/05/1112.png) 

# 原理总览

<img src="https://coolqifiles.oss-cn-hangzhou.aliyuncs.com/uPic2/2022/10/24/03/19e559177000a7db6424f781bd7da2ef.png" width="70%" />

# 工程结构

<img src="https://coolqifiles.oss-cn-hangzhou.aliyuncs.com/uPic2/2022/10/24/03/e81a4a6afaec51f2a6371398e3a9590e123.png" width="70%"  />

# 接口动态化原理



<img src="https://coolqifiles.oss-cn-hangzhou.aliyuncs.com/uPic2/2022/10/24/03/1eaff16b6499abc0d1278411062335d1.png" width="70%" />

# 插件包的组成

<img src="https://coolqifiles.oss-cn-hangzhou.aliyuncs.com/uPic2/2022/10/24/04/image-20221024042017313.png" width="70%"  />

<img src="https://coolqifiles.oss-cn-hangzhou.aliyuncs.com/uPic2/2022/10/24/03/6d60ff0bd75bfa6398cbb24c1ef02ab5.png" width="70%"  />

# 宿主如何启动插件 Activity

<img src="https://coolqifiles.oss-cn-hangzhou.aliyuncs.com/uPic2/2022/10/24/03/1ad330e14208feae1bdddae0acf01f18.png" width="70%"  />

<img src="https://coolqifiles.oss-cn-hangzhou.aliyuncs.com/uPic2/2022/10/24/03/8e20f889f6bcbcf0da8f7feedabe7728.png" width="70%"  />

<img src="https://coolqifiles.oss-cn-hangzhou.aliyuncs.com/uPic2/2022/10/24/03/fb607615626d95b232ddaef8a17fcacf.png" width="70%"  />

<img src="https://coolqifiles.oss-cn-hangzhou.aliyuncs.com/uPic2/2022/10/24/03/4ddb06df7f1b71d1a0c5e89ba5e69619.png" width="70%"  />

<img src="https://coolqifiles.oss-cn-hangzhou.aliyuncs.com/uPic2/2022/10/24/03/dff534d7bf719e3d16433363b60c7703.png" width="70%"  />

<img src="https://coolqifiles.oss-cn-hangzhou.aliyuncs.com/uPic2/2022/10/24/03/c2958941fb6154cccc247e59c8f34056.png" width="70%"  />

<img src="https://coolqifiles.oss-cn-hangzhou.aliyuncs.com/uPic2/2022/10/24/03/87a07583f881eb55a7f90a475e898a1e.png" width="70%"  />

# PPS 的作用

<img src="https://coolqifiles.oss-cn-hangzhou.aliyuncs.com/uPic2/2022/10/24/03/37d98fc1f32655ce51909c1993c21f19.png" width="70%"  />

# 插件独立进程的好处



<img src="https://coolqifiles.oss-cn-hangzhou.aliyuncs.com/uPic2/2022/10/24/03/0dda4f65105ab27bb292ecf3918f970c.png" width="70%"  />

#  同名 **View** 问题和解决方案

<img src="https://coolqifiles.oss-cn-hangzhou.aliyuncs.com/uPic2/2022/10/24/03/914a74508b0fd7a2551ef79bf89c34f9.png" width="70%"  />

## 传统解决方案

<img src="https://coolqifiles.oss-cn-hangzhou.aliyuncs.com/uPic2/2022/10/24/03/5faca70d15065311d4d001e2c7cda4ce.png" width="70%"  />

## Factory 注入

<img src="https://coolqifiles.oss-cn-hangzhou.aliyuncs.com/uPic2/2022/10/24/03/8f738b7ac1d781bb3d46af1bf02febee.png" width="70%"  />

com.tencent.shadow.core.runtime.ShadowLayoutInflater/InnerInflater 

com.tencent.shadow.core.runtime.ShadowFactory2



<img src="https://coolqifiles.oss-cn-hangzhou.aliyuncs.com/uPic2/2022/10/24/03/ad9dc342c2f3081de329690b67236e3a.jpeg" width="70%"  />

<img src="https://coolqifiles.oss-cn-hangzhou.aliyuncs.com/uPic2/2022/10/24/03/3dca7657fdb73098a4096c5f9df157a3-1234.jpeg" width="70%"  />
ShadowFactory2  ShadowLayoutInflater

<img src="https://coolqifiles.oss-cn-hangzhou.aliyuncs.com/uPic2/2022/10/24/03/2619db515edbd3b4c16c1020aa4cfb4d.jpeg" width="70%"  />

LayoutInflater

## LayoutInflater 的 Merge



<img src="https://coolqifiles.oss-cn-hangzhou.aliyuncs.com/uPic2/2022/10/24/03/70166414fd349723eda83b4cfb37e4ed.png" width="70%"  />

# 插件 PluginClassLoader 浅析

<img src="https://coolqifiles.oss-cn-hangzhou.aliyuncs.com/uPic2/2022/10/24/03/095604ba7367ff4aca77402da896a0d4.png" width="70%"  />

<img src="https://coolqifiles.oss-cn-hangzhou.aliyuncs.com/uPic2/2022/10/24/05/image-20221024055237183.png" width="70%"  />

# 插件包名和宿主包名



<img src="https://coolqifiles.oss-cn-hangzhou.aliyuncs.com/uPic2/2022/10/24/03/87feec32ede63fdc3fa4b48550a51957.jpeg" width="70%"  />

# 插件包的资源



<img src="https://coolqifiles.oss-cn-hangzhou.aliyuncs.com/uPic2/2022/10/24/03/593ead7257a5ec97b4663d1f63e0a76e.jpeg" width="70%"  />

# ShadowContext 代理

<img src="https://coolqifiles.oss-cn-hangzhou.aliyuncs.com/uPic2/2022/10/24/03/309885dc6166b26379bcdaef422d8c76-1234.jpeg" width="70%"  />

ShadowActivity、ShadowApplication 的父类都是 ShadowContext

# 宿主如何启动插件 Service

<img src="https://coolqifiles.oss-cn-hangzhou.aliyuncs.com/uPic2/2022/10/24/03/61121db483849a08f069316ea70cd3fc.png" width="70%"  />

<img src="https://coolqifiles.oss-cn-hangzhou.aliyuncs.com/uPic2/2022/10/24/03/e67cb88c01ec821585d695539f76fc16.png" width="70%"  />

<img src="https://coolqifiles.oss-cn-hangzhou.aliyuncs.com/uPic2/2022/10/24/03/be12ef34cd7eaba1b9e9b31442585da8.png" width="70%"   />

<img src="https://coolqifiles.oss-cn-hangzhou.aliyuncs.com/uPic2/2022/10/24/04/9c0730de7472524cc8a82ff4f1de5eb7-1234.jpeg" width="70%"  />

# 宿主如何启动和访问插件 provider



<img src="https://coolqifiles.oss-cn-hangzhou.aliyuncs.com/uPic2/2022/10/24/04/94cf71f07a736b41e1562cbd6ee662fa.png" width="70%"  />

<img src="https://coolqifiles.oss-cn-hangzhou.aliyuncs.com/uPic2/2022/10/24/04/4332787979cd2014ef823657c81b81e9.png" width="70%"  />

<img src="https://coolqifiles.oss-cn-hangzhou.aliyuncs.com/uPic2/2022/10/24/04/6dcb6e2bca42ba4c3ef96282fea68f7b.png" width="70%"  />

<img src="https://coolqifiles.oss-cn-hangzhou.aliyuncs.com/uPic2/2022/10/24/04/e5b571aec8e013c103836e64377a15e5.png" width="70%"  />



# 宿主如何启动和访问插件 receiver



<img src="https://coolqifiles.oss-cn-hangzhou.aliyuncs.com/uPic2/2022/10/24/04/512ebd907d835af5bd0e3e79e105cf71.png" width="70%"  />


# Gradle Plugin 简析-配置阶段

<img src="https://coolqifiles.oss-cn-hangzhou.aliyuncs.com/uPic2/2022/10/24/04/d1820c195a73b212927273db1a01a957.jpeg" width="70%"  />

<img src="https://coolqifiles.oss-cn-hangzhou.aliyuncs.com/uPic2/2022/10/24/04/7e535b08dffc9e4cc049833399ad27ac.jpeg" width="70%"  />

<img src="https://coolqifiles.oss-cn-hangzhou.aliyuncs.com/uPic2/2022/10/24/04/c4d52eff69590f1d8f6cd05e747f2352.jpeg" width="70%"  />


# Gradle Plugin 简析-Task


![](https://coolqifiles.oss-cn-hangzhou.aliyuncs.com/uPic2/2022/10/24/04/3a4dc470f1c202bbb1a31ed755a26833.jpeg)

<img src="https://coolqifiles.oss-cn-hangzhou.aliyuncs.com/uPic2/2022/10/24/04/eaf1bd949d5134e76ee0475d1aa6de04.png" width="70%"  />



<img src="https://coolqifiles.oss-cn-hangzhou.aliyuncs.com/uPic2/2022/10/24/04/image-20221024042430828.png" width="70%"  />



![](https://coolqifiles.oss-cn-hangzhou.aliyuncs.com/uPic2/2022/10/24/04/4de26cc8cb37e6fd7d18ed1f688aa691.jpeg)



# Gradle Plugin 简析- Transformer



<img src="https://coolqifiles.oss-cn-hangzhou.aliyuncs.com/uPic2/2022/10/24/04/image-20221024041343449.png" width="70%"  /> 

![](https://coolqifiles.oss-cn-hangzhou.aliyuncs.com/uPic2/2022/10/24/04/image-20221024041417834.png)



<img src="https://coolqifiles.oss-cn-hangzhou.aliyuncs.com/uPic2/2022/10/24/04/59a6618b11def831941c87d646cea777.jpeg" width="70%"  />

# 简单的 SpecificTransform

<img src="https://coolqifiles.oss-cn-hangzhou.aliyuncs.com/uPic2/2022/10/24/04/88801d524e3219153bc6751a2079eeb2.jpeg" width="70%"  />

<img src="https://coolqifiles.oss-cn-hangzhou.aliyuncs.com/uPic2/2022/10/24/04/1cc0bd8a5123dbcdd72f11e7fcb2fa4c.jpeg" width="70%"  />

<img src="https://coolqifiles.oss-cn-hangzhou.aliyuncs.com/uPic2/2022/10/24/04/9b27a8746ce6e32ecfbe94a45bc14e15.jpeg" width="70%"  />



![](https://coolqifiles.oss-cn-hangzhou.aliyuncs.com/uPic2/2022/10/24/04/image-20221024041510698.png)

![](https://coolqifiles.oss-cn-hangzhou.aliyuncs.com/uPic2/2022/10/24/04/image-20221024041520343.png)

# 复杂的 SpecificTransform

## ContentProviderTransform

<img src="https://coolqifiles.oss-cn-hangzhou.aliyuncs.com/uPic2/2022/10/24/04/image-20221024041601332.png" width="70%"  />

<img src="https://coolqifiles.oss-cn-hangzhou.aliyuncs.com/uPic2/2022/10/24/04/image-20221024041707286.png" style="zoom: 67%;" />

<img src="https://coolqifiles.oss-cn-hangzhou.aliyuncs.com/uPic2/2022/10/24/04/7d4a91de71034adc326dfa8b36041474.jpeg" style="zoom: 67%;" />

<img src="https://coolqifiles.oss-cn-hangzhou.aliyuncs.com/uPic2/2022/10/24/04/dbc54d03d18165607adc639e7b29f046.jpeg" style="zoom: 67%;" />



## DialogSupportTransform

<img src="https://coolqifiles.oss-cn-hangzhou.aliyuncs.com/uPic2/2022/10/24/04/image-20221024041755647.png" width="70%"  />

<img src="https://coolqifiles.oss-cn-hangzhou.aliyuncs.com/uPic2/2022/10/24/04/25c486e9ea14d0b468a237986f19cbf8.jpeg" style="zoom: 67%;" />



## LayoutInflaterTransform

<img src="https://coolqifiles.oss-cn-hangzhou.aliyuncs.com/uPic2/2022/10/24/04/9b0e6d2485eafcd403d955116472b14f.png" width="70%"  />

<img src="https://coolqifiles.oss-cn-hangzhou.aliyuncs.com/uPic2/2022/10/24/04/51c6a2de8e34b18fb11f17c498781b7f.jpeg" width="70%"  />  

<img src="https://coolqifiles.oss-cn-hangzhou.aliyuncs.com/uPic2/2022/10/24/04/b8dd3837c60bd23809ee18a496357000.jpeg" width="70%" style="zoom:50%;" />

## PackageManagerTransform

<img src="https://coolqifiles.oss-cn-hangzhou.aliyuncs.com/uPic2/2022/10/24/04/a1889e74e623a0e46bc3365ca92a8bc6.png" width="70%"  />

![](https://coolqifiles.oss-cn-hangzhou.aliyuncs.com/uPic2/2022/10/24/04/d7ab9e13fc701548536b9d5ce69c3184.jpeg)

![](https://coolqifiles.oss-cn-hangzhou.aliyuncs.com/uPic2/2022/10/24/04/image-20221024041930534.png) 

<img src="https://coolqifiles.oss-cn-hangzhou.aliyuncs.com/uPic2/2022/10/24/04/f5c2b3edbe7189acea056ca80bb0fa07.jpeg" width="70%"  />

# 参考文献


Shadow 官网https://github.com/Tencent/Shadow

Shadow 作者本人的博客： https://juejin.cn/user/536217405890903
