---
layout:     post
title:      "Autorelease的实现原理"
date:       2017-11-26 22:52:00
author:     "dingjingpisces"
header-img: "img/post-bg-2015.jpg"
catalog: true
tags:
    - 技术
    - iOS开发
    - runtime
    - 源码 
---

# Autorelease的实现原理

Autorelease是苹果为开发者提供的用于垃圾回收的API，实现了在每个Runloop结束后自动释放无人引用的内存的功能。默认生成的代码会把我们要执行的代码包裹在AutoreleasePool里，在编译器的配合下，很多时候甚至感觉不到垃圾回收的存在。这篇文章希望通过源码分析Autorelease的实现原理。


## 使用场景

我理解，Autorelease的使用时机有两种：

1. 开发者手动将变量加入AutoreleasePool；
2. 函数返回时，如果有返回对象，编译器自动添加到AutoreleasePool;

## 关于返回值的优化

对于第2中情况，苹果做了优化，对象的引用计数不直接添加在AutoreleasePage里，而是经过TLS中转，直接传递给调用方，这么做还可以避免两侧多余的Retain,Release操作。具体逻辑如下.

被优化后的的被调用方(callee)会查看紧跟在return后的调用方(caller)指令。如果caller的指令也是经过优化的，那么callee会跳过所有引用计数操作（no autorelease,no retain release）。而是在TLS中设置标志位，经过优化的caller会查看TLS，如果发现设置了TLS值，则不对返回对象进行retain/release等操作，而是根据TLS中的值设置retainCount.

在runtime源码中，被优化的两个callee是：
objc_autoreleaseReturnValue  flag设置+1
objc_retainAutoreleaseReturnValue  flag设置+0

被优化的两个caller是：
objc_retainAutoreleasedReturnValue 调用者希望TLS值是+1
objc_unsafeClaimAutoreleasedRetainValue 调用者希望TLS值是+0

如:
Callee:
     return objc_autoreleaseReturnValue(ret)；// flag=1
Caller:
     ret = callee();
     ret = objc_retainAutoreleasedReturnValue(ret); //直接得到ret

Callee发现了优化的caller,设置TLS为+1
Caller查看TLS，清空，此时在没有retain, autorelease的情况下，callee,caller合作使ret得引用计数依然正确。

根据系统的不同，callee会识别一些特殊的机器码。


## AutoreleasePage 

Autorelease的实现数据结构是AutoreleasePage.AutoreleasePage中的地址，以数组形式组织，而AutoreleasePage本身以链表形式组织。
最新的Page称为hotPage,存储在TLS中，当新增一个obj到自动释放池时，如果hotPage已满或者不存在，会新生成一个Page，添加到原有链表中，并将该Page设置为hotPage

如下图所示：
![AutoreleasePage](https://raw.githubusercontent.com/dingjingpisces2015/dingjingpisces2015.github.io/master/img/autoreleasePage.png)

## 数据操作 (Push/Pop)

### Push
向自动释放池中添加一个对象，代码如下：


```
static inline void *push()
    {
        id *dest;
        if (DebugPoolAllocation) {
            // Each autorelease pool starts on a new pool page.
            dest = autoreleaseNewPage(POOL_BOUNDARY);
        } else {
            dest = autoreleaseFast(POOL_BOUNDARY);
        }
        assert(dest == EMPTY_POOL_PLACEHOLDER || *dest == POOL_BOUNDARY);
        return dest;
```

DebugPoolAllocation在不存在AutoreleasePage且可调试的情况下触发，一般而言会走到else的逻辑，即autoreleaseFast,这个函数如下


```
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

可以看到，这个函数

- 首先从TLS中取出了当前正在使用的hotPage,并且在页面不满的情况下，向page中添加这个对象，即向next数组中添加了obj


```
   id *add(id obj)
    {
        id *ret = next;  // faster than `return next-1` because of aliasing
        *next++ = obj;
        return ret;
    }
```

- 当hotPage存在时，调用autoreleaseFullPage方法，生成一个新的page设置为hotPage,并将obj add到这个新hotPage中
- 其他情况下，调用autoreleaseNoPage方法，判断一些极端情况，在逻辑无误的情况下还是生成新page并添加对象

### Pop
Pop的实现与Push类似，都是对Page及next指针的操作。比较特别的是pop可以带一个token进去，实现释放对象，直至这个token位置的功能，
其做法是首先倒叙释放hotPage的next指针对象，如果Page中的next数组中对象全部释放，且没到token位置，那么继续释放Page直到该位置。

## 总结
AutoreleasePage的数据组织结构十分清晰，比较特别的是看到了AutoreleasePage释放后指针是0xA3A3A3A3，以及结合以前代码，看到了大量的对TLS的使用。
