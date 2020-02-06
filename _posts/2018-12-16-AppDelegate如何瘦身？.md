---
title: AppDelegate如何瘦身？
layout: post
date: 2018-12-16 22:14:18
tags:
  - iOS
---

### 前言

AppDelegate瘦身指的就是对AppDelegate类进行解耦和拆分。AppDelegate 作为整个应用程序的代理对象，处理应用程序的生命周期、根视图控制器初始化、应用级URL跳转、推送通知接收、网络配置、及其他第三方SDK初始化等工作。

为了让AppDelegate内部代码简洁，逻辑清晰，则需要对其进行整理拆分。下面有几种瘦身方式：

#### 模块化管理

通过一个模块化管理类，将AppDelegate中的职责拆分至各个模块中去处理，简化了AppDelegate内部代码。

**FRDModuleManager**就是一个模块化管理工具，通过在`UIApplicationDelegate`各代理方法中留下钩子，当代理回调时，会将任务分发到各自模块中。实现原理如下图所示：

![img1](http://images.kyson.cn/delegate/appdelegate_04.jpg)

优点：

- 简单，只需要几行代码就可以解决。
- 被添加的每个模块都可以“享受”AppDelegate的各个生命周期。

缺点：

- 每个模块都要初始化并分配内存，当`FRDModuleManager`里注册了大量模块时，会创建大量对象并影响App启动速度。
- 缺少模块初始化优先级，当有三个模块A,B,C时，正好C依赖于B，B依赖于A，如果在配置文件中配置A，B，C的顺序又是打乱时，初始化会出问题。

#### AppDelegate+Service 分类

通过对AppDelegate 添加相应的分类来处理不同的初始化需求，在AppDelegate中调用处理方法，也可以控制调用顺序。 比如：`AppDelegate+AppService` 处理RootViewController的初始化，网络及三方sdk的初始化等任务，`AppDelegate+RemotePush`来处理远程推送相关的逻辑。

```objc
// 初始化window
[self initWindow];

// 初始化app服务
[self initService];

// 启动app
[kCGAppManager appStart];
```

使用分类的优势是没有侵入性，易于维护，结构清晰。缺点是无法直接对分类添加属性，降低了使用灵活性，不过也可以通过RunTime添加关联属性的方式来解决，但是一般情况下不推荐使用。

#### URL路由分发

从App架构层面对AppDelegate进行解耦瘦身，可以使用全局路由来实现页面跳转、通知、逻辑处理等任务。通过注册url，然后调用对应的url和参数实现页面跳转。

**JLRoutes**是一个优秀的URL解析库，可以很方便的处理不同`URL Scheme`以及解析它们的参数，并通过回调block来处理URL对应的操作。

```objc
// 注册路由
[routes addRoute:@"/user/view/:userID" handler:^BOOL(NSDictionary *parameters) {
    NSString *userID = parameters[@"userID"]; // defined in the route by specifying ":userID"

    // present UI for viewing user with ID 'userID'

    return YES; // return YES to say we have handled the route
  }];
// 执行路由
[JLRoutes routeURL:@"/user/view?userID=123"];
```

但是由于[JLRoutes](https://github.com/joeldev/JLRoutes)的内部实现中对应查找URL使用了遍历的方式，时间效率比较低。GitHub上的另一种库[MGJRouter](https://github.com/meili/MGJRouter)解决了这个问题，使用了匹配查找的方式调高了库的运行效率，可参看使用。

#### 参考资料

> [AppDelegate瘦身指南](https://kyson.cn/index.php/archives/105/)