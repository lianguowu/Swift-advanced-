# 错误处理4-defer和Rethrows


## 使用defer进行清理

很多编程语言都有 try/finally 这样的结构，当函数返回的时候，无论是否发生错误，finally 指定的代码块总是会被执行。Swift 中的 defer 关键字功能和它类似，但具体做法却稍有不同。和 finally 类似的是，当离开当前作用域的时候，defer 指定的代码块也总是会被执行，无论是因为执行成功返回，还是因为发生了某些错误，亦或是其它原因。这使得 defer 代码块成为了执行清理工作的首选场所。和 finally 不同的是，defer 不需要前置的 try 或 do 代码块，你可以把它部署到代码中的任何地方。

```
func contents(ofFile filename: String) throws -> String {
	let file = open(filename, O_RDONLY)
	defer { close(file) }
	return try load(file: file)
}
//无论 contents 执行成功或者抛出了错误，第二行的 defer 代码块都可以确保文件在函数返回的时候可以被关闭
```

如果相同的作用域中有多个 defer 代码块，它们将按照定义的顺序逆序执行。你可以把这些 defer 想象成一个栈。起初，你可能会觉得逆序执行 defer 很奇怪，不过，如果看看下面这个执行数据库查询的例子，你就会觉得这种做法非常合理了：
```
let database = try openDatabase(...)
defer { closeDatabase(database) }
let connection = try openConnection(database)
defer { closeConnection(connection) }
let result = try runQuery(connection, ...)
```

在执行查询之前，我们必须打开数据库并创建一个数据库连接。接下来，如果在执行 runQuery 的时候抛出错误了，资源的清理工作当然应该按照和代码正常执行相反的顺序完成。我们应该先关闭数据库连接，再关闭数据库自身。由于 defer 语句就是逆序执行的，因此这正是我们期望的结果。

一个 defer 代码块会在程序离开 defer 定义的作用域时被执行。甚至 return 语句的评估都会在同作用域的 defer 被执行之前完成。你可以利用这个特性在返回某个变量之后再修改某个变量的值。在接下来的例子中，increment 函数在返回 counter 之后，使用 defer 代码块递增了捕获到的 counter：
```
var counter = 0
func increment() -> Int {
	defer { counter += 1 }
	return counter
}

increment() // 0
counter // 1
```
当然，也有一些 defer 语句不会执行的情况，例如：当程序发生段错误的时候，或者触发了致命错误的时候 (使用 fatalError 函数或者强制解包 nil)，这时所有代码执行都会立即终止。


## Rethrows

```
func filter(_ isIncluded: (Element) -> Bool) -> [Element]
```
这个定义没问题，但它有个缺陷：编译器不会接受一个可抛出错误的函数作为谓词，因为 isIncluded 参数没有标记为 throws。

假设我们有一个文件名数组，要从中筛选出不可用的文件。很自然地，我们会选择使用 filter 方法，但是编译器不会让我们这么干，因为 checkFile 是一个可能抛出错误的函数：”
```
func checkFile(filename: String) throws -> Bool

//编译器不会让我们这么干，因为 checkFile 是一个可能抛出错误的函数
let filenames: [String] = ...
// Error: Call can throw but is not marked with 'try'.
let validFiles = filenames.filter(checkFile)
```

解决方案可以在谓词函数中进行错误处理，但这样很不方便，甚至这都不是我们想要的效果 —— 上面的代码用 false 遮掩了 checkFile 所有可能抛出的错误。
```
let validFiles = filenames.filter { filename in
	do {
		return try checkFile(filename: filename)
	} catch {
		return false
	}
}
```

另外一种实现方案是实现两个版本的 filter，分别接受普通的和可抛出错误的谓词函数。除了要用 try 调用谓词函数之外，这两个版本的 filter 实现，是完全一样的。我们可以依赖编译器根据函数重载的规则自动选择正确的版本。这样做看上去更好一些，至少不同版本的调用很清晰，但是这还是太浪费了。
```
func filter(_ isIncluded: (Element) throws -> Bool) throws -> [Element]
```

Swift 通过 rethrows 关键字提供了一个更好的方案。用 rethrows 标记一个函数就相当于告诉编译器：这个函数只有它的参数抛出错误的时候，它才会抛出错误。因此，filter 方法最终的签名是这样的：
```
func filter(_ isIncluded: (Element) throws -> Bool) rethrows -> [Element]
```
谓词函数仍旧被标记成了 throws 函数，表示调用者可能会传递一个可抛出错误的函数。在 filter 的实现里，必须使用 try 调用谓词函数。而 rethrows 则确保了 filter 会把谓词函数中的错误沿着调用栈向上传递，但 filter 自身不会抛出任何错误。因此，当传递的谓词函数不会抛出错误时，编译器就不会要求使用 try 调用 filter 了.


## 将错误桥接到OC

在 Objective-C 里，并没有像 throws 和 try 这样的机制。Cocoa 的通用做法是在发生错误时返回 NO 或者 nil。另外，可能发生错误的方法还会接受一个 NSError 指针的引用作为额外的参数。它们使用这个指针给函数的调用者传递错误的详细信息。例如，Objective-C 版本的 contents(ofFile:) 写出来可能是这样的：
```
- (NSString *)contentsOfFile(NSString *)filename error:(NSError **)error;
```

Swift 会自动把遵循这个规则 (译注：这个规则指的是接受 NSError ** 作 为参数) 的方法转换为 throws 语法的版本。因为不再需要参数传递错误了，它会从声明中删除。于是，contentsOfFile 被导入到 Swift 就会变成这样：
```
func contents(ofFile filename: String) throws -> String
```

