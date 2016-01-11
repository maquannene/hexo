---
layout: post
title: "从 Self-Mananger 说开去"
date: 2016-01-05 09:28:24 +0800
comments: true
tags: "iOS"
categories: "技术探讨"
toc: false
---

前一阵看了一片文章，是来自于百度的阳神（Sunnyxx）写的 [iOS 开发中的 Self-Manager 模式](http://blog.sunnyxx.com/2015/12/19/self-manager-pattern-in-ios/)，对于文章中所描写的使用 Self-Mananger 的场景，我在平时的开发中也遇到过很多次，所以在这里就举几个例子写一下我对于此类问题自己的一些看法。

<!--more-->

#### 从具体例子出发：
关与对 View 写类别，并将原属于 Controller 中的业务逻辑东西移入类别中，这里拿一个我最近写的一个功能做例子：

![密码锁](http://ww2.sinaimg.cn/large/65312d9agw1ezoh83xpe9j206y05cdfv.jpg)

这是一个输入验证密码的 View，他的功能是当用户输入密码时，图中的圆圈填充记录，当输入位数到 4 位时，进行密码验证。

那么理所当然这个 View 要保持纯净，即 View 类中不包含业务逻辑，并且能提供复用、易拆除等特点，我们需要将验证的事件回调出去，所以这个类大概是这样的：

```objc
@protocol PwdVerifyViewDelegate <NSObject>

- (BOOL)pwdVerifyView:(PwdVerifyView *)view canThroughPasswordVerify:(NSString *)password;

@end

@interface PwdVerifyView : NSObject

@property (nonatomic, weak) id<PwdVerifyViewDelegate> delegate;

@end
```
我们需要在一个 ControllerA 中用到这个 PwdVerifyView时，只需要实例化 PwdVerifyView 并且将 pwdVerifyView.delegate 指向 ControllerA 并实现代理即可。

如果想要将 pwdVerifyView.delegate 的逻辑单独分离出去，只需要对 ControllerA 写一个 category 即 ControllerA（PwdVerifyView），让其实现代理即可。

所以我当时也就理所当然的在阳神的博客下面留下了这段话：

![评论](http://ww1.sinaimg.cn/large/65312d9agw1ezoh84gkawj21r005m75o.jpg)

#### 忽略的点：

后来我发现我忽略了 sunnyxx 所说蛋疼问题的第二点，比如类似这样的 View 可能出现在工程的其他几个地方，并且每次需要校验的密码时一致的，即代理中校验的过程是一致的，那么我们不得不在每个实例化这个 View 的控制类中去重复实现相同的代理和处理逻辑，此时 Self-Mananger 这种模式就开始发挥威力了。

#### 改善：
接下来，我们对这个 PwdVerifyView 开一个 Category，叫 PwdVerifyView（SelfManager），且自己实现自己的代理，大致代码如下：

```objc
//	类方法实例
+ (instancetype)PwdVerifyViewWithSelfMananger
{
    PwdVerfiyView *pwdVerfiyView = [[PwdVerfiyView alloc] init];
    pwdVerfiyView.delegate = pwdVerfiyView;
    return pwdVerfiyView;
}

- (BOOL)pwdVerifyView:(PwdVerifyView *)view canThroughPasswordVerify:(NSString *)password
{
    ...
    利用 SSKeychain 中存的密码验证
    ...
}

```

到这里我以为万事大吉了，但是 **这个世界在变，而人的心也在变，产品也在变**，直到有一天他们告诉我有5个地方需要处理一套密码验证逻辑，6个地方需要处理另一套验证逻辑（夸张一下），当时我想，好办，再开一个 PwdVerifyView（SelfManager2），将代理实现的验证过程改一改就哦了。正当我要开始时，我抖了一个机灵，不对啊，如果我要再写个类别并且重写了那个代理方法，当文件加入工程中时，就会把先加入工程中的那个类别中的代理方法覆盖了，我百思不得骑姐之后想了下，如果代理会覆盖，那我就不用代理，用 block 回调。

**block 和 delegate 回调方式的最大区别在于：**block 回调是离散式的，哪里创建回调哪里；delegate 可以多对一可聚合，即多个相同组件代理可以指向同一个对象，并调用同一个方法，具体处理流程可在方法中做区分。

delegate 是基于方法的，那么 category 中重复实现接口就会被覆盖，所以最后我将 PwdVerifyView 改成了 block 回调，如下：

```objc
@interface PwdVerifyView : NSObject

@property (nonatomic, copy, nullable) BOOL (^canThroughPasswordVerify) (PwdVerifyView *pwdVerifyView, NSString *passwrod);

@end
```

此时我也不用新开一个类别，只用将原来类别中代理回调改为 block 回调，并且新增一个类方法创建另一种验证方式的 PwdVerifyView 即可，代码如下：

```objc
+ (instancetype)PwdVerifyViewWithSelfManangerInXXXCondition
{
    PwdVerfiyView *pwdVerfiyView = [[PwdVerfiyView alloc] init];
    pwdVerifyView.canThroughPasswordVerify = ^(PwdVerifyView *pwdVerifyView, NSString *password) {
    	...
    	xxxCondition 条件下利用 SSKeychain 中存的另一种密码验证
    	...
        return xxx;
    };
    return pwdVerfiyView;
}
```
#### 继续思考：
那么这种用法到底有什么局限性呢？

我发现如果一旦将整个流程规制在类别中，那基本算是和外界断了耦合；换句话说外界用这个类别的类方法，整个密码验证逻辑和外界是没有任何关系的。但是如果我们有需求验证的最终密码由外面提供，那么我们就可以给类构造方法添加参数，类似下面：

```objc
+ (instancetype)PwdVerifyViewWithSelfMananger:(NSString *)pwd
{
    PwdVerfiyView *pwdVerfiyView = [[PwdVerfiyView alloc] init];
    pwdVerifyView.canThroughPasswordVerify = ^(PwdVerifyView *pwdVerifyView, NSString *password) {
    	...
    	（1）利用传进来的 pwd 和 password 进行对比运算进行验证等
    	（2）当然不只有对比而已，可能还要操控 pwdVerifyView 进行错误提示等等。
    	...
        return xxx;
    };
    return pwdVerfiyView;
}

```
pwd 可能处处不同，所以加参数由外检提供，属于耦合部分；而（1）（2）流程都属于公有行为，即处处相同，属于无耦合部分。

#### 总结：
经过对这个类一系列的演化，最终总结出以下几点：</br>
1.尽可能将公有的逻辑放在一个地方，只写一次，用时每次调用即可，如上述相同的验证流程；</br>
2.有耦合条件时，有外接提供，另加参数，可提供不同的类构造方法；</br>
3.针对于离散式回调行为 block 比 delegate 更适合；</br>
4.具体应该将一个模块封装到多高的可复用程度？上面的代码中，复用可能程度最高的是 PwdVerfiyView，这一部分是不依赖业务逻辑的，可随处复用；接下来是 PwdVerfiyView（SelfMananger），这一部分在工程中可复用；最后把不可复用的部分由外界提供；拥有这样的代码结构，我们在后期拆除代码时才能轻松的一层一层剥掉不需要的，去用需要的部分；

其实回头看看，这个问题并没有什么好研究的，也没什么高端技巧，就是我个人对于代码的矫情程度罢了罢了。
