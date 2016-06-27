---
title: iOS10 User Notifications 学习笔记
date: 2016-06-27 23:17:41
tags: "iOS" 
categories: "装逼指南"
---

![Advanced Notifications](http://ww3.sinaimg.cn/large/65312d9agw1f5a8zns61uj215y0esdgp.jpg)

这几天研究了一下 iOS10 的 User Notifications，相比于老的，新的通知中心增加了一些功能，比如以 Extension 形式提供给用户使用的 `UNNotificationContentExtension（推送通知内容扩展）` ；供开发者自定义通知触发条件的 `UNNotificationTrigger（触发条件设定）`；以及提供用户可加入 `UNNotificationAttachment（附件携带）` 用来预览。除此之外从苹果的 PDF 上还写了 User Notifications 其它几项优势：

* 和功能更加相似的 API 设计
* 处理远程推送和本地推送处的代码位置相同
* 简单的代理使用模式以及更好的通知管理
* 当然最重要一点：通知扩展 Notification Extension

<!--more-->

具体我会从下面几个方面讲解新通知中心的功能：

* 1. UNNotificationTrigger（触发条件设定）
* 2. UNNotificationAttachment （附件携带）
* 3. UNNotificationContent （通知内容扩展）
* 4. UNNotificationServiceExtension （通知服务扩展）

并且在文章最后给出具体的的 Demo 供大家参考。

### 1. UNNotificationTrigger - 通知触发条件设定

![Trigger](http://ww3.sinaimg.cn/large/65312d9agw1f59uzqam85j21kw0iaq55.jpg)

A UNNotificationTrigger object represents an event that triggers the delivery of a notification. This class is abstract, so you do not create instances of it directly. Instead, you create instances of a concrete subclass that define the trigger conditions you want. You then use the trigger object to create the UNNotificationRequest object needed to schedule the delivery of the notification.

Trigger 是新加入的一个特性，通过此类可设置本地通知的触发条件。UNNotificationTrigger 是通知触发条件类的基类，不应该直接使用它，而是使用下面这些子类：

* UNPushNotificationTrigger

这个是苹果通知服务的 Trigger，对外没有暴露任何接口属性，不需要开发者创建，由系统创建。

* UNTimeIntervalNotificationTrigger

时间触发器，通过初始化方法设置通知的`timeInterval（触发间隔）`和`repeats（是否重复触发）`，若 repeats 设置为 false，通知将在 timeInterval 之后被推送。

* UNCalendarNotificationTrigger

类似 UNTimeIntervalNotificationTrigger，也是时间触发器，只不过初始化时第一个参数为 `DateComponents` 类型，可设置具体的触发时间比如`8:00 AM`等，也可设置是否重复触发。

* UNLocationNotificationTrigger

地点触发器，即通过设置 `CLRegion` 类型参数设置位置信息，当用户处于某一区域时，自动推送通知。

**使用：**

初始化 UNNotificationRequest 通知请求时传入 UNNotificationTrigger 实例，来设置推送通知触发条件：

```Swift
//  通知触发条件设定为2秒触发一次
let trigger = UNTimeIntervalNotificationTrigger.init(timeInterval: 2,
                                                     repeats: true)
                                                     
//  创建通知请求，设置触发器
let request = UNNotificationRequest.init(identifier: identifier,
                                         content: content,
                                         trigger: trigger)
```

### 2. UNNotificationAttachment - 通知附件

![Attachment](http://ww2.sinaimg.cn/large/65312d9agw1f59vm2s2uwj21gc0g2acx.jpg)

A UNNotificationAttachment object contains audio, image, or video content to display alongside the notification content. Your app always supplies attachments. For local notifications, the app adds attachments when creating the rest of the notification’s content. You can add attachments to a remote notification by implementing a notification service extension, as represented by the UNNotificationServiceExtension class.

推送的通知允许夹带一个附件，格式支持视频，音频，图片。

苹果的文档中对文件的类型和大小做了如下限制：

![文件限制](http://ww1.sinaimg.cn/large/65312d9agw1f5a6clk2uij20lq0ki0u7.jpg)

**使用：**

* 对于本地通知，创建 UNMutableNotificationContent（通知内容）时设置 attachments （附件）属性即可加入附件。
* 对于远程推送通知服务，需要实现 UNNotificationServiceExtension（通知服务扩展），在回调方法中处理 payload 时设置 request.content.attachments 属性，之后调用 contentHandler 方法即可。
* 无论是本地或者远程推送，如果没有使用 UNNotificationContentExtension （通知内容扩展），则下拉或重压后（仅限支持3DTouch），附件会直接出现在 Content 中。如下图：

![Attachment](http://ww2.sinaimg.cn/large/65312d9agw1f5a7x2lu34j20a80dw0tl.jpg)

这里需要说明的是 UNNotificationAttachment 的初始化方法：

```Swift
func init(identifier: String, url URL: URL, options: [NSObject : AnyObject]? = [:]) throws
```
的第二个参数 `url` 指的是附件的 url，此附件必须在本地硬盘上，否则创建时可能出现异常抛出。

如果采取的是本地推送，直接读取文件的所在的位置的 URL 即可；如果是远程推送，由于 attachments 的设置是在 NotificationServiceExtension 的 Target 中，若需读取 Bundle 中的文件，记得将资源加入 NotificationServiceExtension Target 的 Copy Bundle Resource，不能直接读取 Containing App Target 中的 Bundle Resource。

### 3. UNNotificationContentExtension - 通知内容扩展

![ContentExtension](http://ww4.sinaimg.cn/large/65312d9agw1f59y734woxj20py068mxu.jpg)

The UNNotificationContentExtension protocol lets you present a custom interface for your app’s notifications. You adopt this protocol in custom UIViewController subclasses, using the view controller’s view to display the notification contents. You deliver your view controller class inside a Notification Content extension.

iOS User Notifications 最大的创新在于加入了以 Extension 形式提供给开发者和用户使用的 UNNotificationContentExtension，真是 $@#$%^&。通知内容扩展需要新建一个 NotificationContent Target，之后只需在 viewcontroller 的中实现相应的接口，即可以对 app 的通知页面进行自定义扩展，扩展主要用于自定义 UI。

![ContentExtension](http://ww2.sinaimg.cn/large/65312d9agw1f5a0bmg3ngj207g08umxf.jpg)

创建好的 Extension Target 自带一个 ViewController，支持 stroyBoard 使用 autoLayout 布局，但是此 ViewController 不支持交互，加载上面的手势、button 等交互控件都将无效。若需交互，使用 UNNotificationAction 提供按键或输入源，关于 UNNotificationAction 的用法之后 UNNotificationAction 小节会具体讲解。

扩展页面可以在 plist 中配置，提供三个 key 进行设置:

![扩展配置](http://ww4.sinaimg.cn/large/65312d9agw1f59zqq40ouj20ss07275w.jpg)

* UNNotificationExtensionCategory: 若需通知支持内容扩展，需要将通知相应的 Category 加入此处。
* UNNotificationExtensionDefaultContentHidden: 默认内容隐藏，如果设为 YES，则最下面通知 content 部分会隐藏。

![contentHidden](http://ww4.sinaimg.cn/large/65312d9agw1f5a0cqbl7aj207e08maaa.jpg)

* UNNotificationExtensionIntialContentSizeRation: 初始内容 Size 的比例。也可以在 viewDidLoad 中使用 self.preferredContentSize 直接设置 Size。

![ration](http://ww2.sinaimg.cn/large/65312d9agw1f5a0b1ufmyj207c08yaae.jpg)

**使用：**

远程和本地通知最终都可以使用此扩展自定义 UI，只需将通知的 categoryIdentifier 加入到 plist 中即可。

* 本地推送时，确保设置的 content.categoryIdentifier 已加入 plist 中。
* 远程推送，需要设置 category 字段，且确保值也已加入 plist 中。

### 4. UNNotificationServiceExtension - 推送服务扩展

![ServiceExtension](http://ww1.sinaimg.cn/large/65312d9agw1f5a7mmfbf7j21e60eqmzi.jpg)

A UNNotificationServiceExtension object, the principle class for a Notification Service app extension, lets you process the payload of a remote (sometimes called push) notification before it is delivered to the user.

UNNotificationServiceExtension 提供在远程推送将要被 push 出来前，处理推送 payload 的机会。此时可以对通知的 request.content 进行配置，如配置 UNNotificationAttachment，添加 userInfo 等。

通过远程推送的通知也支持 UNNotificationContentExtension，只需将推送数据中 category 的值设置为已经添加到 UNNotificationContentExtension plist 中的值即可。

服务器推送信息示例：

```
{
  "aps" : {
    "alert" : {
      "title" : "title",
      "body" : "Your message Here"		
    },
    // 开启可变内容
    "mutable-content" : "1",			
    // category 为已添加入 UNNotificationContentExtension plist 的值时，即可支持 UNNotificationContentExtension
    "category" : "Cheer"				
  },
  // 加入自定义数据，图片 url 路径
  "imageAbsoluteString" : "http://....jpg"
}
```

### 5. UNNotificationAction 通知交互事件

A UNNotificationAction object represents a task that your app can perform in response to a notification. You can define custom different actions for each type of notification that your app supports. The action object itself contains information about how to display that action onscreen. When the user selects that action, the system forwards the action’s identifier string to your app so that you can perform the corresponding task.

UNNotificationAction 代表一个响应通知的事件。可以为每个通知设置不同的交互事件。下拉推送通知或处在锁屏界面侧滑通知时，会出现交互按键。

交互事件主要分为以下两类：

**UNNotificationAction：**

普通点击按键，可设置标示 identifier、 title 及 点击后的响应，例如：foreground 前台响应，destructive 点击后销毁通知，authenticationRequired 响应前是否需要解锁。

![ContentExtension](http://ww4.sinaimg.cn/large/65312d9agw1f5a8uirxk9j20aa0hqwfh.jpg)

**UNTextInputNotificationAction：**

继承自 UNNotificationAction，点击后呼出输入框进行输入交互。需要说的是，如果设备不支持 3DTouch，则不会支持这个功能，又是一个 @$#@%#$ 的设定。

![UNTextInputNotificationAction](http://ww4.sinaimg.cn/large/65312d9agw1f5a92073g9j20ac0e0mxx.jpg)

**响应处理：**

若处于 UNNotificationServiceExtension 通知扩展界面时，点击按键会回调 UNNotificationServiceExtension 扩展接口的方法：

```Swift
func didReceive(_ response: UNNotificationResponse,
                completionHandler completion: (UNNotificationContentExtensionResponseOption) -> Void) 
```

如果不支持 UNNotificationServiceExtension

![noContentExtension](http://ww2.sinaimg.cn/large/65312d9agw1f5a8siotq5j20d2072mxm.jpg)

则点击会回调 UNUserNotificationCenterDelegate 中的方法：

```Swift
func userNotificationCenter(_ center: UNUserNotificationCenter,
                            didReceive response: UNNotificationResponse,
                            withCompletionHandler completionHandler: () -> Void) {
```

### 结束语

到这里 User Notifications 的功能就差不多了，可以看出几个核心的功能都需要 3DTouch 的支持，所以实用性方面还是比较受限制。

UNNotificationAttachment 的加入给产品的带来了更多的思路，这个方向还是很值得去探索的。

如果支持 3DTouch，UNTextInputNotificationAction 对于消息的回复或者评论的回复都很方便，甚至不用开启应用就能评论或转发一条新鲜的微博。

[Demo](https://github.com/maquannene/UserNotifications) 我已上传至 github，如发现问题，请帮忙指正，感谢阅读。（Demo运行时，请重新设置配置文件方可运行）
