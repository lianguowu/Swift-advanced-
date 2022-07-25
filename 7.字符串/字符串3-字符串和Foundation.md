# 字符串3-字符串和Foundation

## 字符串和Foundation

Swift 的 String 类型和 Foundation 的 NSString 有着非常密切的关系。任意的 String 实例都可以通过 as 操作桥接转换为 NSString，而且那些接受或者返回 NSString 的 Objective-C API 也会把类型自动转换为 String。

由于 Swift 字符串在内存中的原生编码是 UTF-8，而 NSString 是 UTF-16，这种差异会导致 Swift 字符串桥接到 NSString 时会有一些额外的性能开销。这就意味着，给诸如 enumerateSubstrings(in:options:using:) 这样的 Foundation API 传递 Swift 原生字符串可能不如直接传递 NSString 执行快。

你可能也发现了，把这些 Foundation 的 API 进行改造，让它们更符合 Swift 的习惯，是一件很费力的事。而且不得不使用处理 Unicode 的 API 来以另一种方式操作字符串，本身就是很不方便的，这也更容易引入难以发现的 bug。


## 字符范围

```
let lowercaseLetters = ("a" as Character)..."z"
for c in lowercaseLetters { // 错误
	...
}
```
在内建集合类型中我们已经解释过这个操作失败的原因了：Character 并没有实现 Strideable 协议，而只有实现了这个协议的范围才是可数的集合。

让一个不属于你的类型实现某个协议是一种有问题的做法，一般来说，不建议这样。如果你使用的其它程序库添加了同样的扩展，或者原始类型的供应商提供了对同样协议的支持 (但实现的方式很可能和你使用的不同)，就会发生代码冲突。一个更好的做法，是为这种类型创建一个包装类，把要实现的协议添加到这个包装类上。


## CharacterSet

让我们来看看最后一个有意思的 Foundation 类型，它是 CharacterSet。CharacterSet 是一种存储一组 Unicode 码点的有效手段。它经常被用来检查一个特定的字符串是否包含像是 alphanumerics (英数字符) 或者 decimalDigits (数字) 这类某个特定的字符子集中的字符。CharacterSet 这个名字是从 Objective-C 中导入进来的，其实 CharacterSet 和 Swift 的 Character 完全不是一回事。可能 UnicodeScalarSet 会是一个更好的名字。


## Unicode属性

在 Swift 5 里，CharacterSet 的部分功能被移植到了 Unicode.Scalar。我们不再需要 Foundation 中的类型来测试一个标量是否属于某个官方的 Unicode 分类了，现在只要直接访问 Unicode.Scalar 中的某个属性就好了，例如：isEmoji 或者 isWhiteSpace。为了避免在 Unicode.Scalar 中塞入过多的成员，所有 Unicode 属性都放在了 properties 这个名字空间里。

Unicode 标量的这些属性非常底层，它们主要是为表达 Unicode 中那些不为人熟知的术语而定义的。如果在更为常用的 Character 这个层面也提供一些类似的分类，用起来会更加方便。为此，Swift 5 也为 Character 添加了一系列表达字符类型的属性。


## String和Character的内部结构

和标准库中的其他集合类型一样，字符串也是一个实现了写时复制的值语义类型。一个 String 实例存储了一个对缓冲区的引用，实际的字符数据被存放在那里。当你 (通过赋值或者将它传递给一个函数) 创建一个字符串的复制，或者创建一个子字符串时，所有这些实例都共享同样的缓冲区。字符数据只有当与另外一个或多个实例共享缓冲区，且某个实例被改变时，才会被复制。

Swift 原生字符串 (相对于从 Objective-C 接收的字符串) 在内存中是用 UTF-8 格式表示的。如果你了解了这一点，就能通过它获取理论上字符串处理的最佳性能，因为遍历 UTF-8 视图要比遍历 UTF-16 视图或 Unicode 标量视图更快。并且，UTF-8 也是大部分字符串处理工作使用的格式，因为通过文件或网络等数据源获取的内容，也大多使用了 UTF-8 编码。





