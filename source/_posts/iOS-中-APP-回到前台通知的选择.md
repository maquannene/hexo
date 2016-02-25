---
title: iOS 中 APP 回到前台通知的选择
date: 2016-02-19 15:01:32
tags: "iOS"
categories: "技术探讨"
---

**UIApplicationWillEnterForegroundNotification**
iOS 中检测 APP 回到前台一般使用通知策略，正常的情况下我们选择此条通知，这条通知的接受时机是 app 将要进入前台。</br>
**UIApplicationDidBecomeActiveNotification**
但是也有人会选择这一条通知，从名称上来看，此通知的接受时机是：app 已经变为激活状态。
<!--more-->
我在做一个需求时，在这两条通知的选择上除了点小错。
需求是：当`程序启动`或者`程序从后台切到前台`时，显示输入密码界面。
于是我监听这个通知 `UIApplicationWillEnterForegroundNotification `，然后在 selector 中去 show 密码界面。
结果发现如果程序启动时，这个通知并不会被调到，之后发现有另外一条通知，`UIApplicationDidBecomeActiveNotification`。
这个通知相比于 `UIApplicationWillEnterForegroundNotification` 在启动的时候是会发送的。</br>
总结：
UIApplicationWillEnterForegroundNotification：只有回到前台，启动 app 不算
UIApplicationDidBecomeActiveNotification：回到前台 + 启动 app</br>
如果两条通知都监听，那么 UIApplicationWillEnterForegroundNotification 是要早于 UIApplicationDidBecomeActiveNotification。</br>
简单的知识点，有时候还是存在疏漏的。

