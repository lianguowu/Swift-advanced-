# 内建集合类型3-Set

标准库中第三种主要的集合类型是集合 Set。集合是一组无序的元素，每个元素只会出现一次。你可以将集合想像为一个只存储了键而没有存储值的字典。和 Dictionary 一样，Set 也是通过哈希表实现的，并拥有类似的性能特性和要求。测试集合中是否包含某个元素是一个常数时间的操作，和字典中的键一样，集合中的元素也必须满足 Hashable。

如果你需要高效地测试某个元素是否存在于序列中并且元素的顺序不重要时，使用集合是更好的选择 另外，当你需要保证序列中不出现重复元素时，也可以使用集合。

Set 遵守 ExpressibleByArrayLiteral 协议，也就是说，我们可以用数组字面量的方式初始化一个集合


## 集合代数

1. 补集 subtracting
2. 交集 intersection
3. 合集 formUnion

几乎所有的集合操作都有不可变版本以及可变版本的形式，后一种都以 form 开头。想要了解更多的集合操作，可以看看 SetAlgebra 协议。


## 索引集合和字符集合

1. IndexSet 索引集合
IndexSet 表示了一个由正整数组成的集合。当然，你可以用 Set<Int> 来做这件事，但是 IndexSet 更加高效，因为它内部使用了一组范围列表进行实现。使用 Set<Int> 的话，根据选中的个数不同，最多可能会要存储 1000 个元素。而 IndexSet 不太一样，它会存储连续的范围，也就是说，在选取前 500 行的情况下，IndexSet 里其实只存储了选择的首位和末位两个整数值。  
不过，作为 IndexSet 的用户，你不需要关心内部实现，所有这一切都隐藏在我们所熟知的 SetAlgebra 和 Collection 接口之下

```
var indices = IndexSet()
indices.insert(integersIn: 1..<5)

```

2. CharacterSet 字符合集
CharacterSet 是一个高效的存储 Unicode 编码点 (code point) 的集合。它经常被用来检查一个特定字符串是否只包含某个字符子集 (比如字母数字 alphanumerics 或者数字 decimalDigits) 中的字符。不过，和 IndexSet 有所不同，CharacterSet 并不是一个集合类型。它的名字，CharacterSet，是从 Objective-C 导入时生成的，在 Swift 中它也并不兼容 Swift 的 Character 类型。可能 UnicodeScalarSet 会是更好的名字。


## 在闭包中使用集合

我们如果想要为 Sequence 写一个扩展，来获取序列中所有的唯一元素，我们只需要将这些元素放到一个 Set 里，然后返回这个集合的内容就行了。不过，因为集合并没有定义顺序，所以这么做是不稳定的，输入的元素的顺序在结果中可能会不一致。为了解决这个问题，我们可以创建一个扩展来解决这个问题，在扩展方法内部我们还是使用 Set 来验证唯一性,这个方法让我们可以找到序列中的所有不重复的元素，并且通过元素必须满足 Hashable 这个约束来维持它们原来的顺序。

```
extension Sequence where Element: Hashable {
	func unique() -> [Element] {
		var seen: Set<Element> = []

		return filter { element in
			if seen.contains(element) {
				return false
			} else {
				seen.insert(element)
				return true
			}
		}
	}
}

```










