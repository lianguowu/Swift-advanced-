# 并发1-Async/Await


## 并发

async/await 是最重要的变化，它让我们能够对异步代码使用像是内建错误处理这样的 Swift 结构化编程技术，从而让我们好像是在写同步代码一样。和 completion handler 相比，async/await 要更加简单，也更容易理解。
```
func loadEpisodes() async throws -> [Episode] {
	let session = URLSession.shared
	let (data, _) = try await session.data(from: Episode.url)
	return try JSONDecoder().decode([Episode].self, from: data)
}

//同样的例子，不使用 async/await，而是使用 completion handler 的话：

func loadEpisodesCont( _ completion: @escaping (Result<[Episode], Error>) -> ())
{
	let session = URLSession.shared
	let task = session.dataTask(with: Episode.url) { data, _, err in 		completion(Result {
			guard let d = data else {
			throw (err ?? UnknownError())
		}
			return try JSONDecoder().decode([Episode].self, from: d)
		})
	}
	task.resume()
}
```

注意，在第一个 (使用 async/await 的) 例子中，执行顺序是自上而下的，这和普通的结构化编程是一样的。在数据从网络加载的过程中，这个函数被挂起 (suspend) 了。然而，在第二个 (使用 completion handler 的) 例子中，执行顺序进行了分叉：一部分代码会在 task.resume() 之后继续执行，而另一部分代码 (completion handler 中的部分) 则在网络请求完成 (或者失败) 的时候异步地运行。观察这两个函数的类型，可以看到第一个函数一定会返回一个数组或者抛出一个错误。而 completion handler，我们只能依靠约定来假设 completion handler 会且仅会被调用一次.

一个 completion handler 也可以被称为一个续体 (continuation)。这个名字描述了 completion handler 的目的：它是在 loadEpisodesCont 完成工作后，让程序继续执行的点。类似地，session.dataTask 也不会返回数据，它也会调用提供的续体 (在这里，是闭包表达式所提供的)。在底层，async/await 也使用续体。在上例中，await 后面的部分就是续体。


## Async/Await

在 async/await 以前，我们通过 completion handler (续体) 和 delegate 来编写非阻塞的代码。这在 Apple 的生态系统中已经是多年的标准实践。但是，这种模型可能会导致深层嵌套，难以追踪的续体。更糟糕的是，我们无法使用像是 Swift 内建的错误处理和 defer 等这些标准的结构化编程的组件。

本质上说，async/await 的版本更短，也更容易懂。它同时也更加清晰；通过查看类型，我们知道这个方法要么返回一个错误，要么返回一个剧集的数组，并且它会且只会返回一次。

将 async/await 的代码进行组合，要比组合 completion handler 的代码容易得多，这个差异在需要考虑错误处理的时候更加明显。比如说，让我们把前面的例子扩展一下，让它下载第一集的海报图片。

很明显，async/await 的代码不论在实现上还是接口上，都要比使用 completion handler 的代码简单得多，编译器可以把整类 bug (虽然不是全部) 都消除掉。和 completion handler 相比，async/await 还有一个好处，我们可以使用 defer 来执行那些想要在退出当前函数作用域时完成的操作。不过，要注意 defer 语句中你是不能使用并发代码的。比如，你不能在 defer 语句里异步地更新一个模型对象。


## 异步函数是如何执行的

想要理解异步函数的执行方式，我们可以把一个异步函数分割成不同的部分，其中每个部分都由一个潜在的暂停点 (suspension point) 划定

```
func loadFirstPoster() async throws -> Data {
	// 第一部分
	let session = URLSession.shared
	let (data, _) = try await session.data(from: Episode.url)

	// 第二部分
	let episodes = try JSONDecoder().decode([Episode].self, from: data)
	let imageData = try await session.data(from: episodes[0].poster_url).0

	// 第三部分
	return imageData
}
```
如果我们使用 completion handler 来重写我们的函数，那么这些部分就对应原来函数中的每个同步代码块。在底层，Swift 将包含暂停点的每个异步函数重写为续体。上面函数的第一部分将正常运行，但是第二和第三部分都是续体。

Swift 的并发模型被叫做协同式多任务 (cooperative multitasking)。简言之，这意味着函数永远不应该阻塞当前的线程，而是应该自愿地暂停。函数只能在 (被 await 标记的) 潜在暂停点才能暂停。当函数被暂停时，并不意味着当前的线程被阻塞了：相反，此时控制权会被交还给负责线程安排的调度器，在此期间，(对应于其他任务的) 其他作业可以运行在这个线程上。在稍后的时间点上，调度器通过调用函数的续体，来让该函数恢复运行。比如，在上面的函数中，函数在“第二部分”之前暂停，数据被从网络加载进来。在此期间，其他的作业可以被执行，一旦数据可用，这个函数就将继续。

注意，一个被暂停的函数并不承诺在继续时会使用它原来的线程。Swift 运行时为异步任务维护了一个线程池，当这些线程池中的线程可用时，作业会被添加到其中并得以运行。一个异步函数具体运行的线程不应该很重要 (通常也不重要)，除非我们谈论的是主线程，而主线程依然是要特殊对待的。

不过，“当前执行线程”并不是在暂停点会发生变化的唯一事项。更一般地，你必须假设所有非本地状态都发生了变化，因为其他代码有机会在函数被暂停期间执行 (非本地状态包含了全局变量以及当 self 没有值语义时它上面的属性)。这可能会造成一些难以发现，且编译器也无法捕获的 bug。我们会在Actor 重入的部分再回到这个话题。

并发模型被称为协作式的原因，是因为它依赖单独的函数协同工作：不能有函数在等待昂贵的 I/O 操作或者执行长时间任务时去阻塞线程。函数需要通过其他针对 I/O 的异步函数来让它们自身暂停，或者在长耗时工作之间通过调用 Task.yield() 来让其他作业得到执行的机会。

async/await 最适合进行 I/O 相关的等待，因为 Swift 的并发系统是针对高效的任务切换来设计的。相比起切换到另一个线程或者创建一个新线程，将一个函数暂停，然后在空闲出来的线程上继续另一个函数的做法要快得多。如果一个函数能够在等待慢速 I、O 的时候不占用线程，那么系统就能够在几乎没有额外开销的情况下让 CPU 核心保持忙碌。


## 和Completion Handler对接

在已有项目中，你可能会有一些代码已经使用了 completion handler。这可以使你自己的代码，第三方的代码，或者是 Apple 自己的还没有更新到 async/await 的代码。使用 withCheckedContinuation (或者三个近似相关的变体其中之一)，你可以把任意的带有 completion handler 的函数包装成异步函数。

```
func loadEpisodeCont(id: Episode.ID, _ completion: (Result<Episode, Error>) -> ()) {
	// ...
}
```
要把这个方法转成异步 API，我们可以创建一个异步函数，并将对原来函数的调用包装到 withCheckedThrowingContinuation 中去。后者会立即执行它的函数体，然后暂停，直到我们在函数体中调用 cont.resume 才会继续：
```
func loadEpisode(id: Episode.ID) async throws -> Episode {
	try await withCheckedThrowingContinuation { cont in loadEpisodeCont(id: id) { 
			cont.resume(with: $0)
		}
	}
}
```

因为我们原来 loadEpisodeCont 方法中的 completion handler 可能会接收到错误值 (也就是 Result 的 .failure 成员)，我们需要把这个异步包装方法标记为 throws，并使用 withCheckedThrowingContinuation 而非 withCheckedContinuation。“Checked” 这个单词表明这两个函数都会进行一些运行时的检查，来确保我们对 resume 的调用进行且仅进行了一次。对它进行多次调用会产生一个运行时错误。如果我们在调用它之间就把这个续体丢弃了的话，我们也会在运行时收到警告。


withUnsafeContinuation 和 withUnsafeThrowingContinuation 时另外两种变体，它们在运行时要稍微高效一些，因为相比起 checked 的版本，它们跳过了这些安全检查：你必须确保续体恰好被调用了一次。没有调用续体将导致任务永远无法继续，而调用超过一次则进入未定义行为。在编写代码时，最好使用带有检查的版本，如果你真的需要这点性能，也请确保在转换成不安全版本之前，进行仔细的检查。

除了把 loadEpisode 写作异步函数以外，我们也可以把它写成一个异步的 init
```
extension Episode {
	init(id: Episode.ID) async throws {
		self = try await withCheckedThrowingContinuation { cont in loadEpisodeCont(id: id, cont.resume(with:))
		}
	}
}
```
当你想要从 Objective-C 中访问这些异步方法时，会存在一些限制。你可以把异步方法标记成 @objc 来使它们在 Objective-C 中以 completion handler 的形式可见。但是你不能把 init 标记成 @objc 变体，因为异步的初始化方法是没有桥接的。
