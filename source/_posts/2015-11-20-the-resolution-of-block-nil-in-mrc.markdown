---
layout: post
title: " MRC 下使用 Block 中的对象被释放，野指针的处理"
date: 2015-11-20 17:44:56 +0800
comments: true
tags: "iOS"
categories: "技术探讨"
---

iOS 开发目前已是全民 ARC 的时代，而且苹果的新语言 swift 也只是 ARC的。ARC 环境中，一个 var 声明的是`__weak`，那么当这个 var 不再被 `__strong` 引用时，这个 `__weak` 的 var 就会自动变成 nil，所以大多数情况下不必担心因为没有置 nil 而导致的野指针崩溃。

但是，如果仍旧使用的是MRC，那么就不得不面对这个问题：
网络请求中，大量使用到 block，如果当 block 中引用到了一个被 `__block` 关键字声明的变量，并且在这个 block 被回调的时候，这个 var 已经被释放了，那么此时 block 中捕获的这个 var 就成为了一个野指针。Objective-C 中，向一个 nil 的对象发送消息是不会崩溃的，但是对一个野指针发送对象是必然崩溃的。


那么问题来了，如何在 MRC 下避免这种崩溃的发生。
<!--more-->
#### 方法一：将 MRC 的文件改成 ARC 的
如果都是这样改，那我还写这篇干个毛啊。关键在于，工程中MRC的文件太多了，行数少的一百多行还能改，上千行的就放弃吧。
#### 方法二：搞一个类似于全局的 NSMutblMeSet 去暂存可能会出现问题中所描述的 var
首先这个 NSMutblMeSet 可以是个全局静态的变量，可以是个属于 AppDelegate 的 var and so on，无所谓。如果一个对象涉及到问题中描述的可能会变成野指针的用法，那么这个对象初始化的时候，就注册进这个NSMutblMeSet；delloc时，就从 NSMutblMeSet 中 remove。在block中使用时，先对对象进行是否存在检查，如果存在就向可以向这个对象发送消息，不存在就返回等等。
这其实是我在网上找到了一种方法，这种方法是可以解决想野指针发送消息引起的崩溃，但是并不是我想要的解决方法，装逼点说，就是我觉得这个方法不够优雅。
#### 方法三：实现类似 ARC 下 __weak 的功能，被释放时自动置为 nil
前一阵看到了一篇文章，讲的是 ARC __weak 自动设置为 nil 的原理。所以我就在想，如果仿照 ARC `__weak` 原理应该可以实现。
所以就大致将这个原理写了一下并且做了封装，其实非常简单，就是 `关联` + `block`。


##### 创建一个叫`MMAutoNilHelper`的类


这个类只有一个属性，并且只干一件事情。这个属性是一个 `block`，类型为`void(^MMAutoNilBlock)(void)`;

```objc
typedef void(^MMAutoNilBlock)(void);

@interface MMAutoNilHelper : NSObject

@property (nonatomic, copy) MMAutoNilBlock autoNilBlock;

@end
```


其次：这个类只干的一件事情就是在自己 dealloc 的时候调用这个 block ：

```objc
- (void)dealloc {
    if (_autoNilBlock) {
        _autoNilBlock();
    }
    self.autoNilBlock = nil;
    [super dealloc];
}
```


##### 怎么用`MMAutoNilHelper`这个类


我们的目的在于，在某一个var被释放时，指向这个 var 的 __block 型 var 都自动变为 nil ，那么我们就可以借助这个`autoNilBlock`来帮助我们完成这个事情，具体操作如下：

```objc
__block typeof(self) bSelf = self;
MMAutoNilHelper *autoNilHelper = [[MMAutoNilHelper alloc] init];
autoNilHelper.autoNilBlock = ^{
    bSelf = nil;
};
_setAssociatedObject(bSelf, &autoNilHelper, autoNilHelper, _ASSOCIATION_RETAIN);
[autoNilHelper release];
```
看过代码之后，其实道理很简单易懂了吧。
假如上面的 `bSelf` 随时都有被释放的可能，那么我们实例一个 `MMAutoNilHelper` 对象，并且将这个对象关联到 `bSelf`（就是自己）上，当bSelf被释放的时候，`autoNilHelper` 也会被释放，`autoNilHelper` 的 `dealloc` 方法就会被调用，此时，`_autoNilBlock()` 就会被调用，bSelf就会被设置为了 nil。这时，无论哪个 block 捕获了 bSelf，bSelf 都会变成 nil，这样就不会出现野指针的现象了。


##### 更加优雅一点


如果每次都要写那么多第二步的代码，那岂不是太啰嗦，所以可以创建一个 NSObject 的 category，将第二步的代码放在 category 的一个方法中：

```objc
- (void)MMMrcWeak:(MMAutoNilConfigureBlock)autoNilConfigureBlock {
    MMAutoNilHelper *autoNilHelper = [[MMAutoNilHelper alloc] init];
    //	注意这里，这里不再是直接给autoNilHelper.autoNilBlock赋值，而是用一个 configureBlock 构造 autoNilHelper
    if (autoNilConfigureBlock) {
        autoNilConfigureBlock(autoNilHelper);
    }
    [autoNilHelper release];
}

```

此时，将配置配置 autoNilHelper 写成宏：

```objc
#define MMMrcWeakObserver(x)                                                        \
void * ptr = &x;                                                                    \
[x MMMrcWeak:^(MMAutoNilHelper *autoNilHelper) {                                    \
	//	注意这里关联的地址是取 x 的地址，解除关联也是利用这个 ptr
    objc_setAssociatedObject(x, ptr, autoNilHelper, OBJC_ASSOCIATION_RETAIN);       \
    //	设置自动置 nil 的block
    autoNilHelper.autoNilBlock = ^{                                                 \
        bSelf = nil;                                                                \
    };                                                                              \
}];        
```
此时如果我们在碰到想要 bSelf 自动置为 nil，就只用写一句话 `MMMrcWeakObserver(bSelf)` 即可。

```objc
__block typeof(self) bSelf = self;
//  声明:MMMrcWeak 将bSelf 变成类似 arc下的weak，实现监听，如果当bSelf释放的时候，自动设为nil
MMMrcWeakObserver(bSelf);                                                  
//  进行网络请求
[self qurey:^{
    if (bSelf) {
        NSLog(@"bSelf = %@ 这个指针还存在（没有被置为nil，可能是野指针）", bSelf);
    }
    else {
        NSLog(@"bSelf 不存在");
    }
}];
```

到这里，自动置 nil 的功能已经达到了，似乎这个东西已经可以拿去用了，但是好像发现少了点什么东西，仔细思考后发现，我们只是对 autoNilHelper 这个对象进行了关联 retain，但是并没有在不用的时候解除关联，这样就造成如果 self 不死，可能会关联 N 个 autoNilHelper，这肯定是我们不希望发生的。


那么我们继续加入解除关联的代码，宏大致如下：

```objc
#define MMMrcWeakObserverCancel(x)                                                  \
//	这里就利用了 ptr
objc_setAssociatedObject(x, ptr, nil, OBJC_ASSOCIATION_RETAIN);                     \
```

现在还有一个问题，我们要在下面这段代码中哪里去插入这句代码 `MMMrcWeakObserverCancel(bSelf)` 呢？

```objc
[self qurey:^{
	//	在这里？（1）
    if (bSelf) {
        NSLog(@"bSelf = %@ 这个指针还存在（没有被置为nil，可能是野指针）", bSelf);
        // 在这里？（2）
        return;
    }
    else {
		// 在这里？（3）
        NSLog(@"bSelf 不存在");
    }
    // 在这里？（4）
}];
```
这样一看，似乎 （2）和 （4）都需要加入 `MMMrcWeakObserverCancel `，好像用起来有点蛋疼，于是我将`MMMrcWeakObserverCancel `的宏改为如下:

```objc
#define MMMrcWeakObserverCancel(x)                                                  \
onExit {                                                                            \
    objc_setAssociatedObject(x, ptr, nil, OBJC_ASSOCIATION_RETAIN);                 \
};    
```
`onExit {...}` 这一句的作用是，当跳出当前所在的大括号时，就执行括号中的内容，所以我们就可以理所当然的在一进入 block `（1)处` 就加入 `MMMrcWeakObserverCancel(bSelf)` 这句代码，而不必担心关联提早解除。

最终的代码在这里：
[**简单版**](https://github.com/maquannene/MQAutoNilHelper/tree/master)
[**完整版**](https://github.com/maquannene/MQAutoNilHelper/tree/branch1.1)
