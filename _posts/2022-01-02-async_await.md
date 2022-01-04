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


