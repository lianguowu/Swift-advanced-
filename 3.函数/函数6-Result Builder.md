# Result Builder

## Result Builder

Result builder 是一种特殊的函数，它允许我们通过简洁和富有表达力的多个语句构建出结果值。
SwiftUI的view bulider语法的原理。
```
struct HStack<Content>: View where Content: View {
	public init(
		alignment: VerticalAlignment = .center, 
		spacing: CGFloat? = nil, 
		@ViewBuilder content: () -> Content)
		// ...
}
```

在一个 result builder 函数中，编译器会重写这些代码，用函数中的所有语句来创建一个合成的值。为了做到这一点，编译器会使用定义在对应 result builder 类型 (比如本例中的 ViewBuilder) 上的一些静态方法。
重写上面的例子时，编译器将使用 ViewBuilder 的静态 buildBlock 方法，它接受三个满足 View 协议的参数：
```
@resultBuilder public struct ViewBuilder {
// ...
	public static func buildBlock<C0, C1, C2>(_ c0: C0, _ c1: C1, _ c2: C2)-> TupleView<(C0, C1, C2)> 
		where C0: View, C1: View, C2: View
		// ...
	}

//重写后的代码将不再依赖于 result builder，类似这样：
HStack {
	return ViewBuilder.buildBlock(
		Text("Finish the Advanced Swift Update"),
		Spacer(),
		Button("Complete") { /* ... */ }
	)
}
```

## Block和表达式（buildExpression）

最基本的构建方法是 buildBlock 和 buildExpression。和所有构建方法一样，它们是静态方法。Builder 类型只是作为命名空间来使用，它们从不会被实例化。
要实现一个 @resultBuilder 类型，只有唯一一个要求，那就是要至少实现一个 buildBlock 方法。因为构建方法允许把多个中间结果合并成一个结果值，所以大部分时候，参数的个数以及它们的类型可能会不同，你会需要实现不止一个的buildBlock 变体。不过要在 builder 函数中支持多种类型的话，实现 buildExpression 通常是更优雅的方式。

```
@resultBuilder
struct StringBuilder {
    static func buildBlock(_ strings: String...) -> String{
        strings.joined()
    }

    //接下来，我们可以通过实现 buildExpression 来为其他类型添加支持，比如整型数：
	static func buildExpression(_ s: String) -> String {
		s
	}
	static func buildExpression(_ x: Int) -> String {
		"\(x)"
	}

}

@StringBuilder func greeting() -> String{
    "Hello, ",
	"World!"
}

greeting()

//Swift 将这个字符串 builder 函数翻译成下面的代码：
func greeting_rewritten() -> String {
	StringBuilder.buildBlock(
		"Hello, ",
		"World!"
	)
}

------------

let planets = [
	"Mercury", "Venus", "Earth", "Mars", "Jupiter", 
	"Saturn", "Uranus", "Neptune"]


@StringBuilder func greetEarth() -> String {
	"Hello, Planet "
	planets.firstIndex(of: "Earth")!
	"!"
}


func greetEarth_rewritten() -> String {
	StringBuilder.buildBlock(
		StringBuilder.buildExpression("Hello, Planet "),
		StringBuilder.buildExpression(planets.firstIndex(of: "Earth")!),
		StringBuilder.buildExpression("!")
	)
}
```

## 条件语句

通过向 builder 类型添加 buildIf 和 buildEither 方法，我们可以把它的能力进行拓展，让它支持 if，if...else 和 switch 语句。
```
static func buildIf(_ s: String?) -> String {
	s ?? ""
}

//这可以让我们把上面的例子重写为：
@StringBuilder func greet(planet: String) -> String {
	"Hello, Planet"
	if let idx = planets.firstIndex(of: planet) {
		" "
		idx
	}
	"!"
}
greet(planet: "Earth") // Hello, Planet 2!
greet(planet: "Sun") // Hello, Planet!

//现在我们向 builder 函数中引入了条件，重写的版本开始变得更有意思了：
func greet_rewritten(planet: String) -> String {
	let v0 = "Hello, Planet"
	var v1: String?
	if let idx = planets.firstIndex(of: planet) {
		v1 = StringBuilder.buildBlock(
			StringBuilder.buildExpression(" "),
			StringBuilder.buildExpression(idx)
		)
	}
	let v2 = StringBuilder.buildIf(v1)
	return StringBuilder.buildBlock(v0, v2)
}
```

想要更好地处理 greeting 函数中输入无效的情况，我们可以添加 if … else 支持。为此，我们必须实现 buildEither(first:) 和 buildEither(second:) 这两个 builder 方法。buildEither(first:) 会被第一个分支的结果调用，而 buildEither(second:) 将被第二个分支的结果调用

## 循环

在 result builder 函数中，还有一种我们可以使用的语句，那就是 for...in 循环。为了使用 for 循环，我们需要为 builder 类型添加 buildArray 方法：
```
static func buildArray(_ components: [String]) -> String {
	components.joined(separator: "")
}

```

## 其他构建方法
buildLimitedAvailability，它可以用来转换带有可用性限定的上下文 (比如 if #available(...)) 中的结果类型。
buildFinalResult：顾名思义，这个方法会对整个 result builder 函数的结果调用一次。这个方法的一个应用场景，是用来隐藏那些被用来构建中间结果的内部类型，让它们不必暴露到外界。



