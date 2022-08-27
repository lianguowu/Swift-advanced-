# Swift的函数派发机制

函数派发就是程序判断使用哪种途径去调用一个函数的机制. 每次函数被调用时都会被触发, 但你又不会太留意的一个东西. 了解派发机制对于写出高性能的代码来说很有必要, 而且也能够解释很多 Swift 里"奇怪"的行为.

编译型语言有三种基础的函数派发方式: 直接派发(Direct Dispatch), 函数表派发(Table Dispatch) 和 消息机制派发(Message Dispatch), 下面我会仔细讲解这几种方式. 大多数语言都会支持一到两种, Java 默认使用函数表派发, 但你可以通过 final 修饰符修改成直接派发. C++ 默认使用直接派发, 但可以通过加上 virtual 修饰符来改成函数表派发. 而 Objective-C 则总是使用消息机制派发, 但允许开发者使用 C 直接派发来获取性能的提高. 这样的方式非常好, 但也给很多开发者带来了困扰,


## 派发方式

程序派发的目的是为了告诉 CPU 需要被调用的函数在哪里, 在我们深入 Swift 派发机制之前, 先来了解一下这三种派发方式, 以及每种方式在动态性和性能之间的取舍.


### 直接派发 (Direct Dispatch) 

直接派发是最快的, 不止是因为需要调用的指令集会更少, 并且编译器还能够有很大的优化空间, 例如函数内联(inline)等, 但这不在这篇博客的讨论范围. 直接派发也有人称为静态调用.

然而, 对于编程来说直接调用也是最大的局限, 而且因为缺乏动态性所以没办法支持继承.


### 函数表派发 (Table Dispatch )

函数表派发是编译型语言实现动态行为最常见的实现方式. 

函数表使用了一个数组来存储类声明的每一个函数的指针. 大部分语言把这个称为 "virtual table"(虚函数表), Swift 里称为 "witness table". 每一个类都会维护一个函数表, 里面记录着类所有的函数, 如果父类函数被 override 的话, 表里面只会保存被 override 之后的函数. 一个子类新添加的函数, 都会被插入到这个数组的最后. 运行时会根据这一个表去决定实际要被调用的函数.

举个例子, 看看下面两个类:
```
class ParentClass {
    func method1() {}
    func method2() {}
}
class ChildClass: ParentClass {
    override func method2() {}
    func method3() {}
}
```
在这个情况下, 编译器会创建两个函数表, 一个是 ParentClass的vtable, 另一个是 ChildClass的vtable

![image](https://upload-images.jianshu.io/upload_images/546449-0d5c0de84a2a6200.png)

当一个函数被调用时, 会经历下面的几个过程:
```
let obj = ChildClass()
obj.method2()
```
1. 读取对象 ChildClass (0xB00) 的函数表.
2. 读取函数指针的索引. 在这里, method2 的索引是1(偏移量), 也就是 0xB00 + 1.
3. 跳到 0x222 (函数指针指向 0x222) 方法method2

查表是一种简单, 易实现, 而且性能可预知的方式. 然而, 这种派发方式比起直接派发还是慢一点. 从字节码角度来看, 多了两次读和一次跳转, 由此带来了性能的损耗. 另一个慢的原因在于编译器可能会由于函数内执行的任务导致无法优化. (如果函数带有副作用的话)

这种基于数组的实现, 缺陷在于函数表无法拓展. 子类会在虚数函数表的最后插入新的函数, 没有位置可以让 extension 安全地插入函数. 这篇提案很详细地描述了这么做的局限.


### 消息机制派发 (Message Dispatch )

消息机制是调用函数最动态的方式. 也是 Cocoa 的基石, 这样的机制催生了 KVO, UIAppearence 和 CoreData 等功能. 这种运作方式的关键在于开发者可以在运行时改变函数的行为. 不止可以通过 swizzling 来改变, 甚至可以用 isa-swizzling 修改对象的继承关系, 可以在面向对象的基础上实现自定义派发.

举个例子, 看看下面两个类:
```
class ParentClass {
    dynamic func method1() {}
    dynamic func method2() {}
}
class ChildClass: ParentClass {
    override func method2() {}
    dynamic func method3() {}
}
```

Swift 会用树来构建这种继承关系:

![image](https://upload-images.jianshu.io/upload_images/546449-0231dc315d6be4f1.png)

这张图很好地展示了 Swift 如何使用树来构建类和子类.

当一个消息被派发, 运行时会顺着类的继承关系向上查找应该被调用的函数. 如果你觉得这样做效率很低, 它确实很低! 然而, 只要缓存建立了起来, 这个查找过程就会通过缓存来把性能提高到和函数表派发一样快. 但这只是消息机制的原理.


## Swift的函数派发机制

那么, 到底 Swift 是怎么派发的呢? 我没能找到一个很简明扼要的答案, 但这里有四个选择具体派发方式的因素存在:

1. 声明的位置
2. 引用的类型决定了派发的方式
3. 特定的行为
4. 编译器显式地优化

在解释这些因素之前, 我有必要说清楚, Swift 没有在文档里具体写明什么时候会使用函数表什么时候使用消息机制. 唯一的承诺是使用 dynamic 修饰的时候会通过 Objective-C 的运行时进行消息机制派发. 下面我写的所有东西, 都只是我在 Swift 3.0 里测试出来的结果, 并且很可能在之后的版本更新里进行修改.

### 声明的位置

在 Swift 里, 一个函数有两个可以声明的位置: 类型声明的作用域, 和 extension. 根据声明类型的不同, 也会有不同的派发方式.
```
class MyClass {
    func mainMethod() {}
}
extension MyClass {
    func extensionMethod() {}
}
```
上面的例子里, mainMethod 会使用函数表派发, 而 extensionMethod 则会使用直接派发. 当我第一次发现这件事情的时候觉得很意外, 直觉上这两个函数的声明方式并没有那么大的差异. 下面是我根据类型, 声明位置总结出来的函数派发方式的表格.

![image](https://upload-images.jianshu.io/upload_images/546449-1f7d7f847bb3b049.png)


总结起来有这么几点:

* 值类型总是会使用直接派发, 简单易懂
* 协议里声明的, 并且带有默认实现的函数(被要求的方法)会使用函数表进行派发
* 而协议和类的 extension 都会使用直接派发
* NSObject 的 extension 会使用消息机制进行派发
* NSObject 声明作用域里的函数都会使用函数表进行派发.

### 引用的类型决定了派发的方式

引用的类型决定了派发的方式. 这很显而易见, 但也是决定性的差异. 一个比较常见的疑惑, 发生在一个协议拓展和类型拓展同时实现了同一个函数的时候.
```
protocol MyProtocol {
}
struct MyStruct: MyProtocol {
}
extension MyStruct {
    func extensionMethod() {
        print("结构体")
    }
}
extension MyProtocol {
    func extensionMethod() {
        print("协议")
    }
}
 
let myStruct = MyStruct()
let proto: MyProtocol = myStruct
 
myStruct.extensionMethod() // -> “结构体”
proto.extensionMethod() // -> “协议”
```

刚接触 Swift 的人可能会认为 proto.extensionMethod() 调用的是结构体里的实现. 但是, 引用的类型决定了派发的方式, 协议拓展里的函数会使用直接调用. 如果把 extensionMethod 的声明移动到协议的声明位置的话, 则会使用函数表派发, 最终就会调用结构体里的实现. 并且要记得, 如果两种声明方式都使用了直接派发的话, 基于直接派发的运作方式, 我们不可能实现预想的 override 行为. 这对于很多从 Objective-C 过渡过来的开发者是反直觉的.

### 指定派发方式 

Swift 有一些修饰符可以指定派发方式.

1. final
final 允许类里面的函数使用直接派发. 这个修饰符会让函数失去动态性. 任何函数都可以使用这个修饰符, 就算是 extension 里本来就是直接派发的函数. 这也会让 Objective-C 的运行时获取不到这个函数, 不会生成相应的 selector.

2. dynamic
dynamic 可以让类里面的函数使用消息机制派发. 使用 dynamic, 必须导入 Foundation 框架, 里面包括了 NSObject 和 Objective-C 的运行时. dynamic 可以让声明在 extension 里面的函数能够被 override. dynamic 可以用在所有 NSObject 的子类和 Swift 的原声类.

3. @objc & @nonobjc
@objc 和 @nonobjc 显式地声明了一个函数是否能被 Objective-C 的运行时捕获到. 使用 @objc 的典型例子就是给 selector 一个命名空间 @objc(abc_methodName), 让这个函数可以被 Objective-C 的运行时调用. @nonobjc 会改变派发的方式, 可以用来禁止消息机制派发这个函数, 不让这个函数注册到 Objective-C 的运行时里. 我不确定这跟 final 有什么区别, 因为从使用场景来说也几乎一样. 我个人来说更喜欢 final, 因为意图更加明显.

译者注: 我个人感觉, 这这主要是为了跟 Objective-C 兼容用的, final 等原生关键词, 是让 Swift 写服务端之类的代码的时候可以有原生的关键词可以使用.

4. final @objc
可以在标记为 final 的同时, 也使用 @objc 来让函数可以使用消息机制派发. 这么做的结果就是, 调用函数的时候会使用直接派发, 但也会在 Objective-C 的运行时里注册响应的 selector. 函数可以响应 perform(selector:) 以及别的 Objective-C 特性, 但在直接调用时又可以有直接派发的性能.

5. @inline
Swift 也支持 @inline, 告诉编译器可以使用直接派发. 


### 编译器显式地优化

Swift 会尽最大能力去优化函数派发的方式. 例如, 如果你有一个函数从来没有 override, Swift 就会检测并且在可能的情况下使用直接派发. 这个优化大多数情况下都表现得很好, 但对于使用了 target / action 模式的 Cocoa 开发者就不那么友好了. 例如:
```
override func viewDidLoad() {
    super.viewDidLoad()
    navigationItem.rightBarButtonItem = UIBarButtonItem(
        title: "登录", style: .plain, target: nil,
        action: #selector(ViewController.signInAction)
    )
}
private func signInAction() {}
```
这里编译器会抛出一个错误: Argument of '#selector' refers to a method that is not exposed to Objective-C (Objective-C 无法获取 #selector 指定的函数). 你如果记得 Swift 会把这个函数优化为直接派发的话, 就能理解这件事情了. 这里修复的方式很简单: 加上 @objc 或者 dynamic 就可以保证 Objective-C 的运行时可以获取到函数了. 这种类型的错误也会发生在UIAppearance 上, 依赖于 proxy 和 NSInvocation 的代码.

另一个需要注意的是, 如果你没有使用 dynamic 修饰的话, 这个优化会默认让 KVO 失效. 如果一个属性绑定了 KVO 的话, 而这个属性的 getter 和 setter 会被优化为直接派发, 代码依旧可以通过编译, 不过动态生成的 KVO 函数就不会被触发.

可以自己选择编译器的优化

打开“全模块优化” (whole module optimization)。在这个编译模式下，当前模块的所有文件都会在一起进行优化。除了泛型特化，全模块优化也会开启其他重要的优化策略。比如说，优化器会识别出那些在整个模块中没有子类的 internal class。因为 internal 修饰符确保了这个类不会被模块外看到，这也就意味着编译器可以把这个类的所有方法由动态派发替换成静态派发。

将泛型函数标记为 @inlinable 以便其他模块使用。这个标记会把函数体也作为模块接口的一部分进行导出，其他模块引用它时，优化器就也能看到这些代码。在这种情况下，当调用者进行编译时，虽然包含泛型函数的模块已经被编译过了，但是优化器仍然能生成一个该函数的特化版本并把它放到调用模块中。
