---
layout:     post
title:      "async/await/actor on swift 5.5"
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

### Task

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

