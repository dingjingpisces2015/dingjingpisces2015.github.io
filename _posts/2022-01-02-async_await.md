---
layout:     post
title:      "async/await/actor on swift 5.5"
subtitle:   "swift 5.5 concurrency 合集"
date:       2022-01-02 13:31:00
author:     "dingjingpisces"
header-img: "img/post-bg-2015.jpg"
catalog: true
tags:
    - 技术
    - iOS开发
    - Swift
---


# WWDC sessions 指路

- 异步
  - [Meet async/await in Swift](https://developer.apple.com/videos/play/wwdc2021/10132/)
  - [Swift concurrency: Behind the scenes](https://developer.apple.com/videos/play/wwdc2021/10254)
  - [Optional][Meet AsyncSequence](https://developer.apple.com/videos/play/wwdc2021/10058)
  - [Optional][Discover concurrency in SwiftUI](https://developer.apple.com/videos/play/wwdc2021/10019)
- Structured concurrency 结构化并发?
  - [Explore structured concurrency in Swift](https://developer.apple.com/videos/play/wwdc2021/10134)
- 可变状态串行访问 actor
  - [Protect mutable state with Swift actors](https://developer.apple.com/videos/play/wwdc2021/10133)
- [Concurrency document](https://docs.swift.org/swift-book/LanguageGuide/Concurrency.html)

# async/await 的一些基本使用

> Meet async/await in Swift


### 一些基本用法的例子

使用async前，如果去需要请求一个图片, 需要由开发者保证每个路径都正确的调用了completion

```
func fetchThumbnail(for id: String, completion: @escaping (UIImage?, Error?) -> Void) {
    let request = thumbnailURLRequest(for: id)
    let task = URLSession.shared.dataTask(with: request) { data, response, error in
        if let error = error {
            completion(nil, error)
        } else if (response as? HTTPURLResponse)?.statusCode != 200 {
            completion(nil, FetchError.badID)
        } else {
            guard let image = UIImage(data: data!) else {
                completion(nil, FetchError.badImage)
                return
            }
            image.prepareThumbnail(of: CGSize(width: 40, height: 40)) { thumbnail in
                guard let thumbnail = thumbnail else {
                    completion(nil, FetchError.badImage)
                    return
                }
                completion(thumbnail, nil)
            }
        }
    }
    task.resume()
}
```

使用async之后，代码简短了不少，并且编译器可以检查是否每个路径都正确的返回了

```
func fetchThumbnail(for id: String) async throws -> UIImage {
    let request = thumbnailURLRequest(for: id) // async表示这个函数是可以被挂起的，每个调用到这个函数的函数都需要被标记为async
    let (data, response) = try await URLSession.shared.data(for: request) // await挂起这个函数直到得到结果
    guard (response as? HTTPURLResponse)?.statusCode == 200 else { throw FetchError.badID }
    let maybeImage = UIImage(data: data)
    // thumbnail property is async
    guard let thumbnail = await maybeImage?.thumbnail else  { throw FetchError.badImage }
    return thumbnail
}

```

函数和属性都可以是async的，但async用来修饰属性有两个限制：
1. `async` 可以从来修饰属性`get`方法，也可以`throw`, `async throw`
2. 只有read only的属性可以用`async`修饰

```
extension UIImage {
    var thumbnail: UIImage? {
        get async {
            let size = CGSize(width: 40, height: 40)
            return await self.byPreparingThumbnail(ofSize: size)
        }
    }
}

```

### await可以用在for循环里

```
for await id in staticImageIDsURL.lines {
    let thumbnail = await fetchThumbnail(for: id)
    collage.add(thumbnail)
}
let result = await collage.draw()
```


### Async/await facts

- `async` enables function to suspend
- `await` marks where an async function may suspend execution 
- other work can happen during suspension
- once an awaited async call completes, execution resumes after the `await`

### async on test
可以直接在test里写await等待结果完成做验证就好

```
class MockViewModelSpec: XCTestCase {
    func testFetchThumbnails() async throws {
        XCTAssertNoThrow(try await self.mockViewModel.fetchThumbnail(for: mockID))
    }
}
```

### async context
在系统函数比如viewDidLoad里因为外面不能加async, 可以手动创建一个async的context

```
Task {
   self.image = try? await self.viewModel.fetchThumbnail(for: post.id)
}
```

### continuation
Continuation这块其实不太理解，感觉像是为了一些不好转换的地方提供一个特殊的接口`withCheckedThrowingContinuation`把现有的回调转换成了async/，await的方式，同时提供了是不是对每条路径都有调用`resume`的检查.
这里面`resume`其实就是返回结果,
- resume 每个路径必须被调用一次
- 如果continuation没有调用resume，swift编译器会报warning

```
func getPersistentPosts(completion: @escaping ([Post], Error?) -> Void) {
    do {
        let req = Post.fetchRequest()
        req.sortDescriptors = [NSSortDescriptor(key: "date", ascending: true)]
        let asyncRequest = NSAsynchronousFetchRequest<Post>(fetchRequest: req) { result in
            completion(result.finalResult ?? [], nil)
        }
        try self.managedObjectContext.execute(asyncRequest)
    } catch {
        completion([], error)
    }
}

// to-be
func persistentPosts() async throws -> [Post] {
    typealias PostContinuation = CheckedContinuation<[Post], Error>
    return try await withCheckedThrowingContinuation { (continuation: PostContinuation) in
        self.getPersistentPosts { posts, error in
            if let error = error {
                continuation.resume(throwing: error)
            } else {
                continuation.resume(returning: posts)
            }
        }
    }
}
```
对于delegate的情况，continuation是可以被保存成属性，然后在delegate方法里分别resume的。

```
class ViewController: UIViewController {
    private var activeContinuation: CheckedContinuation<[Post], Error>?
    func sharedPostsFromPeer() async throws -> [Post] {
        try await withCheckedThrowingContinuation { continuation in
            self.activeContinuation = continuation
            self.peerManager.syncSharedPosts()
        }
    }
}

extension ViewController: PeerSyncDelegate {
    func peerManager(_ manager: PeerManager, received posts: [Post]) {
        self.activeContinuation?.resume(returning: posts)
        self.activeContinuation = nil // guard against multiple calls to resume
    }

    func peerManager(_ manager: PeerManager, hadError error: Error) {
        self.activeContinuation?.resume(throwing: error)
        self.activeContinuation = nil // guard against multiple calls to resume
    }
}
```

# 结构化并发

> Explore structed concurrency in Swift

## Task

- Task 提供了一个concurrently执行 async代码的异步环境, 系统会让task context环境内的async代码并发并且高效执行
- 使用task, 编译器会检查task的用法来避免并发问题
- 在task context里调用async函数的时候并不会创建另一个task

### Async let task

表示不等待`async let`修饰的变量绑定，就执行后面的语法
只有`try await`的时候才去等待当前变量绑定
在`try await`之前访问变量并不会导致计算变量（这个时候访问会得到啥？默认的绑定值？)
![async let](https://github.com/dingjingpisces2015/dingjingpisces2015.github.io/raw/master/img/blog/2022.01.02/async_let.png)

一个例子
```
// before using async let
// 用let其实就已经得到了这个值，相当于第二个图片以来第一个图片下载玩才能执行
func fetchOneThumbnail(withID id: String) async throws -> UIImage {
    let imageReq = imageRequest(for: id), metadataReq = metadataRequest(for: id)
   	let (data, _) = try await URLSession.shared.data(for: imageReq)
    let (metadata, _) = try await URLSession.shared.data(for: metadataReq)
    guard let size = parseSize(from: metadata),
        let image = await UIImage(data: data)?.byPreparingThumbnail(ofSize: size) else {
        throw ThumbnailFailedError()
    }
    return image
}

// after using async let
// try await 用来等待async let修饰的变量绑定
func fetchOneThumbnail(withID id: String) async throws -> UIImage {
    let imageReq = imageRequest(for: id), metadataReq = metadataRequest(for: id)
    async let (data, _) = URLSession.shared.data(for: imageReq)
    async let (metadata, _) = URLSession.shared.data(for: metadataReq)
    guard let size = parseSize(from: try await metadata),
        let image = try await UIImage(data: data)?.byPreparingThumbnail(ofSize: size) else {
        throw ThumbnailFailedError()
    }
    return image
}
```
当其中一个任务出错时，另一个任务会被cancel，比如当data请求出错了，metadata的请求会被cancel, `fetchOneThumbnail`函数会等所有的任务完成才抛出异常, 这么做是为了避免内存泄漏

async任务执行是有一个任务树的只有当依赖的任务都完成，上层的任务才会标记为完成,  mark一个task为cancel 并不会stop这task， task还是会继续执行，并且把所有依赖的task也被标记为cancel

![async let](https://github.com/dingjingpisces2015/dingjingpisces2015.github.io/raw/master/img/blog/2022.01.02/task_tree.png)

### Cancellation is cooperative

其实我并不太理解这段具体的含义，感觉是需要使用者来保证cancel是可以被传递给系统函数的？

- Tasks are not stopped immediately when cancelled
- Cancellation can be checked from anywher
- Design with cancellation in mind

```
if (Task.isCancelled) { return }
try Task,checkCancellation()

````

### Group tasks

group task是用来并行的执行一系列子任务,group task和async let task可以组合使用

![group](https://github.com/dingjingpisces2015/dingjingpisces2015.github.io/raw/master/img/blog/2022.01.02/group.png)

group task的错误处理
如果是gruop的block里抛出错误，group里其他的任务都会cancel
如果是group外部抛出异常，group里的其他任务不会被cancel,需要手动取消
### Data race 问题

- Task 创建的闭包是一种特殊类型的闭包叫@Sendable(我理解想上图里的group.async{}就是这样的一个闭包)
- 编译器会检查创建的@Sendable闭包内有没有捕获可变变量，如果有会报错
- 闭包里只支持包含线程安全的变量，比如值类型，int, String，或者actors 以及内部有做同步的类

```
func fetchThumbnails(for ids: [String]) async throws -> [String: UIImage] {
    var thumbnails: [String: UIImage] = [:]
    // return key value in group
    try await withThrowingTaskGroup(of: (String, UIImage).self) { group in
        for id in ids {
            group.async {
                return (id, try await fetchOneThumbnail(withID: id))
            }
        }
        // this part is sequentially, can safely add
        for try await (id, thumbnail) in group {
            thumbnails[id] = thumbnail
        }
    }
    return thumbnails
}
```

### Unstructured Tasks

提供了一组api用来处理不能结构化编程的情况：

- 一些任务需要从non-async环境去开始（比如系统UI函数 viewDidAppear？)
- 任务存在于不同的scope(比如delegate)

```
Task{
// do something
}
// Task对象也可以被赋值，用于后面取消

@MainActor
class MyDelegate: UICollectionViewDelegate {
    var thumbnailTasks: [IndexPath: Task.Handle<Void, Never>] = [:]
    func collectionView(_ view: UICollectionView,
                        willDisplay cell: UICollectionViewCell,
                        forItemAt item: IndexPath) {
        let ids = getThumbnailIDs(for: item)
        thumbnailTasks[item] = async {
            let thumbnails = await fetchThumbnails(for: ids)
            display(thumbnails, in: cell)
        }
    }

    func collectionView(_ view: UICollectionView,
                        didEndDisplay cell: UICollectionViewCell,
                        forItemAt item: IndexPath) {
        thumbnailTasks[item]?.cancel()
    }
}
```
- Inherit actor isolation and priority of the origin context（原来是啥线程，task执行还是啥线程）
- Lifetime is not confined to any scope
- Can be launched anywhere, even non-async functions
- Must be manually cancelled or awaited

### Detached tasks
- Unscoped lifetime, manually cancelled and awaited 声明周期不受限制
- Do not inherit anything form their originating context 也不继承原来的的执行环境
- Optional parameters control priority and other traits 可以控制优先级和task在什么时候，在哪执行

```
Task.detched(priority: .background) { // 创建一个新的task
	withTaskGroup(of: Void.self) { g in // 创建一组想要在后台执行的任务，当然不用这个也行，只是为了取消比较方便
		g.async { writeToLocalCache(thumbnails) } // 子任务是继承父任务的优先级的
        g.async { log(thumbnails) } 
        g.async { ... }
    }
}
```

### 各种task特点总结
![tasks](https://github.com/dingjingpisces2015/dingjingpisces2015.github.io/raw/master/img/blog/2022.01.02/tasks.png)


# Actors

- Actors 用于同步的访问可变状态
- Actors 会把可变状态从程序中分离出来
  - 对该状态的访问都必须通过actor
  - actor 会保证在当前代码执行期间没有其他线程并行的访问该状态

## Actor的使用
Actor是一个类型关键字
- 能力类似 struct, enum, class（可以包含方法，属性之类的）
- 引用类型
- Actor类型和其他类型的关键区别是： actor类会保证同步的访问自己实例变量的数据

当actor正在执行任务的时候，再次访问actor保护的数据的代码会被挂起，直到第一个任务被执行完，第二个任务才会开始

访问actor保护的变量是同步的，这个同步的访问是不能打断的

## Actor的可重入性

尽管actor会保证同步访问实例变量，但是如果在actor之前用了个await，会导致函数挂起，在挂起后actor保护的状态是有可能被其他线程改变的，所以我们需要await后检查actor保护的状态是否改变

![actor_re](https://github.com/dingjingpisces2015/dingjingpisces2015.github.io/raw/master/img/blog/2022.01.02/actor_re.png)

## Actor isolation

### actor可以服从协议
访问actor 实例变量的函数应该是async的因为actor可能会挂起这个函数，但是对一些不是async的系统函数时，可以在前面加`nonisolated`修饰， `nonisolated`意味着这个函数是不属于actor的，在这个函数内部不会使用actor的可变变量(编译器会检查，如果使用了可变变量会报错)

![actor_protocol](https://github.com/dingjingpisces2015/dingjingpisces2015.github.io/raw/master/img/blog/2022.01.02/actor_protocol.png)


### actor closure
closure和函数类似，可以是isolated也可以是nonisolated的

如果在actor类里使用了引用变量，对引用变量的访问就是out of actor，编译器不能保证同步

![actor_pass](https://github.com/dingjingpisces2015/dingjingpisces2015.github.io/raw/master/img/blog/2022.01.02/actor_pass.png)

这里引入了一个新的定义`Sendable`, `Sendable`表示可以安全并发的类型，以下集中类型是`Sendable`的：
- 值类型
- Actor类型
- 不可变量类
- 内部实现了同步的类
- @Sendable 函数类型（一种新定义的类型）
  - Sendable是一种新引入的协议

### @Sendable function 
!感觉有很多东西没搞明白
[To read more SE-0302](https://github.com/apple/swift-evolution/blob/main/proposals/0302-concurrent-value-and-concurrent-closures.md)
[Short version](https://www.hackingwithswift.com/swift/5.5/sendable)

函数是sendable的意味着可以安全的把函数值传给actor

@Sendable 函数类型服从Sendable协议
@Sendable 对closure有一些严格的限制：

- 没有捕获可变变量
- 只能捕获Sendable类型
- 一个同步的sendable closure不能是actor-isolated，因为actor-isolated会允许代码在actor外部调用

### MainActor

MainActor修饰保证代码在主线程执行

```
@MainActor func checkout(_ booksOnLoan: [Book]) {
}

await checkout(booksOnLoan)
```

类型也可以是MainActor的，使用MainActor类型意味着：

- 类里所有的方法和属性的类型都是MainActor的
- 支持把特殊的成员变量,函数设置成nonisolated



# Concurrency背后的设计思想

> Swift concurrency: Behind the scenes

## 线程模型

### Threading model（线程模型）
从线程角度，当一个任务被block而有其他任务等待执行的时候，GCD会创建更多线程来执行任务,这么做出于两个原因：

1. 多创建一个线程，这样保证每个内核都可以有任务去执行， 同时保证app更好的并发
2. 被阻塞的线程可能正在等待某个信号量，而新的任务执行会unblock之前的线程

创建过多线程的问题：

1. 线程数多过CPU core的数量
2. 线程爆炸，可能引发死锁等一系列线程问题
3. 性能问题
 - 内存开销： 被阻塞的线程会使用部分内存和资源直到运行结束
 - 调度开销： 大量线程切换导致的调度问题
从CPU角度

### Swift concurrency 模型

每个CPU内核运行一个线程，被阻塞的代码变成一个blocked continuations， 当阻塞接触的时候就执行这个continuation, 开销就是一次函数切换的开销（比线程切换开销小）

## swift 在运行时层面怎么支持async/await

## 语言上的特性

- `await`和`async`的实现
- Tracking swift task模型依赖

### `await`和`async`的实现

- 不需要跨 suspension point使用的局部变量被存在栈区, `id`, `article`就被存在了栈区的`add`帧里, 因为他们的定义后就被立即使用了，定义和使用之间并没suspension point
- `add`和`updateDatabase`这两个帧存在堆上， 因为async 的frame需要在suspension point前后都可用
- `newArticles`在suspension point前被定义，后面才使用，所以也是存在堆上的`add`frame里
- 随着函数执行，当执行到`save`函数时， `save`函数栈会直接替换掉`add`函数栈，而不是和普通函数一样压栈， 因为await之后需要的变量都被存在了堆上的`add`帧里， 在`save`函数调用的时候，也会在堆上添加一个`save`的帧
- 这个时候save函数是可以被挂起的，正在运行的线程的帧会被正在执行的函数替换掉， 而堆上会有一系列的帧，也就是待执行的continuation
![async-implementation](https://github.com/dingjingpisces2015/dingjingpisces2015.github.io/raw/master/img/blog/2022.01.02/async-implementation.png)

- 当save的数据库请求执行完毕之后，save函数会重新回到栈里执行（线程和cpu可以和原来不是同一个），等save函数执行完毕，在堆上的`save`帧会退掉，栈上的`save`帧会被替换成`add`继续执行
- 当执行到`zip`时，因为zip不是async函数，所以会创建一个zip的栈压在add上执行（栈区）


```
func save(...) async throws -> [ID] { ... }
func add(_ newArticles: [Article]) async throws {
	let ids = try await database.save(newArticles, for:self)
	for (id, article) in zip(ids, newArticles) {
		articles[id] = article
	}
}

func updateDatabase(...) async {
	await feed.add(articles)
}

```
### Tracking swift task模型依赖

这里面的依赖指的是，`await`后面的continuation执行需要等待的`await`执行.这一部分是通过swift runtime实现的.

#### Swift线程模型 （Cooperative thread pool）

只会产生和CPU数量相等的线程（区别于GCD会在block后开启新的线程)

### 采用 swift concurrency

#### 性能

- concurrency 会引入cost(内存分配和swift runtime逻辑)，
- 保证引入concurrency的收益大于管理的cost(一些耗时不高的代码不考虑使用async做)
- 推荐先profile 代码看看性能瓶颈

#### await和原子性

- swift不保证await前后的代码在同一线程执行
- await意味着原子性被显示的broken了
- 应该避免持有锁跨await使用
- 线程相关的数据也应该避免跨await使用

### concurrency让开发者持有了一个运行时合约（Runtime contract: forward progress of threads）

保持runtime contract的编译器支持关键字

![swift_contract](https://github.com/dingjingpisces2015/dingjingpisces2015.github.io/raw/master/img/blog/2022.01.02/swift_contract.png)



- 不要使用unsafe primitives 去等待跨越task边界(比如通过信号量去跨task等待？)
- 可以用debug环境变量LIBDISPATCH_COOPERATIVE_POOL_STRICT=1去测试是否违背了 forward progress
 



### Actor 互斥性

![swift_actor_thread](https://github.com/dingjingpisces2015/dingjingpisces2015.github.io/raw/master/img/blog/2022.01.02/swift_actor_thread.png)

example of news app：
有四个actor:

1. database actor
2. sport actor
3. weather actor
4. health actor


![actor_hopping_1](https://github.com/dingjingpisces2015/dingjingpisces2015.github.io/raw/master/img/blog/2022.01.02/actor_hopping_1.png)

- 当sports feed请求database的时候，sport feed actor暂时被挂起，当前线程开始执行database任务
- 当weather feed需要写入database的时候，因为已经有database任务在执行，因此weather feed被挂起，开始执行另外的任务health actor
- 当database 任务D1执行完，这时会从suspend任务里挑选一个继续在当前线程执行，可能是sport feed也可能是D2,还有可能是其他的任务（这个顺序是根据优先级确定的）
  

![actor_hopping_2](https://github.com/dingjingpisces2015/dingjingpisces2015.github.io/raw/master/img/blog/2022.01.02/actor_hopping_2.png)


### 可重入性和优先级


**(actor的)可重入性：当actor的某个work item被阻塞而又有另外的任务请求当前的actor内容时，会创建新的work item执行。**
例子：
- 当database actor被挂起等待D1
- 这时sport feed请求database save
- database actor会创建一个新的work item D2执行所需要的操作

![actor_reentrancy](https://github.com/dingjingpisces2015/dingjingpisces2015.github.io/raw/master/img/blog/2022.01.02/actor_reentrancy.png)

**swift的concurrency是按优先级顺序执行work item(区别于GCD的严格的先进先出)**

B任务会被移动到1-6任务前先执行，因为B任务有更高的优先级（user initiated）
![actor_priority](https://github.com/dingjingpisces2015/dingjingpisces2015.github.io/raw/master/img/blog/2022.01.02/actor_priority.png)

### Main actor

main actor保证任务在主线程执行


```
// on database actor
func loadArticle(with id: ID) async throws -> Article


@MainActor func updateUI(for article: Article) async

@MainActor func updateArtic(for ids:[ID]) async throws {
	// 不推荐
	for id in ids {
		let article = try await database.loadArticle(with: id) // 因为loadArticle是定义在database actor的，所以有个context switch, 这样就会有个频繁的的线程切换, 比较推荐的方法是在一个actor/线程里把所有任务都执行完，再进行context切换
		await updateUI(for: article)
	}
	// 推荐
	let articles = try await database.loadArticles(with: ids)
	await updateUI(for: articles)
}
```

![main_actor](https://github.com/dingjingpisces2015/dingjingpisces2015.github.io/raw/master/img/blog/2022.01.02/main_actor.png)




### Reference

- https://juejin.cn/post/6973336267794677791
- https://hevawu.github.io/blog/2021/06/09/WWDC2021-Swift-Concurrency
