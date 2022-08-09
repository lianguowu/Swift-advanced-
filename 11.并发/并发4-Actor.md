# 并发4-Actor

不论你使用的是老式的线程操作，还是 GCD 队列，又或者 Swift 任务，并发编程都必须要处理共享资源的潜在冲突这一固有问题。只要代码会被并发执行 (以时间共享的方式或者并行的方式)，你就必须保证保护共享资源，比如对象上的属性、一般性质的内存、文件或者数据库连接等。

传统上，我们为此使用过各种各样的锁相关的 API，而最近，GCD 串行队列已经成为确保共享资源无冲突访问的一种流行方式。随着 Swift 原生的并发模型被引入，这门语言获得了一种全新的资源隔离机制：actor。


## 资源隔离

actor 是引用类型，它通过只允许在 self 中对可变属性进行直接访问 (对于 let 声明的常量，可以安全地跨越 actor 边界进行访问)，以及把它的函数部分用一个串行执行上下文进行隔离，来保护状态。有一点需要特别记住，actor 并不是把整个方法的执行进行隔离：一般规则是，只有那些暂停点之间的函数的部分被视为原子操作。对于没有使用 await 调用其他异步函数的方法这种特殊情况，这个方法会被作为单独一个作业执行，整个方法成为一个原子操作。

actor 的访问模型提供了资源隔离，用来避免数据竞争。如果两个线程在同一时间尝试访问同一位置的内存，并且其中之一是写操作的话，就会发生数据竞争。

actor 为它们的属性提供免于数据竞争的保护。不论是读取还是写入，上面所描述的 actor 的访问模型都会严格地限制两个线程同时对一个属性进行访问的可能性。不过，要注意的是，使用 actor 并不会让你的并发代码神奇地自动正确，在竞态条件 (race condition) 这个更广泛的范畴中，数据竞争只是其中一种类型。

让我们假设网络爬虫使用若干个并发的 worker 来处理处于等待状态的 URL，我们想要在所有的 worker 之间共享这个爬虫队列。一个 worker 应该能够从队列中获取到一个等待爬取的 URL，把它新发现的 URL 添加到队列里，然后把处理过的 URL 的爬取结果存储起来。这正对应了 CrawlerQueue 实例的两个属性 pending 和 finished，在多个并发 worker 之间共享的资源，必须要进行免于数据竞争的保护。
```
class CrawlerQueue {
	private var pending: Set<URL> = []
	private var finished: [URL: String] = [:]
	private var queue = DispatchQueue(label: "crawler-queue")

	public func getURL() -> URL? {
		queue.sync {
			self.pending.popFirst()
		}
	}

	public func enqueue(_ url: URL) {
		queue.async {
			guard self.finished[url] == nil else { return }
			self.pending.insert(url)
		}
	}

	public func store(_ contents: String, for url: URL) {
		queue.async {
			self.finished[url] = contents
		}
	}
}

// 这个队列用 actor 的实现看起来很相似，只是去掉了那些将操作派发到私有串行队列的代码：”

actor CrawlerQueue {
	var pending: Set<URL> = []
	var finished: [URL: String] = [:]
	public func getURL() -> URL? {
		pending.popFirst()
	}

	public func enqueue(_ url: URL) {
		guard finished[url] == nil else { return }
		pending.insert(url)
	}

	public func store(_ contents: String, for url: URL) {
	finished[url] = contents
	}
}
```
pending 和 finished 属性默认就可以被 actor 的 方法访问到 (属性是被隔离在 actor 的执行上下文中的)，actor 的每个方法也都运行在 actor 的串行执行器中，这确保了我们不会在属性访问上遇到数据竞争。


## 可重入 
（未完待续）


## Actor的性能

对外部世界来说，actor 的所有公开方法都是异步方法：它们必须通过 await 进行调用，以便让 actor 能切换到它自己的执行上下文中。比如，下面的函数连续调用了两个 actor。我们把这称为任务在 actor 之间进行了跳跃 (hop)：”
```
// Counter 是 actor
let counter1 = Counter()
let counter2 = Counter()
func incrementAll() async {
	await counter1.increment()
	await counter2.increment()
}
```
actor 跳跃不可避免地存在一些开销，但 Swift 的并发系统经过设计，以确保 actor 跳跃要比切换线程或者队列派发快得多，所以你不必太担心这里的性能损耗。因为调用 actor 方法的任务 (在我们的例子中，就是执行 incrementAll 函数的任务) 必须等待 actor 方法完成，所以 actor 可以跑在任务的当前线程上。如果 actor 在调用发生时没有在进行其他工作，那么 actor 跳转就只是在 actor 上设置一个标记，把自己“锁定”起来，让别的调用者不能使用，然后执行一个普通的函数调用。当方法返回时，actor 又被“解锁”。但在竞争存在时 (比如 actor 正在执行其他任务)，上下文的切换就会变得更昂贵。在这种情况下，我们的任务必须在等待 actor 可用期间，暂停并放弃当前线程。如果你能够把你的程序设计成大多数时候actor都免于竞争的话，你就能得到最好的性能特性。

当向 main actor 切换或者从 main actor 切换出来时，actor 跳跃的开销也会变大。这是因为 main actor 是运行在主线程上的，从另一个执行上下文 (通常是运行在协同式线程池的一个线程上) 切换到 main actor，或者反过来从 main actor 切换到一个一般的 actor 时，会涉及到相对昂贵的线程切换。


## Main Actor

有时候你并不需要为了同步某些状态的访问而去创建自己的 actor。特别是对 GUI 程序来说，通常你会想要某个特定的方法或者属性仅在主线程被调用和访问。由于主线程 (或者在 GCD 术语中的主队列) 在 Apple 平台上一直扮演着特殊的角色，它现在也以全局 actor 的方式被暴露出来了，也就是 MainActor。在底层，main actor 使用主线程作为它的串行执行上下文。

“全局 actor 的特殊之处在于它可以被作为标签使用，来标记其他的类型、属性或者函数。比如，不需要把 CrawlerQueue 自身定义为一个 actor，我们也可以使用系统定义的 @MainActor 标签来确保它的方法和属性访问都只发生在主线程上：

```
@MainActor
final class CrawlerQueue {
	var pending: Set<URL> = []
	var inProcess: Set<URL> = []
	var finished: [URL: String] = [:]
	func process(_ handler: (URL) async -> (String, [URL])) async {
	// ...
	}
}
```
将整个类型标记为 @MainActor，相当于把它上面的所有属性个方法一个个单独进行标记为@MainActor。
我们也可以采取相反的做法：把整个类型标记为 @MainActor，然后用 nonisolated 关键字为特定的方法或者属性取消这个标记。

大多数情况下，@MainActor 标签做的事情都符合你的预期：它确保被标记的方法或者属性只会在主线程上被访问。不过，有一些微妙的边缘情况值得思考。具体来说，有三种不同情况我们需要考虑：使用 @MainActor 标注 async 方法，使用它标注非 async 方法，以及使用它标注属性 (我们可以把它想象为非 async 的 getter 和 setter)

+ 异步的 @MainActor 方法：当把一个 async 方法标注为 @MainActor 时，我们可以肯定这个方法中的代码时运行在主线程的。编译器会强制我们在调用这个 async

+ 非异步的 @MainActor 方法。当我们用 @MainActor 标记一个非 async 的方法时，情况会稍为复杂一些。编译器依然会强制我们只能在 main actor 的执行上下文直接调用这个方法，从其他上下文进行调用的话需要加上 await。然而，因为方法是同步的，它不能在运行时再被派发到正确的执行上下文中。所有的检查都必须在编译期间完成，而这种检查可能会在 Objective-C 代码中失效。

+ @MainActor 属性：对于属性进行标记的 @MainActor 执行的也是编译期间的检查。如果我们尝试从一个其他执行上下文访问 main actor 隔离的属性，编译器会阻止我们这么做。

## 推断执行上下文

当你在阅读那些用 GCD 实现并发的 Swift 代码时，你基本上是通过“在脑海中”执行这些代码，并在运行时追踪它们的派发路径，来推断哪些代码是执行在哪个队列的。此外，你会需要查看 Apple 或者第三方 API 提供的文档，来确认 completion handler 或者是 delegate 方法究竟是在主队列还是其他什么队列被执行的。

“有了 Swift 的 actor 隔离函数模型，这个推理过程就不再只是单纯的运行时行为了。你可以通过检查一个 async 函数所在的词法范围，来确定这个函数会运行在哪个 actor 执行上下文中。如果一个方法存在于 actor 中，你就知道不论在运行时这些代码处在什么样的环境里，它们都会在这个 actor 的执行上下文中以隔离的方式运行。同样，任何一个被 (像是 @MainActor 标签这样) actor 隔离 async 函数，都可以确保是在指定的 actor 中隔离运行的。如果 async 函数没有明确地以这两种方式之一进行 actor 隔离，那么它最终就会运行在一个没有关联任何 actor 的泛用执行器上，这个提案对此进行了描述。

