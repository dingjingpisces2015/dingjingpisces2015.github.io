---
layout:     post
title:      "synchronized实现原理"
date:       2017-10-28 17:40:00
author:     "dingjingpisces"
header-img: "img/post-bg-2015.jpg"
catalog: true
tags:
    - 技术
    - iOS开发
    - 源码阅读 
    - 多线程
    - sychronized
---

# @synchronized实现原理
在iOS面试中，常常会问到锁的性能问题，对于@synchronized方式加锁,会的到认可的回答是，synchronized性能比较差，不建议使用。这个结论往往是通过测试得到，最近看了synchronized的runtime实现，希望从源码的角度分析为什么慢。

先给出结论：**@synchronized持有的锁本质是递归锁，由于开发者使用@synchronized的时候不持有声明锁，这锁其实是由系统持有并维护的，锁的存取会耗费额外的时间。**

## @synchronized与runtime
先通过clang的-rewrite-objc将带有synchronized的的OC代码转换成C,OC代码如下

```
int main(int argc, const char * argv[]) {
    @autoreleasepool {
        NSObject *obj = [NSObject new];
        @synchronized(obj) {
            NSLog(@"sync log");
        }
    }
    return 0;
}
```
转换成C代码如下：

```
int main(int argc, const char * argv[]) {
    /* @autoreleasepool */ { __AtAutoreleasePool __autoreleasepool;
        NSObject *obj = ((NSObject *(*)(id, SEL))(void *)objc_msgSend)((id)objc_getClass("NSObject"), sel_registerName("new"));// NSObject *obj = [NSObject new];
        {
id _rethrow = 0; id _sync_obj = (id)obj; objc_sync_enter(_sync_obj);//调用objc_sync_enter方法，参数是生成的对象obj
try {
  	struct _SYNC_EXIT {
      _SYNC_EXIT(id arg) : sync_exit(arg) {}
  	  ~_SYNC_EXIT() {
        objc_sync_exit(sync_exit);
      }
  	id sync_exit;
    }
    _sync_exit(_sync_obj);//为啥这用小写，感觉是转换问题，理论上不能这么用，看含义是生成局部变量，离开try block时被析构，调用objc_sync_exit函数，参数仍然是obj

    NSLog((NSString *)&__NSConstantStringImpl__var_folders_yl_kqnznqhd4bb7gm2_sfxs1_j40000gn_T_synchronized_7ca75b_mi_0);
       
  } catch (id e) {_rethrow = e;} //cache了加解锁可能抛出的异常，重新付给_rethrow
{ struct _FIN { _FIN(id reth) : rethrow(reth) {}
	~_FIN() { if (rethrow) objc_exception_throw(rethrow); }
	id rethrow;
	} _fin_force_rethow(_rethrow);}
} //析构时，调用objc_exception_throw抛出异常

    }
   }
    return 0;
}

```

抛开大小写和奇怪的调用，@synchronized(obj)的过程可以看成是：

```
1、objc_sync_enter(obj)
2、执行block代码
3、objc_sync_exit(obj)
```

接下来看objc_sync_enter &  objc_sync_exit这两个函数是做什么的


## objc_sync_enter &  objc_sync_exit

下面是objc_sync_enter &  objc_sync_exit的运行时代码（709.1）关键部分进行了注释

objc_sync_enter


```
int objc_sync_enter(id obj)
{
    int result = OBJC_SYNC_SUCCESS;

    if (obj) {
        SyncData* data = id2data(obj, ACQUIRE); //调用id2data获取SyncData结构对象
        assert(data); //如果是空就抛出异常异常
        data->mutex.lock();//最终的枷锁，查看SyncData结构可以发现mutex是一个递归锁
    } else {
        // @synchronized(nil) does nothing
        if (DebugNilSync) {
            _objc_inform("NIL SYNC DEBUG: @synchronized(nil); set a breakpoint on objc_sync_nil to debug");
        }
        objc_sync_nil();
    }

    return result;
}
```

objc_sync_exit

```

// End synchronizing on 'obj'.
// Returns OBJC_SYNC_SUCCESS or OBJC_SYNC_NOT_OWNING_THREAD_ERROR
int objc_sync_exit(id obj)
{
    int result = OBJC_SYNC_SUCCESS;
   
    if (obj) {
        SyncData* data = id2data(obj, RELEASE);  //调用id2datas释放对obj的锁，同时返回SyncData对象
        if (!data) {//找不到就返回错误
            result = OBJC_SYNC_NOT_OWNING_THREAD_ERROR;
        } else {
            bool okay = data->mutex.tryUnlock();//解锁
            if (!okay) {
                result = OBJC_SYNC_NOT_OWNING_THREAD_ERROR;
            }
        }
    } else {
        // @synchronized(nil) does nothing
    }
	

    return result;
}

```

```
typedef struct SyncData {
    struct SyncData* nextData;
    DisguisedPtr<objc_object> object;
    int32_t threadCount;  // number of THREADS using this block
    recursive_mutex_t mutex;
} SyncData;
```

从上述加解锁的过程可以看出，synchronized是通过传入的obj获取到对应的SyncData，最终对SyncData中的递归锁进行操作，实现了同步。从obj到SyncData的转化就要看id2data的实现了。

## id2data-找到对象对应的锁

代码如下

```
static SyncData* id2data(id object, enum usage why)
{
    spinlock_t *lockp = &LOCK_FOR_OBJ(object);// LOCK_FOR_OBJ = sDataLists[obj].lock，从全局变量sDataLists中取出锁

    SyncData **listp = &LIST_FOR_OBJ(object);
    SyncData* result = NULL;

#if SUPPORT_DIRECT_THREAD_KEYS
//这一段是尝试从TLS中快速获取最新访问到的数据,如果发现所给obj和在TLS中存储的obj是同一个的话,更新对象被锁的计数值，并返回
// 目前看到SUPPORT_DIRECT_THREAD_KEYS是 0因此并不会执行
  // Check per-thread single-entry fast cache for matching object
    bool fastCacheOccupied = NO;
    SyncData *data = (SyncData *)tls_get_direct(SYNC_DATA_DIRECT_KEY);
    if (data) { //判断取出的对象是否为空
        fastCacheOccupied = YES;

        if (data->object == object) {//判断是否是同一个对象
            // Found a match in fast cache.
            uintptr_t lockCount;

            result = data;
            lockCount = (uintptr_t)tls_get_direct(SYNC_COUNT_DIRECT_KEY);//获得这个对象被锁定的次数
            if (result->threadCount <= 0  ||  lockCount <= 0) {
                _objc_fatal("id2data fastcache is buggy");
            }

            switch(why) {
            case ACQUIRE: {
                lockCount++;
                tls_set_direct(SYNC_COUNT_DIRECT_KEY, (void*)lockCount);//重新搞回去
                break;
            }
            case RELEASE:
                lockCount--;
                tls_set_direct(SYNC_COUNT_DIRECT_KEY, (void*)lockCount);
                if (lockCount == 0) {
                    // remove from fast cache
                    tls_set_direct(SYNC_DATA_DIRECT_KEY, NULL);//清空被锁的值
                    // atomic because may collide with concurrent ACQUIRE
                    OSAtomicDecrement32Barrier(&result->threadCount);
                }
                break;
            case CHECK:
                // do nothing
                break;
            }

            return result;
        }
    }
#endif

    // Check per-thread cache of already-owned locks for matching object
    SyncCache *cache = fetch_cache(NO);//从缓存中查找这个对象是否存在
    if (cache) { //如果能找到
        unsigned int i;
        for (i = 0; i < cache->used; i++) {
            SyncCacheItem *item = &cache->list[i];
            if (item->data->object != object) continue;

            // Found a match.
            result = item->data;
            if (result->threadCount <= 0  ||  item->lockCount <= 0) {
                _objc_fatal("id2data cache is buggy");
            }
               
            switch(why) {
            case ACQUIRE://如果是获得锁操作，就给lockCount甲乙
                item->lockCount++;
                break;
            case RELEASE:
                item->lockCount--;
                if (item->lockCount == 0) {
                    // remove from per-thread cache
                    cache->list[i] = cache->list[--cache->used];//更新缓存对象的使用状态
                    // atomic because may collide with concurrent ACQUIRE
                    OSAtomicDecrement32Barrier(&result->threadCount);
                }
                break;
            case CHECK:
                // do nothing
                break;
            }

            return result;
        }
    }

    // Thread cache didn't find anything.
    // Walk in-use list looking for matching object
    // Spinlock prevents multiple threads from creating multiple
    // locks for the same new object.
    // We could keep the nodes in some hash table if we find that there are
    // more than 20 or so distinct locks active, but we don't do that now.
   
    lockp->lock();//上面的解释比较清楚了，在缓存中也没找到，就遍历sDataLists查找或者新生成一个SyncData对象，并添加到sDataLists中，因此需要锁定sDataLists的这一列

    {
        SyncData* p;
        SyncData* firstUnused = NULL;
        for (p = *listp; p != NULL; p = p->nextData) {
            if ( p->object == object ) {//查找看是否存在
                result = p;
                // atomic because may collide with concurrent RELEASE
                OSAtomicIncrement32Barrier(&result->threadCount);//更新正在使用该锁的线程数
                goto done;
            }
            if ( (firstUnused == NULL) && (p->threadCount == 0) )
                firstUnused = p;
        }
   
        // no SyncData currently associated with object
        if ( (why == RELEASE) || (why == CHECK) )
            goto done;
   
        // an unused one was found, use it
        if ( firstUnused != NULL ) {//找到了一个已经在sDataLists中的但没有正在使用的SyncData对象，避免直接生成，而是将object指向传入值后并标记线程数为1，进行使用
            result = firstUnused;
            result->object = (objc_object *)object;
            result->threadCount = 1;
            goto done;
        }
    }

    // malloc a new SyncData and add to list.
    // XXX calling malloc with a global lock held is bad practice,
    // might be worth releasing the lock, mallocing, and searching again.
    // But since we never free these guys we won't be stuck in malloc very often.
    result = (SyncData*)calloc(sizeof(SyncData), 1);
    result->object = (objc_object *)object;
    result->threadCount = 1;
    new (&result->mutex) recursive_mutex_t(fork_unsafe_lock);
    result->nextData = *listp;
    *listp = result;
   
 done:
    lockp->unlock();//sDataLists的数据操作结束了，把锁解开
    if (result) {
        // Only new ACQUIRE should get here.
        // All RELEASE and CHECK and recursive ACQUIRE are
        // handled by the per-thread caches above.
        if (why == RELEASE) {
            // Probably some thread is incorrectly exiting
            // while the object is held by another thread.
            return nil;
        }
        if (why != ACQUIRE) _objc_fatal("id2data is buggy");
        if (result->object != object) _objc_fatal("id2data is buggy");

#if SUPPORT_DIRECT_THREAD_KEYS
        if (!fastCacheOccupied) {//更新TLS
            // Save in fast thread cache
            tls_set_direct(SYNC_DATA_DIRECT_KEY, result);
            tls_set_direct(SYNC_COUNT_DIRECT_KEY, (void*)1);
        } else
#endif
        {
            // Save in thread cache
            if (!cache) cache = fetch_cache(YES);//更新缓存
            cache->list[cache->used].data = result;
            cache->list[cache->used].lockCount = 1;
            cache->used++;
        }
    }

    return result;
}
```


可以看到所有的同步对象都由 *StripedMap\<T\>* 这个类进行管理，*StripedMap\<T\>*是objective -C runtime中定义的一种底层结构，实现了一种类似斑马线的结构，一共分了8条线，每个对象根据自己的内存地址被映射到不同的线上，每条线由一个锁控制，这么做的目的是尽可能的减少锁竞争（先挖个坑，之后会补充一篇StripedMap\<T\>的blog）


## 总结
从上面的同步过程可以看到**synchronized加锁本质是递归锁，SyncData这个结构将对象和递归锁绑定，StripedMap\<T\>这个全局结构维护了所有锁，尽管为了提高性能苹果大量的使用了TLS缓存，但比起直接用互斥锁或者递归锁进行加锁，对每个新对象都需要锁住StripedMap\<T\>的某条line,引入了一道加锁，同时还可能引起锁竞争，因此性能会比直接用锁差**
但比较明显的好处是不需要显式维护锁对象，代码阅读上清爽了不少。


