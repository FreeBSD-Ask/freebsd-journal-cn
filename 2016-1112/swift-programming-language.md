# Swift 编程语言

- 原文：[Swift Programming Language](https://freebsdfoundation.org/wp-content/uploads/2016/12/Swift-Language.pdf)
- 作者：**Steve Wills**

Swift 是 Apple 推出的新通用编程语言，于 2014 年 Apple 年度 WWDC 活动上发布，2015 年以 Apache 许可证 2.0 版开源。它旨在替代 Objective C，但更简洁、更安全，也确已如此。Swift 主要面向 iOS 应用开发，但它是一门完整的通用系统编程语言，可用于多种任务。它正在快速发展，可移植到多种操作系统。Swift 全面支持 Unicode。它易学、好用、运行快。Swift 支持命令式与面向对象编程、泛型、扩展、try/throw/catch 错误处理、动态分派、可扩展编程与晚绑定。

## 安全性

内存自动管理，无需手动分配或释放，避免相关错误。作为使编程更安全的设计目标之一，Swift 在很大程度上不向开发者暴露指针，但需要时也可使用。它要求变量使用前先声明，避免脚本语言中有时出现的隐式声明问题。为简洁起见，类型可推断；为安全与清晰起见，也可显式声明。为安全起见，数据类型严格强制；为灵活起见，类型也可轻松转换。

对符号（变量、函数、类等）支持五种访问控制：

- private——仅在直接作用域内可访问
- fileprivate——同一文件内的任何代码都可访问
- internal——同一模块内的任何代码都可访问
- public——可从任何模块访问
- open——可在模块外被继承（仅限类与方法）

这些访问控制不受继承影响，允许显式控制代码与数据的使用位置，避免意外与棘手的故障排查。

## 在 FreeBSD 上构建

首先确保你有 FreeBSD 11 与最新的 FreeBSD Ports 树。然后：

```sh
cd /usr/ports/lang/swift ; make install
```

Swift 使用定制版的 llvm、clang、lldb、cmark、llbuild，因此构建要花些时间。之后应可运行 `swift` 命令并获得交互式提示符。你也可以保存 Swift 文件并调用 Swift 解释器。

## 缺什么？

Swift 既可解释也可编译，但目前 FreeBSD 上只有解释器可用。撰文时，Port lang/swift 为 2.2.1 版，而 Swift 最新版本为 3.0.1。更新 Port 与启用编译器的工作正在进行。Swift 教程往往围绕编写 iOS 应用展开，但这些应用所需的库属于 iOS，目前只在 iOS 上可用。因此 FreeBSD 上只有 Swift 核心语言可用。

## Hello, World!

像任何编程语言一样，我们在 Swift 中写的第一个程序就是打印“Hello, World”。Swift 中只需：

```swift
print("Hello, World!")
```

在解释器中输入时如下：

```swift
(swift) print("Hello, World!")
Hello, World!
```

Swift 行末不要求分号，但允许：

```swift
(swift) print("Hello"); print("World!");
Hello
World!
```

这让代码易于阅读，同时允许开发者想多简洁就多简洁。

## 变量与常量

变量必须在使用前声明：

```swift
(swift) var message = "Hello, World!" // message : String = "Hello, World!"
(swift) print(message)
Hello, World!
```

再次，简洁性得以体现，因为允许开发者跳过变量类型声明——Swift 已自动判断出 message 是字符串。当然，在需要或希望处也可声明变量类型：

```swift
(swift) var message2: String = "Hello"
// message2 : String = "Hello"
```

糟糕，我们忘了消息的一部分。补上：

```swift
(swift) // 补上消息剩余部分
(swift) message2 += ", World!"
(swift) print(message2)
Hello, World!
```

简洁性再次体现——简单的字符串拼接即可。我们也可指定常量：

```swift
(swift) let message3 = "Hello" // message3 : String = "Hello"
```

常量不能修改：

```swift
(swift) message3 += ", World!"
<REPL Input>:1:9: error: left side of mutating operator isn't mutable: 'message3' is a 'let' constant
<REPL Input>:1:1: note: change 'let' to 'var' to make it mutable
let message3 = "Hello"
var
```

这服务于安全目标——允许程序员在必要时把某些值标为只读。错误消息清楚指出想改时该如何下手。

## 类型

Swift 提供所有 C 与 Objective-C 类型，包括 Int、Double、Float、Bool 与处理文本的 string。

```swift
(swift) var four: Float = 4
// four : Float = 4.0
(swift) var five: Double = 5
// five : Double = 5.0
(swift) let six: Float = 6
// six : Float = 6.0
```

类型强制再次服务于安全：

```swift
(swift) let result = four + five
<REPL Input>:1:19: error: binary operator '+' cannot be applied to operands of type 'Float' and 'Double'
<REPL Input>:1:19: note: overloads for '+' exist with these partially matching parameter lists: (Float, Float), (Double, Double)
let result = four + five
               ~~~~ ^ ~~~~
```

但类型也可轻松转换，灵活方便：

```swift
(swift) let result2 = four + Float(five)
// result2 : Float = 9.0
```

Swift 让数字更易读：

```swift
(swift) let million = 1_000_000
// million : Int = 1000000
(swift) print(million)
1000000
```

简洁性再次体现——Swift 允许一次为多个变量赋值：

```swift
(swift) var (foo, bar, baz) = (10, 100, 1000)
// (foo, bar, baz) : (Int, Int, Int) = (10, 100, 1000)
```

此外，Swift 提供 Array、Set、Dictionary、Ranges、Tuples：

```swift
(swift) let myarray: Array<String> = ["first", "second", "third"]
// myarray : Array<String> = ["first", "second", "third"]
(swift) let secondarray = [1: "Alice", 2: "Bob", 3: "Chuck"]
// secondarray : [Int : String] = [2: "Bob", 3: "Chuck", 1: "Alice"]
(swift) let People: Dictionary<Int, String> = [1: "Alice", 2: "Bob", 3: "Chuck"]
// People : Dictionary<Int, String> = [2: "Bob", 3: "Chuck", 1: "Alice"]
(swift) let range = 0...5
// range : Range<Int> = Range(0..<6)
(swift) let range2 = 0..<10
// range2 : Range<Int> = Range(0..<10)
```

Swift 还引入了创新概念 Optionals，即值可能存在也可能不存在的变量：

```swift
(swift) var daytime: Bool? = true
// daytime : Bool? = Optional(true)
(swift) var mayContainNumber = "404"
// mayContainNumber : String = "404"
(swift) var actualNumber = Int(mayContainNumber)
// actualNumber : Int? = Optional(404)
```

Swift 推断 actualNumber 可能含也可能不含数字，而非要求我们指定。用“!”操作符取值：

```swift
(swift) print("actualNumber has an integer value of \(actualNumber!).")
actualNumber has an integer value of 404.
```

但为安全起见，使用前必须检查值是否存在：

```swift
(swift) actualNumber = nil
(swift) print("actualNumber has an integer value of \(actualNumber!).")
fatal error: unexpectedly found nil while unwrapping an Optional value
```

因此必须这样写：

```swift
(swift) if actualNumber != nil {
    print("actualNumber has an integer value of \(actualNumber!).")
}
(swift) actualNumber = Int(mayContainNumber)
(swift) if actualNumber != nil {
    print("actualNumber has an integer value of \(actualNumber!).")
}
actualNumber has an integer value of 404.
```

## 类型安全

与 C 不同，赋值操作符不返回值，因此这里给出漂亮错误消息，再次保护我们免于低级笔误：

```swift
(swift) if foo = bar {
    print("fail")
<REPL Input>:1:8: error: use of '=' in a boolean context, did you mean '=='?
if foo = bar {
       ==
```

而且贴心建议了必要修改。类似地，整数不能用作布尔值：

```swift
(swift) let i = 1
// i : Int = 1
(swift) if i {
    print("foo")
<REPL Input>:1:4: error: type 'Int' does not conform to protocol 'BooleanType'
if i {
```

加上变量使用前必须声明的要求，这些约束消除了若干类常见错误。

## 控制流

Swift 有常见的控制流机制：条件用 if 与 switch，循环用 for 与 while：

```swift
var value = 10
// 注意：if 与 while 即使只有一条语句也必须用 {}
if value > 0 {
    print("true")
}
// 注意：没有 break 语句，case 默认不贯穿，但可用 "fallthrough" 贯穿
var weekday = "Thursday"
switch weekday {
case "Monday":
    print("They call it stormy Monday")
case "Friday":
    print("TGIF")
default:
    print("Is it Friday yet?")
}
// 注意：使用 Range 与 "in"
for value in 1...10 {
    print("\(value) * 9 = \(value * 9)")
}
while value > 0 {
    print("Not yet")
    value -= 1
}
```

## 函数

Swift 函数使用具名参数，使代码更易读、易维护，不过在 Swift 2.x 中第一个名字必须省略。函数可返回也可不返回值。看这段示例代码：

```swift
// 向用户打招呼的函数
// 返回问候语
func hello(user: String) -> String {
    let result = "Hello, " + user + "!"  // result 保持不变，故为常量
    return result
}

/* 再次向用户打招呼的函数
   （注释两种风格均可） */
func helloAgain(user: String) -> String {
    return "Hello again, " + user + "!"
}

// 接受额外 bool 参数的函数
func hello(user: String, seen: Bool) -> String {
    if seen {
        return helloAgain(user)
    } else {
        return hello(user)
    }
}

// 向多位用户打招呼，不返回值
func helloAll(users: [String]) {
    guard users.count > 0 else { return }  // 确保传入了用户，否则下行会报错
    users.forEach { user in
        print(hello(user, seen: false))
    }
}

let users = ["Alice", "Bob", "Chuck"]
var seenUsers = ["Alice": true, "Bob": false]
seenUsers["Chuck"] = false
var nousers: Array<String> = Array()
helloAll(users)
print(hello("Alice", seen: true))
helloAll(nousers)
```

将其保存为 multihello.swift 并运行：

```sh
$ swift multihello.swift
Hello, Alice!
Hello, Bob!
Hello, Chuck!
Hello again, Alice!
```

通过“...”支持可变参数函数：

```swift
func sum(values: Int...) -> Int {
    var sum = 0
    for value in values {
        sum += value
    }
    return sum
}
sum(10, 20, 30)
```

支持函数嵌套：

```swift
func returnValue() -> Int {
    var i = 1
    func add() {
        i += 1
    }
    add()
    add()
    add()
    return i
}
returnValue()
// （返回 4）
```

函数是一等类型，可从其他函数返回：

```swift
func returnAdder() -> ((Int) -> Int) {
    func adder(number: Int) -> Int {
        return 1 + number
    }
    return adder
}
var adder = returnAdder()
adder(10)
```

## 类

Swift 也是面向对象的。类可以这样声明：

```swift
class Building {
    var numberOfFloors = 0

    init(numberOfFloors: Int) {
        self.numberOfFloors = numberOfFloors
    }

    func getDescription() -> String {
        return "A building with \(numberOfFloors) floors."
    }
}

class PaintedHouse: Building {
    var color: String

    init(floors: Int, color: String) {
        self.color = color
        super.init(numberOfFloors: floors)
    }

    override func getDescription() -> String {
        return "A \(color) building with \(numberOfFloors) floors."
    }
}

var house = Building(numberOfFloors: 2)
var houseDescription = house.getDescription()
let myHouse = PaintedHouse(floors: 3, color: "blue")
print(myHouse.getDescription())
```

## 结语

Swift 还有更多内容，鼓励大家试一试。网上有许多学习 Swift 的好资源：

- <https://swift.org/>
- <https://developer.apple.com/swift/>
- <https://developer.apple.com/library/content/documentation/Swift/Conceptual/Swift_Programming_Language/GuidedTour.html>

甚至有几个允许你通过浏览器在线运行代码的网站：

- <https://www.weheartswift.com/swift-sandbox/>
- <https://swiftlang.ng.bluemix.net/#/repl>

---

**STEVE WILLS** 是居住在北卡罗来纳州的丈夫与父亲。他是 FreeBSD Ports 提交者，专注 Ruby 等编程语言。
