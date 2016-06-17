---
title: Swift Cache - Track 实现笔记
date: 2016-06-17 12:47:53
tags: "iOS"
categories: "学习笔记"
---
</br>

### 引言

开始萌生出要写这个 [**Cache**](https://github.com/maquannene/Track) 念头，是想要练习一下 `Swift` 这门语言，顺便实战 `GCD 达到多线程安全` 和思考 `如何写一个易用的库`。所以大概花了一个礼拜的时间完成了初级版，后续断断续续修补功能又花了两个礼拜，最终在 v1.2.0 的时候，达到了一个让我比较满意的程度。

这个库没有用到特别高深的技巧，也没有特别复杂的算法，但是完成的过程让我学习到了很多东西，如果你想要实战 GCD 的基本用法、又或者是想要学习库基本的设计等等，建议读下去。

<!--more-->

### 动手写之前

在开始写这个库之前，我已经拜读过 Objective-C 的一些 Cache 的源码，例如 Star 比较多的 [TMCache](https://github.com/tumblr/TMCache) 以及它的改良版 [PINCache](https://github.com/pinterest/PINCache)，以及功能不是那么强大的 EGOCache 和 SDImageCache，当然还有大名鼎鼎的 [YYCache](https://github.com/ibireme/YYCache)。相比之下 Swift 的此类库就相对少一些，[AwesomeCache](https://github.com/aschuch/AwesomeCache) 算是 Star 相对多一些的库了，其他类似 Haneke 功能不在对比的范围内。

我对几个功能齐全的库的同步读写做了一个大概的测试（这里没有将任何一个 Swift 库加入对比中，因为确实没有找到功能比较齐全的库，比如 AwesomeCache 是没有区分 Memory 和 Disk 的，并且功能比较少。AwesomeCache 的 Memory 直接用的是 NSCache，所以我将 NSCache 加入了测试中）结果如下图：

下图为 `MemoryCache` 对随机产生的不重复 key value 数组进行读写测试：

<img src="http://ww2.sinaimg.cn/large/65312d9agw1f4x68eph0bj20q80iewg6.jpg" width = "400" height = "300" />
<img src="http://ww1.sinaimg.cn/large/65312d9agw1f4x68gjqt9j20pg0iejt2.jpg" width = "400" height = "300" />

YY 和 Track 内部都采用了 LRU 淘汰算法，PIN 和 TM 有简单的淘汰功能，但并没有引入 LRU 算法，所以在写入后的淘汰数据阶段 YY 和 Track 要快于其他 Cache 的重排序淘汰。其中 TM 速度非常慢，原因在于 TM 的 GCD 调度策略存在很大的问题，会导致同步小数据读写性能都损耗在 GCD 的调度上。这里值得一说的是 NSCache 对随机 key value 的读写性能不错，尤其是读，但是一旦出现相似形数据，性能就会变得非常低。

下图为 `DiskCache` 对随机产生的不重复 key value 数组进行读写测试：

<img src="http://ww3.sinaimg.cn/large/65312d9agw1f4yfxjo9rgj20p00ie3zp.jpg" width = "400" height = "300" />
<img src="http://ww4.sinaimg.cn/large/65312d9agw1f4yfxlqtabj20po0iq3zq.jpg" width = "400" height = "300" />

很明显，底层采用 sqlite 的 YY 性能要高于其他所有基于文件系统的库，所以这里基本可以分为 YYDiskCache 和 其他DiskCache。

### 开始动手写

在动手之前，已经了解到了各个库的优劣，所以在写的时候，我尽量提取了一些优点融入了 Track 中，接下来会主要针对以下几点进行说明，某些点对缓存的性能起到了决定性的作用：

#### 1.线程调度

良好的线程调度，是高性能的一个重要保证，如果没有使用良好的线程调度，就会造成上图中 TMMemoryCache 那种结果。（下面都是针对同步操作的效率的讨论，异步操作讨论意义不大）

因为 Cache 要保证多线程安全，那么就必须有一套好的线程调度，经过一些源码的研究，我发现大部分缓存采取的线程调度策略分为下面两种：

**方式一：** `并发队列 + barrier` + `信号量等待` 或 `串行队列` + `信号量等待`

- 同步操作方式：
	* 读：在`并发队列`或`串行队列`中同步进行读取
	* 写：如果写队列为`串行队列`，则写操作直接异步扔串行队列，之后最外层加`信号量等待锁`变同步，详细参考 TMDiskCache 的同步写操作。</br> 如果写队列为`并发队列`，则写操作先外包裹 barrier，保证原子互斥性， 然后异步扔进并发队列，之后最外层加`信号量等待锁`变同步，详细参考 TMMemoryCache 的 同步写操作；</br>
- 异步操作方式：async 到操作队列执行 </br>

上述这种模型，如果使用的是`并发队列`，即 TMMemoryCache 的调度模型，最终能达到读取时支持大并发同步读，写入时用 barrier 保证了写入的原子性、并且和读操作之间的互斥性。

`并发队列` + `barrier` 亦或者直接使用 `串行队列` 看似是一个十分完美的解决方案，但是实际上隐藏着很大的弊端，因为往往使用者会忽略掉线程切换造成的性能损耗。千万不要小看这一点损耗，试着想一下，如果我们写入或者读取的数据非常小，那么就会造成实际写入或者读取的时间远小于线程切换的时间，最终得不偿失。试图想要用并发队列使同一时间尽可能多的执行任务，以提高效率，但实际却发现时间全部消耗在了线程切换上。

同样的道理就是类似下面这种代码：

```Objective-C
dispatch_apply(count, dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_HIGH, 0), ^(size_t i) { 
	operation();
}

//	这里使用 dispatch_apply 放入并发队列执行，如过 operation() 并不是非常耗时，不如直接使用 for loop

```
总结一句话就是：如果发现实际的操作并不是非常耗时，就尽量不要用多线程去优化性能，否则大多数时间反而会消耗在线程切换上。

除此之外，使用这种模型，当大量并发的调用同步读写时，会造成死锁的问题。
   
**方式二：** `并发队列` + `锁`

- 同步操作方式：
	* 读：当前线程直接加锁读。
	* 写：当前线程直接加锁写。
- 异步操作方式：
	* 读：异步到并发队列加锁读。
	* 写：异步到并发队列加锁写。
	* 其实就是异步到并发队列调用上然后调用同步读写。

这种线程安全模型简单点说就是最终的读写操作都加高性能锁，保证每次最终的读写都互斥。相比于方式一，首先解决的问题就是同步操作的效率问题，因为都是在当前线程直接进行读写操作，没有任何线程调度，所以省去了线程切换的开销，同步读写性能远远高于方式一。其次解决的问题是没有了死锁，即使大量并发调用同步读写时，因为没有了方式一的信号量等待使异步变同步，并不会造成线程资源饱和导致无法解锁信号量导致死锁的问题。

相比于方式一，方式二其实并不支持真正的并发同步读，因为最终读操作都是加锁的，所以每个读都互斥，而方式一是可以做到并发读。但是鉴于方式一的策略本身就有死锁的问题，并且这些并不能提高效率的并发操作也是建立在有死锁的风险上，所以方案并不可取。

Track 使用方式二，`MemoryCache` `DiskCache` 文件的类是线程安全的，`LinkList` 中的类是非线程安全的。`MemoryCache` `DiskCache` 文件中在使用 `LinkList` 中的类时做了线程安全封装。

#### 2.最近最不常使用淘汰

缓存的另一个功能是淘汰，每次设置数据完成后，都要对 count（总数） 和 cost（总内存占有量） 超出的部分进行移除，这两个淘汰功能所依据的条件是缓存对象的年龄，即 count 和 cost 淘汰每次从最老的数据开始移除。所以如何对对象年龄进行排序，也是决定性能好坏的因素之一。

在我读过的上面几个库中，实现淘汰的就只有 TM PIN 和 YY，TM 和 PIN 有 cost 和 date 淘汰，YY 和 Track 支持 count、cost、date 淘汰，实现方式分为两种如下：

**方式一：**每次需要淘汰时重新排序，然后从最最老的数据开始移除

TM 和 PIN 各自有一个记录每个存储对象 date 的字典，每次写入之后的淘汰都是基于对这个数组重新排序然后开始末尾移除。

这样做的劣势很明显，就是每次都会重新进行排序，如果你在使用 PINMemoryCache 时，设置了 costLimit 属性，那么你会发现效率立刻从摩托变成了单车，只能用惨不忍睹来形容，并且随着存储对象数量增加，会变得越来越慢。

**方式二：**使用链表也存储一份对象模型的引用，用于排序淘汰

链表数据结构的优势是在已知节点的前提下，插入、移动、和删除的时间复杂度都是O(1)，所以可以借助这个优势用实现 LRU。每次读取数据时，从字典中查询节点，然后在链表中将节点移到头部，写入数据后的淘汰就避免了重新根据年龄排序耗时的问题，直接从链表的末尾开始向前移除即可。

YYMemoryCache 和 Track.MemoryCache 都采用了这种方式，稍微有点不同的是，YY 的 cost 淘汰是异步到主线程调用的，Track 的 cost 是当前线程直接调用，这里各自的利弊就不多做评价了，各有各的想法吧，[具体讨论可以看这里](https://github.com/ibireme/YYCache/issues/54)。另外一点是很值得学习的，YY 在 count 淘汰时，是可选异步 release，设置 `releaseAsynchronously` 即可，这里需要提醒的是，我在小对象写入性能测试时，发现如果设置了 countLimit，YYMemoryCache 的效率就有明显下降，Profile 显示原因在异步到其他线程的过程消耗了一些性能，所以存储对象都为小对象且有 countLimit 时，建议将 releaseAsynchronously 设置为 NO。

#### 3.容器选取及对象存储模型

##### 容器选取

**（1）Memory**

Memory 存储容器大致有这几种：NSCache，NSMutableDic，CFMutableDic

NSCache 和 Dic 的区别在于 NSCache 本身就是线程安全的，所以上层不再需要安全线程调度的保护，而 Dic 是非线程安全的。AwesomeCache 就是采用 NSCache 直接作为内存缓存，NSCache 的效率上面的统计图已经给出，这里就不多说了。我在写 Track 时，本身第一想法是用 Dictionary，但是发现它的效率没有 NSMutableDic 高，所以最终选取了 NSMutableDic 作为存储和查询容器，LinkedList 作为调整顺序的容器。

**（2）Disk**

毫无疑问，文件系统的效率远远小于数据库。由于对数据库并不是很了解，所以关于 Disk 部分就不做讨论了，可以去看 YY 的作者写的文章，里面对磁盘存储做了很好的讨论。

##### 存储模型

这里想说的其实就是 TM 和 PIN 存在的另外一个性能问题，TM 和 PIN 在写入 Memory 时，除了写入本身存储 object 的 dictionary，还另外有两个字典 dates 和 costs。看源码中，会发现每次写入 object 时，都会对这三个字典写入一遍，性能分析时，发现写入的时间基本被这三个操作平分，性能立刻下降一大截。

Track 和 YY 则采用的是将 date 和 cost 都封装起来，每次写入或修改数据时，只用写或者读一次字典，这样性能就提升了很多。

#### 4. 追求性能慎用闭包

我在刚开始写 Track 的同步操作时，发现每次保证线程安全都需要加信号量锁，所以每次的代码都是这样的：

```
lock()
...
unlock()

```

我觉得这样实在是太不优雅了，所以写了个这样的函数：

```
func threadSafe(handler: () -> Void) {
	lock()
	handler()
	unlock()
}
```

后期我在优化性能时，发现闭包简直就是一个性能杀手，最后还是老老实实的前后调用加解锁函数。

### 功能增加

在写到 1.0 版本之后，我就在想，既然库是用 Swift 写的，如果没有一点 Swift 的功能，那岂不是等于抄袭别人写的代码然后翻译了一遍？所以就加入了些 Swift 的东西：

#### 支持 GeneratorType、SequenceType

我给 Track 中底层的的 LinkedList 和 LRU 实现了 `GeneratorType` 和 `SequenceType` 接口，这样 Track 上层封装后也支持了线程安全的 `GeneratorType` 和 `SequenceType`。这样就 Track 的 Cache、Memory、DiskMemory 都支持了 `for ... in` `map` `flapmap` `filter` 等一系列方法，功能更加强大啦。

这里值得一说的是，Disk 的 Generator 有实现的是 `FastGeneratorType`，这样使得 Cache 遍历时，只要 MemoryCache 能读出值，DiskCache 指针快速移动即可，效率并不会降低，当 MemoryCache 读不到内容时，便开始读 DiskCache，并且数据衔结正确。

#### 支持 Subscript

这个就不多说了，小功能，方便使用。

### 写在最后

目前就功能和性能上来讲，Objective-C 的此类 Cache 库中，YYCache 绝对是最好的。Swift 中目前我还没有找同类功能较齐全的库，AwesomeCache 只拥有基本的功能，所以如果你在写 Swift 的项目，正巧需要一个 Cache，那么请使用 [**Track**](https://github.com/maquannene/Track) 吧。

[**Track**](https://github.com/maquannene/Track) 还是花了我一些心思去尽量写好它，包括前期调研、后期加功能以及优化等等，以后仍将继续维护。

所以支持的话，就👍一下吧。