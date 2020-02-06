---
title: 深入剖析Autorelease Pool (自动释放池)
date: 2018-08-28 11:06:49
layout: post
tags: 
     - iOS
---

### 前言

在MRC的内存管理模式下，可以将创建的对象加入自动释放池，程序员则无需手动调用release方法来释放对象，而是当自动释放池销毁的时候会对其中的每一个对象发送release消息，从而达到自动释放的目的。下面我们一步步揭开它的神秘面纱，深度剖析autoreleasepool的实现原理。

### @autoreleasepool 实现原理

**main.m文件中的@autoreleasepool()**

在iOS代码`main.m`文件中，我们可以看到`@autoreleasepool{}`代码块，其中包含的这一行代码将所有事件、消息全部交给了`UIApplication`来处理。需要注意的是：**整个iOS的应用都是包含在一个自动释放池block中的**。

```objc
// main.m 文件中的内容
int main(int argc, char * argv[]) {
    @autoreleasepool {
        return UIApplicationMain(argc, argv, nil, NSStringFromClass([AppDelegate class]));
}
```

我们在命令行中使用 `clang -rewrite-objc main.m` 让编译器重新改写这个文件，可以看到该文件的c++代码实现细节，关键代码如下：

```c++
...
int main(int argc,const char * argv[]){
    /* @autoreleasepool */ { __AtAutoreleasepool __autoreleasepool;
    }
    return 0;
}
...
```

通过分析上述代码，可以看到 `{__AtAutoreleasepool __autoreleasepool;}` 这段代码即是`@atuoreleasepool`的c++实现，通过创建一个`__autoreleasepool`对象来管理其他对象的自动释放操作。

**分析`__AtAutoreleasepool` 结构体**

我们在main.cpp文件中，定位到了其结构体的定义：

```c++
...
struct __AtAutoreleasepool {
	__AtAutoreleasepool() { atautoreleasepoolobj = objc_autoreleasePoolPush();}
    ~__AtAutoreleasepool() { objc_autoreleasePoolPop(atautoreleasepoolpbj);}
    void * atautoreleasepoolobj;
}
...
    
/*
代码解释：
1. __AtAutoreleasepool() 是其构造函数
2. ~__AtAutoreleasepool() 是其析构函数

析构函数：与构造函数相反, 析构函数是在对象被撤销时被自动调用, 用于对成员撤销时的一些清理工作, 例如在前面提到的手动释放使用 new 或 malloc 进行申请的内存空间。

析构函数的特点：
1. 析构函数函数名与类名相同, 紧贴在名称前面用波浪号 ~ 与构造函数进行区分, 例如: ~Point();
2. 构造函数没有返回类型, 也不能指定参数, 因此析构函数只能有一个, 不能被重载;
3. 当对象被撤销时析构函数被自动调用, 与构造函数不同的是, 析构函数可以被显式的调用, 以释放对象中动态申请的内存。
*/
```

这个结构体会在初始化时调用 `objc_autoreleasePoolPush()` 方法，会在析构时调用 `objc_autoreleasePoolPop` 方法。

这表明，我们的 `main` 函数在实际工作时其实是这样的：

```c++
int main(int argc, const char * argv[]) {
    {
        void * atautoreleasepoolobj = objc_autoreleasePoolPush(); 
        // do whatever you want
        objc_autoreleasePoolPop(atautoreleasepoolobj);
    }
    return 0;
}
```

通过分析将`@autoreleasepool{}`的代码展开即可得到上述代码实现，在一个代码块中，先创建`atautoreleasepoolobj` 对象，然后中间插入iOS应用入口代码，最后自动调用析构函数`objc_autoreleasePoolPop`给当中的每一个对象发送 -release 消息销毁对象。

### AutoreleasePool 实现原理

从上一节我们对`__AtAutoreleasepool`结构体分析可以看出，自动释放池得以实现的核心是在它的构造函数和析构函数中，所以我们接下来就分析这两个函数的内部实现。

我们定位到 `objc_autoreleasePoolPush` 和 `objc_autoreleasePoolPop` 函数的实现代码如下：

```c++
void *objc_autoreleasePoolPush(void) {
    return AutoreleasePoolPage::push();
}
void objc_autoreleasePoolPop(void *ctxt) {
    AutoreleasePoolPage::pop(ctxt);
}
```

从函数实现中可以看出，内部使用了一个名为`AutoreleasePoolPage`的c++类，通过调用此类的push() 方法和pop()方法来实现被释放对象的添加和销毁操作。

既然这几个方法是拨开云雾的关键，那么我们就逐一分析：AutoreleasePoolPage的结构、objc_autoreleasePoolPush（） 方法、objc_autoreleasePoolPop（）方法。

**AutoreleasePoolPage的结构**

`AutoreleasePoolPage`的一个c++类，它在NSObject.mm 文件中的定义如下：

```c++
class AutoreleasePoolPage {
    magic_t const magic;
    id *next;
    pthread_t const thread;
    AutoreleasePoolPage * const parent;
    AutoreleasePoolPage *child;
    uint32_t const depth;
    uint32_t hiwat;
};

/* 类成员结构解析：
	magic：用来校验AutoreleasePoolPage 的结构是否完整
	*next：指向最新添加的auoreleased 对象的下一个位置，初始化时指向begin()
	thread：指向当前线程
	parent：指向父结点，第一个结点的 parent 值为 nil
	child：指向子结点，最后一个结点的 child 值为 nil
	depth：代表深度，从 0 开始，往后递增 1；
	hiwat：代表 high water mark 。
*/
```

一个空的AutoreleasePoolPage 的内存结构如下图所示：

![img1](http://blog.leichunfeng.com/images/AutoreleasePoolPage.png)

另外，当 `next == begin()` 时，表示 AutoreleasePoolPage 为空；当 `next == end()` 时，表示 AutoreleasePoolPage 已满。

每个自动释放池都是由一系列的`AutoreleasePoolPage`组成的，并且每一个`AutoreleasePoolPage`的大小都是4096字节（16进制`0x1000`）

**AutoreleasePoolPage在自动释放池中的组织结构**

`AutoreleasePoolPage`在自动释放池中是以**双向链表**的形式链接起来的：

![img1](https://i.loli.net/2019/02/25/5c73a340b7686.png)

> parent 和 child 就是用来构造双向链表的指针。

**objc_autoreleasePoolPush()  方法解析**

其中有个很重要定义`POOL_SENTINEL` ，它叫哨兵对象，本质是一个`nil`的宏定义：

```swift
#define POOL_SENTINEL nil
```

在每个自动释放池初始化调用 `objc_autoreleasePoolPush()` 的时候，都会把一个 `POOL_SENTINEL` push 到自动释放池的栈顶，并且返回这个 `POOL_SENTINEL` 哨兵对象。上面讲解`@autoreleasepool()`代码块的时候，其中的`atautoreleasepoolobj`对象就是一个`POOL_SENTINEL`。

接下来我们分析`objc_autoreleasePoolPush`方法的实现，从上面的分析可以得出`objc_autoreleasePoolPush`方法、`push()`方法等的实现如下：

```c++
/* 代码从上至下依次调用 */ 

// 1. objc_autoreleasePoolPush()
void *objc_autoreleasePoolPush(void) {
    return AutoreleasePoolPage::push();
}

// 2. push()
static inline void *push() {
   return autoreleaseFast(POOL_SENTINEL);
}

// 3. autoreleaseFast(), 参数obj 就是需要自动释放的对象
static inline id *autoreleaseFast(id obj)
{
   AutoreleasePoolPage *page = hotPage();
   if (page && !page->full()) {
       return page->add(obj);
   } else if (page) {
       return autoreleaseFullPage(obj, page);
   } else {
       return autoreleaseNoPage(obj);
   }
}
```

上述方法`autoreleaseFast:`的实现中有三种选择：

> hotPage：可以理解为当前正在使用的 AutoreleasePoolPage

- 当page 存在，且没有满时：调用`page->add(obj)`方法将对象添加至`AutoreleasePoolPage`的栈中
- 当page存在，已经满时：
  - 调用`autoreleaseFullPage`初始化一个新页
  - 调用`page->add(obj)`将obj添加到新创建的page的栈中
- 无page时：
  - 调用`autoreleaseNoPage`创建一个`hotPage`
  - 调用`page->add(obj)`方法将对象添加至`AutoreleasePoolPage`的栈中

**page-> add() 方法的实现**

```c++
id *add(id obj) {
    id *ret = next;
    *next = obj;
    next++;
    return ret;
}
```

将obj对象添加到`hotPage`中，并移动 `*next`指针指向obj对象的下一个位置，本质上是一个压栈操作。

**objc_autoreleasePoolPop()  方法解析**

回顾上述分析，`objc_autoreleasePoolPop`方法的实现如下：

```c++
void objc_autoreleasePoolPop(void *ctxt) {
    if (UseGC) return;
    
    // fixme rdar://9167170
    if (!ctxt) return;

    AutoreleasePoolPage::pop(ctxt);
}

/*
看起来传入任何一个指针都是可以的，但是在整个工程并没有发现传入其他对象的例子。不过在这个方法中传入其它的指针也是可行的，会将自动释放池释放到相应的位置。
*/
```

pop 函数的入参就是 push 函数的返回值，也就是 `POOL_SENTINEL` 的内存地址 。当执行 pop 操作时，内存地址在 `POOL_SENTINEL` 之后的所有 autoreleased 对象都会被 release 。直到 `POOL_SENTINEL` 所在 page 的 `next` 指向 `POOL_SENTINEL` 为止。

下面是某个线程的 autoreleasepool 堆栈的内存结构图，在这个 autoreleasepool 堆栈中总共有两个 `POOL_SENTINEL` ，即有两个 autoreleasepool 。该堆栈由三个 AutoreleasePoolPage 结点组成，第一个 AutoreleasePoolPage 结点为 `coldPage()` ，最后一个 AutoreleasePoolPage 结点为 `hotPage()` 。其中，前两个结点已经满了，最后一个结点中保存了最新添加的 autoreleased 对象 `objr3` 的内存地址。

![img3](https://i.loli.net/2019/02/25/5c73b361cab65.png)

此时，如果执行 `pop(token1)` 操作，那么该 autoreleasepool 堆栈的内存结构将会变成如下图所示：

![img4](https://i.loli.net/2019/02/25/5c73b363bd161.png)

**autorelease 方法**

最后我们分析下`autorelease` 方法内部的实现原理。首先，我们先看一下方法调用栈：

```objc
- [obj autorelease]
└── id objc_object::rootAutorelease()
    └── id objc_object::rootAutorelease2()
        └── static id AutoreleasePoolPage::autorelease(id obj)
            └── static id AutoreleasePoolPage::autoreleaseFast(id obj)
                ├── id *add(id obj)
                ├── static id *autoreleaseFullPage(id obj, AutoreleasePoolPage *page)
                │   ├── AutoreleasePoolPage(AutoreleasePoolPage *newParent)
                │   └── id *add(id obj)
                └── static id *autoreleaseNoPage(id obj)
                    ├── AutoreleasePoolPage(AutoreleasePoolPage *newParent)
                    └── id *add(id obj)
```

 从上面从调用栈中可以看出， 会先调用`AutoreleasePoolPage`的`autorelease()`方法， 最后会调用上面提到`autoreleaseFast`方法，将obj对象添加`AutoreleasePoolPage`中。 下面是`autorelease()`方法的实现：

```c++
static inline id autorelease(id obj)
{
    assert(obj);
    assert(!obj->isTaggedPointer());
    id *dest __unused = autoreleaseFast(obj); // 最终会调用 autoreleaseFast 方法
    assert(!dest  ||  *dest == obj);
    return obj;
}
```

### 总结

整个自动释放池 `AutoreleasePool` 的实现以及 `autorelease` 方法都已经分析完了，归纳总结后，得出了下面的几个要点：

- 自动释放池是由 `AutoreleasePoolPage` 以双向链表的方式实现的
- 当对象调用 `autorelease` 方法时，会将对象加入 `AutoreleasePoolPage` 的栈中
- 调用 `AutoreleasePoolPage::pop` 方法会向栈中的对象发送 `release` 消息

### 参考/引用

> [自动释放池的前世今生 ---- 深入解析 autoreleasepool](https://draveness.me/autoreleasepool)
>
> [Objective-C Autorelease Pool 的实现原理](http://blog.leichunfeng.com/blog/2015/05/31/objective-c-autorelease-pool-implementation-principle/) 