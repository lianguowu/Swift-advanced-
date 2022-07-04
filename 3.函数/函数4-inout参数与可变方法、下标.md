# 函数4-inout参数与可变方法、下标

## inout参数与可变方法
Swift 中用在 inout 参数前面的 & 可能会给你一种这是在传递引用的错觉。但事实并非如此，inout 做的事情是传值，然后复制回来，并不是传递引用。
对于 inout 参数，你只能传递左值，因为右值是不能被修改的。

实际上，所有的下标操作符 (包括那些你自定义的)，只要它们同时定义了 get 和 set 方法，也都可以作为 inout 参数。类似的，只要同时定义了 get 和 set 方法，属性也可以作为左值。

你可以在嵌套函数中使用 inout 参数，Swift 会保证你的使用是安全的。比如说，你可以定义一个嵌套函数 (使用 func 或者使用闭包表达式)，然后安全地改变一个 inout 的参数。“不过，你不能够让这个 inout 参数逃逸。

说到不安全 (unsafe) 的函数，你应该小心 & 的另一种含义：把一个函数参数转换为一个不安全指针。


## 下标 subscript

例如：用 dictionary[key]这样的方式在字典查找元素。这些下标很像函数和属性的混合体，只不过它们使用了特殊的语法。
比如，数组默认有两个下标操作，一个用来访问单个元素，另一个用来返回一个切片 (更精确地说，它们是被定义在 Collection 协议中的)：
```
let fibs = [0, 1, 1, 2, 3, 5]
let first = fibs[0] // 0
fibs[1..<3] // [1, 1]
```

下标进阶-subscript；修改latitude的值
```
var japan: [String: Any] = [
	"name": "Japan",
	"capital": "Tokyo",
	"population": 126_740_000,
	"coordinates": [
		"latitude": 35.0,
		"longitude": 139.0
	]
]

// 错误：类型 'Any' 没有下标成员
japan["coordinate"]?["latitude"] = 36.0
// 错误：不能对不可变表达式赋值
(japan["coordinates"] as? [String: Double])?["coordinate"] = 36.0

```


通过拓展dictionary的下标方法subscript 
```
extension Dictionary {
    subscript<Result>(key: Key, as type: Result.Type) -> Result? {
        get {
            return self[key] as? Result
        }
        set {
            // 如果传入 nil, 就删除现存的值。
            guard let value = newValue else {
                self[key] = nil
                return
            }
            // 如果类型不匹配，就忽略掉。
            guard let value2 = value as? Value else {
                return
            }
            self[key] = value2
        }
    }
}

```



