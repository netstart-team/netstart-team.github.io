---
layout:     post
title:      "WebKit.framework 简明教程"
subtitle:   "WebKit.framework 简明教程"
date:       2015-09-22
author:     "leon"
header-img: "img/post-bg-js-version.jpg"
tags:
    - WKPreferences
---

# WebKit.framework 简明教程

**注意：这不是一份完整的API手册，只是帮助大家快速入门的简明教程。完整的手册请参考 [WebKit Framework Reference](https://developer.apple.com/library/prerelease/ios/documentation/Cocoa/Reference/WebKit/ObjC_classic/index.html)**。示例代码请参考[Leon's Github](https://github.com/Leon1108/StudyWebKitFramework)。


## 概述
在iOS 8中Apple提供了一个新的WebView框架 **WebKit.framework** 相较于UIKit中得UIWebView。
* WKWebView使用了与Safari相同的JS引擎，使App中使用的WebView具有与Safari同样的性能表现。
* 提供了更清晰的API结构，提供了更好的与Web页面的交互接口。

**注意：该框架并非线程安全的，所以调用该框架的方法或函数时，必须在主线程中调用。**

#### 目前的主要问题：
* 没有提供直接加载HTML Data（或String）的方法。
  * 目前想到只能先保存成文件，然后再通过File URL加载。
* 没有像UIWebView的``那样，直接执行Javascript的方法。
  * 目前来看，只能通过[WKUserContentController](#wkusercontentcontroller)来添加一个[WKUserScript](#wkuserscript)，然后需要 reload [WKWebView](#wkwebview)。

## API 流水账
### WKWebView
该框架中唯一的UI组件，实现Web页面的展现以及交互。

需要通过一个 [WKWebViewConfiguration](#wkwebviewconfiguration) 进行初始化，其中封装了用于初始化WKWebView所需要的必要参数。

```
var webView = WKWebView(frame:CGRectMake(0, 40, 320, 320),
                configuration:config)
```

#### 回调函数（Delegate）
在生命周期事件方面 WKWebView 提供了两个委派接口（Protocol）

* UIDelegate ([WKUIDelegate](https://developer.apple.com/library/prerelease/ios/documentation/WebKit/Reference/WKUIDelegate_Ref/index.html))
  * 提供了让Web页面呈现原生界面的方法，主要包含3个回调方法，分别对应 Javascript的 `alert()`, `confirm()`, `prompt()`函数。
  * 一些相关的类:
     * [WKFrameInfo](https://developer.apple.com/library/prerelease/ios/documentation/WebKit/Reference/WKFrameInfo_Ref/index.html)
         * 封装了Frame的信息。在API中描述，是由哪个Frame发起的这个调用。
* *navigationDelegate* ([WKNavigationDelegate](https://developer.apple.com/library/prerelease/ios/documentation/WebKit/Reference/WKNavigationDelegate_Ref/index.html))
  * 提供了追踪页面主框架（main frame）加载进程回调方法，以及决定主框架或者子框架加载策略的方法。
  * 如上所属，该Protocol中包含了两大类回调方法。跟踪主框架的加载过程，以及决定框架（包括子框架）的加载策略。按照调用发生的顺序，描述如下：

No.| Call Back Method                                  | Type | Description
---|---------------------------------------------------|------|------------------------------------------------------------------------------------------
1  | decidePolicyForNavigationAction:decisionHandler:  |Policy| 决定是否允许或取消一次Navigation. `对于自定义Schema，就需要在这里拦截了`
2  | didStartProvisionalNavigation:                    |Nav.  | 当主框架开始加载时，调用该方法。
3  | didReceiveServerRedirectForProvisionalNavigation: |Nav.  | 当主框架收到Server端返回的从定向请求时
4  | decidePolicyForNavigationResponse:decisionHandler:|Policy| 在获得了一次Navigation的应答后，决定是否继续。（读取完HTTP头？？）
5  | didFailProvisionalNavigation:withError:           |Nav.  | 当主框架开始加载数据期间发生错误时，比如在 `decidePolicyForNavigationResponse:` 回调中，取消了后续的处理。
6  | didCommitNavigation:                              |Nav.  | 当开始接收到主框架的内容时，调用此方法。
7  | didFailNavigation:withError:                      |Nav.  | 当已经进入Committed 状态的主框架加载中发生错误时，调用此方法。
8  | didFinishNavigation:                              |Nav.  | 当主框架加载完成时，调用此方法。

为了更好理解回调的过程，画了下面这张图，仅供参考。（没有找到Apple官方的图）

![](res/WebViewNavigationDelegate.png)


#### 相关的类

* [WKNavigation](https://developer.apple.com/library/prerelease/ios/documentation/WebKit/Reference/WKNavigation_Ref/index.html)
	* 唯一标识一次Web页面的加载过程。由WKWebView的loadRequest方法返回，并在navigation的回调方法中传回。
	* 包含 `initialRequest`, `request`, `response`, `error` 四个属性。
* [WKNavigationAction](https://developer.apple.com/library/prerelease/ios/documentation/WebKit/Reference/WKNavigationAction_Ref/index.html)
	* 封装了引发这次Navigation的行动（Action)
	* 可能的Action包括: `LinkActivated`, `FormSubmitted`, `BackForward`, `Reload`, `FormResubmitted` 以及 `Other`。封装在 **navigationType** 属性中。
* [WKNavigationResponse](https://developer.apple.com/library/prerelease/ios/documentation/WebKit/Reference/WKNavigationResponse_Ref/index.html)
	* 封装了导航（navigation）的应答内容。
	* 包含 `canShowMIMEType`, `forMainFrame`, `response` 三个属性 


### WKWebViewConfiguration
该类中封装了用于初始化WKWebView所需要的参数，主要包括：

* [WKPreferences](#wkpreferences)
  * WebView的偏好设置
* [WKProcessPool](#wkprocesspool) 
  * 获得该Web内容处理进程的进程池（process？处理池？）
* [WKUserContentController](#wkusercontentcontroller)
  * 提供了使用Javascript与Web内容进行交互（互相调用）的接口。



### WKPreferences
包含WebView的偏好设置。

#### 关于DefaultsKeyPrefix: `TODO`

#### 主要的设置包括

* minimumFontSize
  * 默认为 `0`
* suppressesIncrementalRendering
  * 默认为 `NO`，表示不需要等内容全部加载到内存中后（也就是不需要全部下载完，能渲染多少就渲染多少），才开始渲染页面。 
  * `YES` 则表示必须等数据全部下载完，才开始渲染页面。
* javaScriptCanOpenWindowsAutomatically 
  * Javascript 是否可以自动打开新窗口（也就是不需要用户的交互）。iOS 默认是 `NO` 
  * `YES` 表示允许自动弹出新窗口。
* javaScriptEnabled
  * 默认为 `YES`
* allowsInlineMediaPlayback
  * 默认为 `NO`，表示使用本地播放器全屏播放。 
  * `YES` 则表示可以再HTML5页面中直接播放。 `TODO Audio呢？`
* mediaPlaybackAllowsAirPlay
  * 默认为 `YES`
* mediaPlaybackRequiresUserAction
  * 默认为 `YES`，表示需要用户来主动开启 HTML5 Video的播放。 
  * `NO` 则表示可以自动播放。`TODO Audio呢？`



### WKProcessPool
`TODO` 还没研究到底干嘛用

### WKUserContentController
该类提供了一组与Web内容之间传递消息的接口，也就是Web Path与Native App之间通过Javascript相互通讯全靠他了。

* 接收来自Web页面的消息
  1. 若想接收来自Web页面的消息，则需要实现 [WKScriptMessageHandler](#wkscriptmessagehandler) Protocol。
  2. 然后将Handler的实现注册到UserContentController中。
  3. 在HTML页面中，通过调用下面的函数来想已注册的Handler来Post消息。
```
window.webkit.messageHandlers.[handlerName].postMessage(messageBody)
```
     其中的 `handlerName` 是注册Handler时的name。 `messageBody`是消息体，可以是多种Javascript类型，会映射到对应的OC类型 NSNumber, NSString, NSDate, NSArray, NSDictionary, and NSNull。

* 向Web页面中注入Javascript
  1. 通过 `addUserScript`方法来向Web页面添加一段Javascript，或调用一个Javascript函数。
  2. 关于 [WKUserScript](#wkuserscript)


### *WKScriptMessageHandler* (Protocol)
实现了该Protocol的类，将会提供一个能够从Web页面中接收Javascript所发出的消息的方法。

该Protocol只包含一个必须实现的方法：

```
- (void)userContentController:(WKUserContentController *)userContentController
      didReceiveScriptMessage:(WKScriptMessage *)message
```
参数说明：

Param                   | Description
------------------------|---------------------------------------------
`userContentController` | 调用此委托方法的WKUserContentController实例。
`message`               | 接收到的消息。是一个 [WKScriptMessage](wkscriptmessage) 的实例。


### WKScriptMessage
封装了来自于Web页面的消息。包含以下属性：

Attribute | Description
----------|------------------------------------------------------------
`body`    | 消息体，可能是 NSNumber, NSString, NSDate, NSArray, NSDictionary 或 NSNull 其中的任意类型。
`name`    | MessageHandler的名字  
`webView` | 发出此消息的WebView

##### 类型映射
Objective-C Type | Javascript Type
-----------------|-----------------
NSNumber         | number, boolean
NSString         | string
NSDate           | Date
NSArray          | Array
NSDictionary     | object，包括自定义的类以及JSON对象
NSNull           | null

### WKUserScript
封装了可以被注入到Web页面中的Javascript，构造的时候需要传入3个参数。

Argument           | Description
-------------------|------------------------------------------------------------
`source`           | Javascript源码
`injectionTime`    | Javascript脚本被注入的时间，有2个选择 **AtDocumentStart** 或 **AtDocumentEnd** 
`forMainFrameOnly` | 是被注入到所有框架中，还是只注入到主框架中

关于 injectionTime:

```
WKUserScriptInjectionTimeAtDocumentStart    Inject the script after the document element has been created, but before any other content has been loaded.
WKUserScriptInjectionTimeAtDocumentEnd      Inject the script after the document has finished loading, but before any subresources may have finished loading.
```
`注意：在 iOS 8 Beta2 中貌似该类并没有开放，API文档无法访问! 后面的版本需要关注！`

