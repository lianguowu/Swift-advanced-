# 内建集合类型4-Range

范围代表的是两个值的区间，它由上下边界进行定义。你可以通过 ..< 来创建一个不包含上边界的半开范围，或者使用 ... 创建同时包含上下边界的闭合范围  
```
“let singleDigitNumbers = 0..<10
Array(singleDigitNumbers) // [0, 1, 2, 3, 4, 5, 6, 7, 8, 9]
// 包含 "z"
let lowercaseLetters = Character("a")...Character("z")”
```
最常用的两种类型是 Range (由 ..< 创建的半开范围) 和 ClosedRange (由 ... 创建的闭合范围)。两者都有一个 Bound 的泛型参数：对于 Bound 的唯一的要求是它必须遵守 Comparable 协议。举例来说，上面的 lowercaseLetters 表达式的类型是 ClosedRange<Character>。  

Comparable协议，其中定义了< 、<=、>=、>四个比较运算符，以及扩展了...、..<区间运算符。


## 可数范围
范围看起来很自然地会是一个序列或者集合类型。并且你确实可以遍历一个整数范围，或像集合类型那样对待它：
但并不是所有的范围都可以使用这种方式。比如，编译器不允许我们遍历一个 Character 的范围：
// 错误：'Character' 类型没有实现 'Strideable' 协议。  

Strideable协议的类型，理论上都是连续的，在单一维度上的值能够被抵消和测量，支持整型，浮点型和索引值， Strideable协议继承于 Comparable。给类扩展方法，并遵守 Strideable协议, advancedBy()和distanceTo()这俩个方法是必须实现，否则会报错，告诉你没按套路来。 Strideable 协议的关联对象是 Stride,遵守 SignedNumberType协议。


让 Range 满足集合类型协议是有条件的，条件是它的元素需要满足 Strideable 协议 (你可以通过增加偏移来从一个元素移动到另一个)，并且步长 (stride step) 是整数，为了能遍历范围，它必须是可数的。对于可数范围 (满足了那些约束)，因为对于 Stride 有整数类型这样一个约束，所以有效的边界包括整数和指针类型，但不能是浮点数类型。如果你想要对连续的浮点数值进行迭代的话，你可以通过使用 stride(from:to:by) 和 stride(from:through:by) 方法来创建序列用以迭代。


在条件化实现协议被 Swift 4.2 引入之前，标准库为了区分可数和不可数范围，包含了两个具体类型，分别名为 CountableRange 和 CountableClosedRange。为了向后兼容，这两个名字还是作为类型别名而存在着

## 范围表达式
所有这五种范围都满足 RangeExpression 协议。这个协议内容很简单，所以我们可以将它列举到书里。首先，它允许我们询问某个元素是否被包括在该范围中。其次，给定一个集合类型，它能够计算出表达式所指定的完整的 Range：
```
public protocol RangeExpression {
	associatedtype Bound: Comparable
	func contains(_ element: Bound) -> Bool
	func relative<C>(to collection: C) -> Range<Bound>
		where C: Collection, Self.Bound == C.Index
}

```
对于下界缺失的部分范围，relative(to:) 方法会把集合类型的 startIndex 作为范围下界。对于上界缺失的部分范围，同“样，它会使用 endIndex 作为上界。这样一来，部分范围就能使集合切片的语法变得相当紧凑：
```
let arr = [1,2,3,4]
arr[2...] // [3, 4]
arr[..<1] // [1]
arr[1...2] // [2, 3]
```
这种写法能够正常工作，是因为 Collection 协议里对应的下标操作符声明中，所接收的是一个实现了 RangeExpression 的类型，而不是上述五个具体的范围类型中的某一个。你甚至还可以将两个边界都省略掉，这样将会得到表示整个集合的一个切片：
```
arr[...] // [1, 2, 3, 4]
type(of: arr) // Array<Int>”

```
这其实是 Swift 标准库中的一个特殊实现，这种无界范围还不是有效的 RangeExpression 类型，不过它应该会在今后遵守 RangeExpression 协议。








