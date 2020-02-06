---
title: ReactiveCocoa 使用指南
layout: post
date: 2019-06-17 22:28:03
tags: 
    - ReactiveCocoa
---

### 简介

`ReactiveCocoa`是由GitHub开源的一款函数响应式编程框架（FRP），打破了Objective-C一贯的命令式编程的风格，结合函数式编程和响应式编程思想，将iOS开发中的各种不同事件抽象成一个数据流（RACSignal），这也被称做信号，并且内部制定了统一接口，并提供了对数据流进行连接、过滤和组合的API接口。

RactiveCocoa 中使用使用到的编程风格：

- 函数式编程（Functional Programming）：使用高阶函数，例如：函数用其他函数作为参数。
- 响应式编程（Reactive Programming）：关注于数据流和变化传播

`ReactiveCocoa`编程框架的强大之处在于，能将原本繁琐的target-action、delegate、KVO、callback等iOS开发中常见的开发模式，使用函数式编程范式将一个个操作串联到一起，并实现了view和Model之间的双向绑定，从而对iOS中实现MVVM开发架构提供了简便而强大的支持。

### 框架类图

> 下图引用自博客《ReactiveCocoa v2.5 源码解析之架构总览》— 雷纯峰的博客

![img1](http://blog.leichunfeng.com/images/ReactiveCocoa%20v2.5.png)

从上面的类图中，我们可以看出，ReactiveCocoa 主要由以下四大核心组件构成：

- 信号源：`RACStream`及其子类；
- 订阅者：`RACSubscriber`的实现类及其子类；
- 调度器：`RACScheduler`及其子类；
- 清洁工：`RACDisposable`及其子类；

其中，信号源又是最核心的部分，其他组件都是围绕它运作的。通过信号，将iOS开发中的所有异步事件封装为一个统一的block回调，大大简化了对异步事件的逻辑处理，从而使整个逻辑代码看起来很连贯、自然，一气呵成。

下面我们通过代码示例来说明，在`RAC`中是如何实现的：

```objc
// 代理方法
[[self rac_signalForSelector:@selector(webViewDidStartLoad:)
    fromProtocol:@protocol(UIWebViewDelegate)]
    subscribeNext:^(id x) {
        // 实现 webViewDidStartLoad: 代理方法
    }];

// target-action
[[self.avatarButton rac_signalForControlEvents:UIControlEventTouchUpInside]
    subscribeNext:^(UIButton *avatarButton) {
        // avatarButton 被点击了
    }];

// 通知
[[[NSNotificationCenter defaultCenter]
    rac_addObserverForName:kReachabilityChangedNotification object:nil]
    subscribeNext:^(NSNotification *notification) {
        // 收到 kReachabilityChangedNotification 通知
    }];

// KVO
[RACObserve(self, username) subscribeNext:^(NSString *username) {
    // 用户名发生了变化
}];
```

然而，`ReactiveCocoa`的强大之处不仅仅是这些，重要的是它还可以将这些信号进行任意地组合、链接、转换等操作，例如将登陆页面中的账号和密码数据的信号通过一个规则进行合并，同时产生一个新的信号：

```objc
[[[RACSignal
    combineLatest:@[ RACObserve(self, username), RACObserve(self, password) ]
    reduce:^(NSString *username, NSString *password) {
      return @(username.length > 0 && password.length > 0);
    }]
    distinctUntilChanged]
    subscribeNext:^(NSNumber *valid) {
        if (valid.boolValue) {
            // 用户名和密码合法，登录按钮可用
        } else {
            // 用户名或密码不合法，登录按钮不可用
        }
    }];
```

### RAC常用API手册

### 常见类

#### RACSiganl 信号类。

- RACEmptySignal ：空信号，用来实现 RACSignal 的 +empty 方法；
- RACReturnSignal ：一元信号，用来实现 RACSignal 的 +return: 方法；
- RACDynamicSignal ：动态信号，使用一个 block - 来实现订阅行为，我们在使用 RACSignal 的 +createSignal: 方法时创建的就是该类的实例；
- RACErrorSignal ：错误信号，用来实现 RACSignal 的 +error: 方法；
- RACChannelTerminal ：通道终端，代表 RACChannel 的一个终端，用来实现双向绑定。

#### RACSubscriber 订阅者

#### RACDisposable 用于取消订阅或者清理资源，当信号发送完成或者发送错误的时候，就会自动触发它。

- RACSerialDisposable ：作为 disposable 的容器使用，可以包含一个 disposable 对象，并且允许将这个 disposable 对象通过原子操作交换出来；
- RACKVOTrampoline ：代表一次 KVO 观察，并且可以用来停止观察；
- RACCompoundDisposable ：它可以包含多个 disposable 对象，并且支持手动添加和移除 disposable 对象
- RACScopedDisposable ：当它被 dealloc 的时候调用本身的 -dispose 方法。

#### RACSubject 信号提供者，自己可以充当信号，又能发送信号。订阅后发送

- RACGroupedSignal ：分组信号，用来实现 RACSignal 的分组功能；
- RACBehaviorSubject ：重演最后值的信号，当被订阅时，会向订阅者发送它最后接收到的值；
- RACReplaySubject ：重演信号，保存发送过的值，当被订阅时，会向订阅者重新发送这些值。可以先发送后订阅

#### RACTuple 元组类,类似NSArray,用来包装值.

#### RACSequence RAC中的集合类

#### RACCommand RAC中用于处理事件的类，可以把事件如何处理,事件中的数据如何传递，包装到这个类中，他可以很方便的监控事件的执行过程。

#### RACMulticastConnection 用于当一个信号，被多次订阅时，为了保证创建信号时，避免多次调用创建信号中的block，造成副作用，可以使用这个类处理。

#### RACScheduler RAC中的队列，用GCD封装的。

- RACImmediateScheduler ：立即执行调度的任务，这是唯一一个支持同步执行的调度器；
- RACQueueScheduler ：一个抽象的队列调度器，在一个 GCD 串行列队中异步调度所有任务；
- RACTargetQueueScheduler ：继承自 RACQueueScheduler ，在一个以一个任意的 GCD 队列为 target 的串行队列中异步调度所有任务；
- RACSubscriptionScheduler ：一个只用来调度订阅的调度器。

### 常见用法

- rac_signalForSelector : 代替代理
- rac_valuesAndChangesForKeyPath: KVO
- rac_signalForControlEvents:监听事件
- rac_addObserverForName 代替通知
- rac_textSignal：监听文本框文字改变
- rac_liftSelector:withSignalsFromArray:Signals:当传入的Signals(信号数组)，每一个signal都至少sendNext过一次，就会去触发第一个selector参数的方法。

### 常见宏

- RAC(TARGET, [KEYPATH, [NIL_VALUE]])：用于给某个对象的某个属性绑定
- RACObserve(self, name) ：监听某个对象的某个属性,返回的是信号。
- @weakify(Obj)和@strongify(Obj)
- RACTuplePack ：把数据包装成RACTuple（元组类）
- RACTupleUnpack：把RACTuple（元组类）解包成对应的数据
- RACChannelTo 用于双向绑定的一个终端

### 常用操作方法

- flattenMap map 用于把源信号内容映射成新的内容。
- concat 组合 按一定顺序拼接信号，当多个信号发出的时候，有顺序的接收信号
- then 用于连接两个信号，当第一个信号完成，才会连接then返回的信号。
- merge 把多个信号合并为一个信号，任何一个信号有新值的时候就会调用
- zipWith 把两个信号压缩成一个信号，只有当两个信号同时发出信号内容时，并且把两个信号的内容合并成一个元组，才会触发压缩流的next事件。
- combineLatest:将多个信号合并起来，并且拿到各个信号的最新的值,必须每个合并的signal至少都有过一次sendNext，才会触发合并的信号。
- reduce聚合:用于信号发出的内容是元组，把信号发出元组的值聚合成一个值
- filter:过滤信号，使用它可以获取满足条件的信号.
- ignore:忽略完某些值的信号.
- distinctUntilChanged:当上一次的值和当前的值有明显的变化就会发出信号，否则会被忽略掉。
- take:从开始一共取N次的信号
- takeLast:取最后N次的信号,前提条件，订阅者必须调用完成，因为只有完成，就知道总共有多少信号.
- takeUntil:(RACSignal *):获取信号直到某个信号执行完成
- skip:(NSUInteger):跳过几个信号,不接受。
- switchToLatest:用于signalOfSignals（信号的信号），有时候信号也会发出信号，会在signalOfSignals中，获取signalOfSignals发送的最新信号。
- doNext: 执行Next之前，会先执行这个Block
- doCompleted: 执行sendCompleted之前，会先执行这个Block
- timeout：超时，可以让一个信号在一定的时间后，自动报错。
- interval 定时：每隔一段时间发出信号
- delay 延迟发送next。
- retry重试 ：只要失败，就会重新执行创建信号中的block,直到成功.
- replay重放：当一个信号被多次订阅,反复播放内容
- throttle节流:当某个信号发送比较频繁时，可以使用节流，在某一段时间不发送信号内容，过了一段时间获取信号的最新内容发出。

### UI - Category（常用汇总）

#### rac_prepareForReuseSignal： 需要复用时用

- 相关UI: MKAnnotationView、UICollectionReusableView、UITableViewCell、UITableViewHeaderFooterView

#### rac_buttonClickedSignal：点击事件触发信号

- 相关UI：UIActionSheet、UIAlertView

#### rac_command：button类、刷新类相关命令替换

- 相关UI：UIBarButtonItem、UIButton、UIRefreshControl

#### rac_signalForControlEvents: control event 触发

- 相关UI：UIControl

#### rac_gestureSignal UIGestureRecognizer 事件处理信号

- 相关UI：UIGestureRecognizer

#### rac_imageSelectedSignal 选择图片的信号

- 相关UI：UIImagePickerController

#### rac_textSignal

- 相关UI：UITextField、UITextView

### 可实现双向绑定的相关API

- `rac_channelForControlEvents: key: nilValue:`
- 相关UI：UIControl类
- `rac_newDateChannelWithNilValue:`
- 相关UI：UIDatePicker
- `rac_newSelectedSegmentIndexChannelWithNilValue:`
- 相关UI：UISegmentedControl
- `rac_newValueChannelWithNilValue:`
- 相关UI：UISlider、UIStepper
- `rac_newOnChannel`
- 相关UI：UISwitch
- `rac_newTextChannel`
- 相关UI：UITextField

### Foundation - Category （常用汇总）

#### NSData

- rac_readContentsOfURL: options: scheduler: 比oc多出线程设置

#### NSDictionary

- rac_sequence
- rac_keySequence key 集合
- rac_valueSequence value 集合

#### NSArray

- rac_sequence 信号集合

#### NSFileHandle

- rac_readInBackground 后台线程读取

#### NSInvocation

- rac_setArgument: atIndex: 设置参数
- rac_argumentAtIndex 取某个参数
- rac_returnValue 所关联方法的返回值

#### NSNotificationCenter

- rac_addObserverForName: object:注册通知

#### NSObject

- rac_willDeallocSignal 对象销毁时发动的信号
- rac_description debug用
- rac_observeKeyPath: options: observer: block:监听某个事件
- rac_liftSelector: withSignals: 全部信号都next在执行
- rac_signalForSelector: 代替某个方法
- rac_signalForSelector:(SEL)selector fromProtocol:代替代理

#### NSString

- rac_keyPathComponents 获取一个路径所有的部分
- rac_keyPathByDeletingLastKeyPathComponent 删除路径最后一部分
- rac_keyPathByDeletingFirstKeyPathComponent 删除路径第一部分
- rac_readContentsOfURL: usedEncoding: scheduler: 比之OC多线程调用
- rac_sequence

#### NSURLConnection

- rac_sendAsynchronousRequest 发起异步请求

#### NSUserDefaults

- rac_channelTerminalForKey 用于双向绑定，此乃一

#### NSEnumerator

- rac_sequence

#### NSIndexSet

- rac_sequence

#### NSOrderedSet

- rac_sequence

#### NSSet

- rac_sequence

### 总结

本文简单介绍了`ReactiveCocoa`函数响应式库，并列举了在实际开发中几个常见的用法，真实感受到了`ReactiveCocoa`库的强大和精妙之处。对iOS开发提供了一条新思路，为实现MVVM架构提供了视图与模型的绑定功能，同时也提升了iOS应用的开发效率。

最后，如果你对响应式编程感兴趣，鉴于强大的`ReactiveCocoa`库，赶紧接入到自己的项目中体验吧！

### 参考/引用

> [ReactiveCocoa v2.5 源码解析之架构总览](<http://blog.leichunfeng.com/blog/2015/12/25/reactivecocoa-v2-dot-5-yuan-ma-jie-xi-zhi-jia-gou-zong-lan/>)
>
> [iOS ReactiveCocoa 最全常用API整理](<http://www.cocoachina.com/ios/20160729/17236.html>)

