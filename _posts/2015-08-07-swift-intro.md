---
layout: post
title: Swift 入门参考
author: kaifeng
date: 2015-08-07
categories: lang
description: 关于Apple Swift语言的入门参考。
---

Swift 是苹果推出的一门新语言，它的语法仍然是在 C 语言基础上的延伸，溶入了面向对象语言和脚本语言的特征，还有一点 Objective-C 的影子。

# 基本类型

Swift 是强数据类型的，也是类型安全的，常量和变量都有着明确的类型。大多数情况下，编译器可根据赋值推导出类型，从而帮助省略代码书写，使得代码比较简洁。Swift 中的类型包括基本类型(Int, Double等)，还包括类、结构和枚举。类型在 Swift 中称作 Type。

定义常量与变量使用 let 和 var 关键字。let 用于定义常量，var 则定义变量，形式为：
```
var v : Type = ..  // 定义一个变量
let c : Type = ..  // 定义一个常量
```

以字典为例，Swift 使用 let 和 var 来定义可修改字典和不可修改字典，这与 Objective-C 中以 NSDictionary 和 NSMutableDictionary 分开实现是不同的。

Swift 的基本类型包括 Int, Float, Double, Bool, String 等。

整型类型其实包含了 Int8, Int16, Int32, Int64 四种类型，在没有明确位宽要求的情况下，使用 Int 即可，编译器会根据机器位宽自动选择。对于整型数值的类型推导，Swift 优先选择 Int 型。

浮点类型包括了32位的 Float 和64位的 Double 两种类型，同样，在没有明确位宽要求的情况下，推荐使用 Double。对于浮点数值的类型推导，Swift 优先选择 Double 型。如果表达式中有整型和浮点型，则推导类型为 Double型。

布尔型 Bool，只能取 true 和 false，Swift 的类型安全不允许像 C 语言那样将整型转为布尔型。

字符串类型 String 是一组 Character 的有序集合，包括了若干 Unicode 字符。

# 运算符

Swift 的运算符完全涵盖了 C 的运算符，包括：基本运算，一元运算，复合运算，男较运算，关系运算，移位运算符，位域运算等等。

Swift 在运算符上还做了一些扩展。最大的变化是增加了范围运算符 `..<` 和 `...`

- `..<` 是半开区间范围，表示 [start, end)，如 `1 ..< 3` 表示 1, 2。
- `...` 是闭区间范围，表示 [start, end]，如 `1 ... 3` 表示 1, 2, 3。

范围运算常用于循环语句和 switch 语句。

Swift 扩展了求模运算符 %，可以对浮点做模运算，如 8 % 2.5 = 0.5。

赋值运算符的语义发生了变化，不再像 C 语言那样返回数值，也即 `(a = b) != c` 这样的语法不再支持。

# Optional

Optional 类型由 `Type?` 定义，它的引入是为了处理有值/无值问题。一个 Optional 变量要么无值（缺失某类型的值），要么有某种值。无值即 unset，或称为 nil。
Swift 的 Optional 类型可以支持任意类型，包括类、结构和枚举类型。Objective-C 的 nil 只能用于 class 这种引用类型，表达的是空指针的含义，含义上与 Swift 也有不同。

可以给 Optional 变量赋 nil，但不能给非 Optional 变量赋 nil。定义Optional 变量但不设初值的，默认被设为nil。

判断一个 Optional 变量是否有值，可以用 if 等条件判断语句，如确定有值，则可在变量后加!来提取类型值，这称作强制取值（forced unwrapping），例如：
```
if someOptional != nil {
  println(“\(someOptional!)”)
}
```

如果对一个无值的 Optional 变量取值，程序将崩溃。

上述语句还可以可以写作：
```
if let arg = someOptional {
  println(“\(arg)”)
}
```

这种用法称作 Optional binding。若语句内需要修改 arg，也可以使用 var 关键字。

如果已知某个 Optional 变量有值，可以不必执行检查而直接取值，这类变量在定义时，类型后面添加!标识，使用时表现为普通变量而非 Optional 变量。
```
let possibleValue : String? = “have value”
let assumedValue : String! = “implicitly unwrapped optional”
```

在使用 assumedValue 时，无需再加!取值，这种方式称作隐式取值。对于这类变量，仍然可以用 if 等条件判断其是否有值。

可以理解为，隐式 Optional 其实仍是 Optional 类型，只是引用时自动加上了取值操作。

Swift 提供了 nil 合并运算符，形式为：`( a ?? b )`

若 Optional 变量 a 有值，则取 a 的值，否则返回非Optional变量 b 作为默认值。

# 数组、字典与集合

数组、字典和集合是 Swift 提供的集合类型。

数组是一组相同元素类型的有序值集合，类型以 Array 表示。数组的定义使用`Array<Type>`或者使用更简单的`[Type]`语法定义，比如创建空数组：`Array<Int>()` 或者 `[Int]()`。

```
let a : Array<Int> = [1, 2, 3]
```

由于 Swift 可以进行类型推导，因此上面的语句也可以写为：
```
let a = [1, 2, 3]
```

数组提供了一些属性，如获取数组元素数量的 count 属性，判断数组是否为空的 isEmpty 属性。

数组元素的引用使用下标语法，下标从0开始，支持范围运算符，如 `a[0]` 或者 `a[0...1]`。下标语法可用作元素赋值，但不可用于添加或插入新元素。

添加元素使用 append 方法。还可通过 += 添加另一个数组，这涉及到了运算符重载的概念（操作符两端须为同一类型，也即需数组类型，而非元素类型）。

其它常用方法还有 insert, removeAtIndex, reverse 等。

遍历数组可使用 for-in 语法，或者通过 enumerate 全局函数获取带下标及值的元组：
```
    for (index, value) in enumerate(array) {
      ...
    }
```

字典类型以 Dictionary 表示，字典的定义与使用与数组类似：
- 定义字典：`Dictionary<Key, Value>`或`[Key:Value]`
- 使用下标语法访问。

由于字典里某个键值不一定有对应的值，因此总是返回 Optional 类型。

集合类型以 Set 表示，一组相同元素类型的无序值集合，与数组类似。
集合并没有简写语法，定义集合：
```
var s : Set<Type> = [value1, value2, value3]
```

# 元组

元组是由多个值组成的复合值，其成员可以是不同类型。元组的引入，使一小组相关数据的传递变得十分方便。元组非常适合函数需要返回多个值的情况，而这在其它语言里需要构造新的对象来实现。

元组的定义语法为 ( Type1, Type2, Type3 )，如
```
let aTuple = (1, “Yes”)
```

默认情况下，元组成员的访问以从0开始的下标方式，如 aTuple.0 取出 1，aTuple.1 取出 ”Yes”。可以为元组定义成员描述，如：
```
let aTuple = (aNumber:1, aString:”Yes”)
```

此时可由 aTuple.aNumber, aTuple.aString 访问指定成员。

# 控制流

Swift 中的代码控制与 C 基本上相同，有部分书写和语义的变化。

if, for, while, do-while, switch 等条件判断语句，条件判断部分不再需要括号。如：
```
if x == y {
}
```

花括号是不可省略的，即使只有一条语句，这与 C 不同。

for-in 语法，用于遍历容器：
```
for index in 1 ..< 10 {} // 使用了范围运算符
for var i = 0; i < 3; i++ {} // 这个语法仍然支持
```

Swift 对 switch 语句作了较大改动，case 的条件判断更加强大。case 条件支持字符串的直接比较，支持范围运算符，支持多个值的匹配，例如：
```
let x = 10
switch x {
  case 1:
    ...
  case 2, 3:  // 匹配多值，以逗号分隔
    ...
  case 10 ... 20: // 范围匹配
    ...
  default:
    ...
}
```

要注意的是，case 必须涵盖 switch 条件的所有可能性，当不可能涵盖全部范围时，必须提供 default 分支。同时，Swift 不再支持 C 语言的 fall through 行为，当 case 体执行完成后，直接跳出，而无需显式书写 break 语句。若确实需要 fall through 行为，使用 fallthrough 关键字：
```
switch x {
  case 1:
    fallthrough
  default:
}
```

由于 Swift 引入了元组，所以 switch 也支持元组匹配，并且可以使用值绑定（Value Binding）的语法获取元组成员值进行比较，更可使用 where 语句扩展条件判断：

```
let point = (1, 1)
switch point {
  case (0, 0):  // 比较元组
    ...
  case (_, 0):   // 第一个成员可以是任意值
    ...
  case (let x, 1): // 值绑定语法，case 体内可使用 x 的值。
    ...
  case let (x, y) where x == y: // where 语句
    ...
  default:
    ...
}
```

let (x, y) 是 (let x, let y) 的简写法，若对元组所有成员做值绑定时，可将 let 提到外面。同样的，在需要的情况下，也可以使用 var。

其它的控制语句还有 continue, break, return 等，这些与 C 的语义相同。


# 枚举

Swift 对枚举进行了较大改动，功能也变得更加强大了。

定义一个枚举类型的语法为：
```
enum SomeEnumeration {
  case North
  case South
  case East
  case West
}
```

case 定义的是枚举的成员值（也即成员），一条 case 可以定义多个成员，以逗号分隔。与 C 不同的是，枚举成员并没有默认的整数值。

每个枚举成员可以有一个关联值，并且不同成员可以用不同的类型，如：
```
enum Barcode {
  case UPCA(Int, Int, Int, Int)
  case QRCode(String)
}
```

关联值在定义枚举变量时提供，比如：
```
var code = Barcode.UPCA(8, 8059, 51226, 3)
switch code {
  case .UPCA(let a, let b, let c, let d): // 可简写为case let .UPCA(a, b, c, d)
    println(...)
  case .QRCode(let a):
    ...
}
```

枚举成员也可以有默认值，这在 Swift 里称为原始值(raw values)，原始值必须是同一种类型，如：
```
enum ASCIIControlCharacter : Character {
  case Tab = "\t"
  case LineFeed = "\n"
  case CarriageReturn = "\r"
}
```

使用整数作为raw values时，其行为与 C 相似，数值会递增。raw values并不是关联值，两者并不相同。Swift 提供了从根据原始值获取枚举成员的方法：
```
let possiblePlanet = Planet(rawValue:7)
```

因为 rawValue 有可能无对应的成员，因此只能返回 Optional 类型。

Optional 实质上也一种枚举类型，也即：
```
enum Optional<T> {
  case None
  case Some<T>
}
```

取值操作等价于下面代码：
```
var y = x! 等价于
switch x {
  case Some(let value): y = value
  case None: // raise exception
}
```

# 函数与闭包

Swift 的函数语法不再像 Objective-C 那样独特了，更像是类 C 的写法，一个函数的一般定义形式如下：
```
func funcname(externalName internalName:Type) -> ReturnType {
  ...
}
```

func 关键字用于定义一个函数，-> 语法用于指定返回类型。比较怪异的是参数部分，不过如果有 Objective-C 的基础，也很容易理解。

一个参数有内部参数名和外部参数名（可省略）两种，内部参数名是函数体内使用的参数名，如果不填写外部参数名，那定义出的函数和其它语言没什么不同，假如定义了这样一个函数：
```
func aFunction(s1: String) -> String
```

其调用语法即：
```
let aString = aFunc(“a string”)
```

Objective-C 的命名方式使得参数更容易阅读，Swift 函数外部参数名的使用也是为了沿用这一优点。

假如定义函数形式为：
```
func aFunction(sourceString src : String, withPostString post : String) -> String
```

在定义了外部参数名的情况下，函数调用者必须书写外部参数名：
```
aFunction(sourceString: “a string”, withPostString: “post”)
```

是不是很像 Objective-C 的写法。如果不需要外部参数名（比如，第一个参数的描述常常不需要外部参数名），那么可以用下划线来表示忽略。当内部名与外部名相同时，可用#name同时生成内外部参数名。

有默认值的入参会自动生成外部入参名，默认值入参必须在入参列表的最后。变参以...表示，一个函数只能有一个，且作为最后一个入参，如存在默认值入参，需置于其后。变参实际上是数组，函数内可用for-in遍历。

函数入参默认是常量，函数内不可修改。参数名前添加 var 可使入参为变量。但是，函数体内的改动只在函数体内有效，若要在函数返回后，修改仍生效，需要使用 inout 关键字声明入参。inout 入参需以 &param 的语法传入。

函数也是一种类型，函数类型由入参和返回值构成，类似其它语言原型的概念。例如 (Int, Int) -> Int。但一般语言不将返回值作为函数定义的一部分。

可以定义函数类型的变量，也可用作其它函数的入参类型，返回值类型。函数类型函数还可以嵌套定义。运算符也是函数。

闭包与 Objective-C 里的 block 对应，与其它语言的 lambda 类似，在本质上就是函数。Swift 对闭包做了一些优化：

- 从上下文推导参数和返回值类型
- 单个表达式闭包可隐式返回
- 简写入参名称
- trailing closure语法：如果函数最后一个入参是闭包，可将其写在函数定义外，供方便阅读。

与嵌套函数相比，闭包有时更加有用。

无返回值的函数其实也是有返回值的，只是返回一个空的元组，即()。当无返回值函数用作闭包时，若不写返回值部分，其写法与元组是一样的，要显式地写作 `func funcName(parameters) -> ()`

闭包表达式是指内联的简单闭包，其一般形式为：
```
{ (parameters) -> return type in
    statements
}
```

比如调用 sorted 函数：
```
sorted(array, closure)
```

以闭包表达式形式写作：
```
sorted(array, { (s1:String, s2:String) -> Bool in return s1 > s2 })
```

闭包表达式可使用常量入参，变量入参，inout 入参，变参，元组，不使用默认值入参。大多数情况下，闭包表达式的入参和返回值是可以推导的，可以省略类型，不需要写完整形式。闭包只有一句表达式时，可以省略 return。闭包有默认入参，从$0, $1依次类推。简写参数名：$0, $1, ... 可在闭包里使用。

因为运算符也是函数，因此 sorted 所接受的闭包，可以是一个运算符。

当闭包表达式是函数最后一个入参时，可使用称作 trailing closure 的尾部闭包写法，这个写法只是为了方便阅读，例如：
```
reversed = sorted(array) { $0 > $1 }
```

如果仅有闭包表达式一个入参，也可省略函数名后的()。

闭包内保持对变量的引用（闭包定义时的上下文），这称为值捕获(Capturing values)。


# 结构与类

Swift 的三大基本构造块是类、结构和枚举，它们的相同点是都可以有属性，函数和初始化函数。而结构与类更为接近，两者都可以定义属性、函数、下标、初始化函数，可扩展/实现协议。但只有类支持继承、类型转换、反初始化、引用计数。

结构与枚举是值类型，传递时使用值拷贝。事实上，Swift 所有的基本类型都是值类型，其底层是以结构实现的。String, Array, Dictionary 也是以结构实现的，这与 Objective-C 对应的 NSString, NSArray, NSDictionary 不同。虽然集合类型采用值拷贝会有性能上的疑虑，但是 Swift 做过优化，仅在必要时才发生拷贝，应该也是采用了与 COW(Copy On Write) 类似的技术。

类是引用类型，使用引用计数来管理。对于引用类型，使用===和!==来比较两个对象是否为同一实例（也即地址相同），而传统的==意义为“相等”。相等是指两个实例被认为是相等的，可以重载运算符来执行比较，如果内容相同或者满足设计者认为相同的条件，即可认为相同。

值类型与引用类型的差异，同 C 语言的值传递与指针传递的概念是相同的，也同样是容易犯错的地方，特别是作为函数入参的时候。

既然类的功能如此强大，那么如何选择使用结构还是类呢？选择结构主要有下列考虑：
1. 主要目的是封装一些简单的值。
2. 值拷贝即可。
3. 结构内的属性也只用值类型。
4. 不需要继承属性或者方法。

如果不满足需要，则使用类。


# 属性

简言之，属性就是属于某个类型的成员变量或常量。属性可分为存储属性、计算属性、懒惰属性和类型属性四种类型。

存储属性也就是普通的成员变量/常量，使用 var 或 let 定义，属性的引用使用点号。如：
```
struct SomeStruct {
  let maxValue : Int = 3
  var current : String = 1
}
var a = SomeStruct()
a.current = 2
```

Swift 还可以直接设置属性的属性，如 mode.res.width。

计算属性并不储值，它的值是根据其它值计算得来。它只是提供了 getter 和 setter 函数。函数名分别为 get 和 set。set 可以省略入参，默认入参名为 newValue。set 是可选的，不提供 set 则该计算属性只读。对于只读的计算属性，语法上还可省略 get 的书写。
```
struct Circle {
  var radius = 1.0
  var area {
    return M_PI * radius * radius  // 省略了 get 书写
  }
}
```

懒惰属性用 lazy 关键字修辞，如：
```
lazy var aLazyVar = SomeHugeClass()
```

懒惰属性在第一次访问时才会被初始化，适合用于较为耗时、开销较大的初始化工作，这也是延迟初始化的策略。只有变量才可以指定为懒惰属性。

Swift 支持属性观察者，它可在属性值发生变化时得到通知（调用）。存储属性都可以有属性观察者，而计算属性由于已经显式得到 set 和 get 调用，因此不需要属性观察者。懒惰属性不支持属性观察者。属性观察者常用于更新界面。

属性观察者涉及 willSet 和 didSet 两个函数，当属性值将被改变时，willSet 先被调用（不写入参默认为 newValue），在属性值被修改后，didSet  得到调用（不写入参默认为 oldValue）。要注意的是，在初始化函数中对属性的设置不会触发 willSet 和 didSet 调用。
```
struct Circle {
  var radius = 1.0 {
    willSet {
      println("will set radius to \(newValue)")
    }
    didSet {
      println("old radius is \(oldValue)")
    }
  }
}

var a = Circle()
a.radius = 2.0
```

类型属性即属于类型而非实例的属性，使用 static 关键字修辞的属性即为类型属性，它使用类型名而非实例变量名引用。

枚举不能有存储属性，但可以有计算属性。


# 方法

方法是与特定类型相关的函数，在其它面向对象语言里，只有类才可以定义方法，而在 Swift 中，结构、枚举和类都可以定义自己的方法。

方法在本质上也是函数，也用 func 关键字定义，它的参数可以有内部参数名和外部参数名。不过方法与函数在有些默认行为上有差异。Swift 为第一个入参生成默认的本地参数名，为其它入参生成默认的外部名和内部名，也即调用时不需要为第一个入参写外部参数名，这是为了兼容 Objective-C 里的写法。若想修改该行为，可自行定义内外部名。

类型方法与类型属性一样，以 static 关键字修辞。在类型方法内，self 指本类型而非实例，扩展来说，在类型方法里，不加 self 默认是引用属于类型的属性或方法。

由于结构和枚举是值类型，因此它们的方法不能修改成员变量，如有需要修改，需要在 func 关键字前加上 mutating 关键字。

# 下标

下标是一种方法，结构、枚举和类都可以定义下标，定义下标函数后，可以用类似索引数组的方式来访问数据。下标以 subscript 关键字定义：
```
struct SomeStruct {
  subscript (index:Int) -> Int {
    get {
      ..
    }
    set(newValue) {
      ..
    }
  }
}
var a = SomeStruct()
a[0] = ..
```

与 setter/getter 一样，如果只支持读操作，可省略 get, set 书写。

下标方法可有多个入参和返回值，入参类型也是任意的。可以根据需要提供多个下标方法，Swift 以入参个数和类型推导使用哪一个，这个特性称作下标重载（Subscript Overloading）。


# 类

类是面向对象语言的核心，三大特性的继承和多态都是由类来支持的。Swift 并没有统一的基类，不指定父类的类即是基类。A是B的子类写作：
```
class A : B {
}
```

由于继承关系，子类得到了父类所定义的一些方法和属性，子类通过 super 关键字引用父类的属性和方法。子类若需重载父类方法，要在方法或属性定义前显式地冠以 override 关键字，这种方式可避免非重载意图的重名。如：
```
override func description() -> String{}
```

required 关键字可要求子类必须实现指定内容，final 关键字则用于阻止子类重载。

继承是多态的基础，多态少不了类型之间的转换，Swift 提供了 as 和 is 两个运算符来处理。

is 用于检查实例是否为属于某个类型（包括父类型），比如判断 Car 是否为 Vehicle：
```
class Vehicle { }
class Car : Vehicle { }

let car : Car = Car()

if car is Vehicle {}   // true
if car is Car {}       //true
```

as 用于处理向下转换，由于向下转是不安全的，因此有 as? 和 as! 两种形式，不确定转换成功的使用 as? 得到一个 Optional，只有在明确是某个类型时才使用 as!。
```
var v : Vehicle = Car()
if let car = v as? Car {
    println("is a car")
}
```

Swift 提供了 AnyObject 和 Any 两个特殊类型来处理无具体类型。AnyObject 可表示任意类的实例，Any 表示任意类型的实例，也包括函数类型。

事实上，AnyObject是一个协议，主要用于兼容现有的基于 Objective-C 的API。因为 Objective-C 在很多情况下并没有明确的类型指定，Cocoa API 常用 AnyObject 表示一个对象。


# 初始化与反初始化

为了保证初始化的完整性，Swift 对类的初始化定义了若干规则和要求，相比其它语言而言，初始化流程要复杂许多，结构的初始化则很简单。与 Objective-C 不同，Swift 的初始化函数不返回自身。类还可以定义反初始化函数。

初始化函数使用 init 作为函数名，且不书写 func 关键字。初始化函数可以使用入参，Swift 为每个入参自动生成外部入参名，即外部入参名是不可缺省的。确实不想要外部名的，手动在外部名位置以下划线来忽略该默认行为。
```
class Circle {
    var radius : Double
    init(radius:Double) {
        self.radius = radius
    }
}

let circle = Circle(radius:1.0)
println(“\(circle.radius)")
```

如果某属性可能无值，初始化阶段也不设置值，则定义为 Optional 类型，初始化阶段默认被赋为 nil。初始化函数也可以给常量属性赋值，一旦初始化结束后，不可再修改。

如果基类的每个属性都提供了默认值且未提供初始化函数，Swift 会提供一个默认的初始化函数。对于结构，就算属性没有默认值，也提供一个默认的初始化函数。默认的初始化函数的入参按照成员定义的顺序依次排列。
```
struct Resolution {
    let width : Int
    let height : Int
}
let via = Resolution(width:640, height:480)
```

初始化函数可以调用其它初始化函数来完成部分初始化工作，这称作初始化代理。
值类型与类在规则上有不同，因为值类型没有继承关系，规则简单。值类型在初始化函数里（也仅能在初始化函数里）用 self.init 来引用其它初始化函数即可。

类所有的属性（包括继承的）必须在初始化过程中赋值。对于类，Swift 提供了指定初始化函数和方便初始化函数来确保所有属性都已有初值。指定初始化函数是类的主要初始化函数，它初始化该类的所有属性并调用合适的父类初始化函数。指定初始化函数数量较少，通常只有一个。

方便初始化函数是次要的支持性的初始化函数，它调用指定初始化函数，提供某些默认参数值。方便初始化函数是可选的，指定初始化函数至少要有一个（可以继承）。
指定初始化函数语法与类型的一样：
```
init(parameters) {
}
```

方便初始化函数使用 convenience 关键字修辞：
```
convenience init(parameters) {
}
```

类的初始化代理有三条规则，这些规则保证了一个类实例初始化的完整性：
1. 指定初始化函数必须调用直接父类的指定初始化函数
2. 方便初始化函数必须调用本类另一个初始化函数
3. 方便初始化函数最终必须调用一个指定初始化函数

类初始化过程大体上分为两个阶段，第一阶段的初始化给本类引入的存储属性赋予初值，第二阶段则进一步修改存储属性至实例可用。这个过程保证了访问属性前已被赋予初值，并防止其它初始化函数意外修改了属性值。针对两阶段初始化，Swift 编译器会执行下面的一些检查：
1. 指定初始化函数必须确保本类所有属性都有初值，然后才可调用父类指定初始化函数。
2. 指定初始化函数必须在调用父类指定初始化函数后才可对继承属性赋值。
3. 方便初始化函数必须调用另一个初始化函数后才可以对属性赋值。
4. 初始化函数不能调用实例方法，读取实例属性值，或引用 self，这些要到第一阶段初始化结束后。

与 Objective-C 不同，Swift 子类默认不继承父类的初始化函数。当子类有与父类同名的初始化函数时，需用 override 关键字。但在特定情况下，父类的初始化函数会被继承。若子类引入的属性都有默认值，有如下2条规则：
1. 若未定义指定初始化函数，则自动继承父类所有指定初始化函数。
2. 若子类提供了父类所有指定初始化函数的实现（通过规则1的继承，或是提供重载实现），则自动继承父类所有的方便初始化函数。

即使子类添加新的方便初始化函数，上述规则也生效。

针对初始化可能失败的情况，Swift 也支持可失败的初始化函数，以 init? 定义。若在初始化过程中失败，返回 nil 即可。但是对于类来说，只有在所有属性已赋值，且初始化代理已发生之后，才能触发初始化失败（返回 nil）。初始化函数事实上并不返回值，正常的初始化并不需要写 return 语句。可以重载父类的可失败初始化函数并提供不失败的实现，但不失败的初始化函数不允许向可失败的初始化函数发出代理。

以 required 关键字修饰的初始化函数为强制实现，子类必须实现。子类也冠以required 而非 override 关键字。如果满足继承初始化函数的条件，则可以不重载。

类可以定义一个反初始化函数，与 init 相对，反初始化函数使用 deinit，无参数也无括号。在实例被释放前被调用。不允许手工调用 deinit，超类的 deinit 会被子类继承，在子类的 deinit 后会被自动调用，编码时无需关心。


# 访问控制

Swift 提供了三种不同的访问级别，默认的访问类型是internal：
- public 对外部模块可见，一般为框架接口。
- internal 对所在模块可见，对外部模块不可见。
- private 仅源文件内可见。

模块是指代码发布的单元、框架或者程序，在其它模块通过 import 引入的部分。Xcode 的每个 build target 都是一个模块。源文件是指构成模块的单个文件。

权限的控制也涉及到相关参数类型的权限问题，例如：
- 类型的访问类型影响其成员的访问类型。若定义类型为 private，则其所有成员默认为 private，若定义类型为 public 或 internal，其所有成员默认为internal。
- 元组的访问类型自动降级为成员中的最低访问权限。
- 函数的访问类型默认也取入参及返回值中最低访问权限。
- 枚举成员的访问类型自动使用枚举的访问类型，且不可更改。
- 子类不能用有高于父类的访问权限。

# 协议

协议也是一个类型，使用 protocol 关键字定义，与 Objective-C 接近。遵循协议的语法与继承相同，和父类写在一起即可，不像 Objective-C 有区分。
```
protocol SomeProtocol {
  ...
}
```

协议可要求遵循者有特定的属性、实例方法、类型方法、操作符、下标等。必需实现的接口用 required 关键字限定，可选需求用 optional 关键字限定。

协议可以继承一个或多个协议并添加更多的要求。还可限制协议只允许类采用
```
protocol SomeClassOnlyProtocol: class SomeInheritedProtocol
```

协议可以组合在一起，常用作入参：
```
protocol <SomeProtocol, AnotherProtocol>
```

要检查一个实例是否遵循某种协议，仍然是用 is, as 运算符来处理。

# 扩展

扩展使用 extension 关键字定义，对应 Objective-C 的 Category 概念，功能上也与 Category 相似，不同处在于 Swift 的扩展没有名字。

扩展可以：
- 添加计算属性，类型计算属性
- 定义实例方法和类型方法
- 提供新的初始化函数
- 定义下标
- 定义和使用新的嵌套类型
- 让一个类型采用某种协议，如
```
extension SomeType : SomeProtocol, AnotherProtocol {
  ...
}
```

只能添加函数和计算属性，不允许添加新的属性。

# 泛型

Swift 的泛型的概念和语法与其它语言一样，以尖括号表示类型参数，如：
```
class Tree<T>
```

Swift 的泛型支持泛型函数，以及类、结构、枚举的泛型。Array和Dictionary其实都是泛型。

不过 Swift 可以限定泛型类型的范围：
```
func someFunction<T:SomeClass, U:SomeProtocol> (someT:T, someU:U)
```

还可加 where 条件语句：
```
func allItemsMatch<C1:Container, C2:Containter where C1.ItemType == C2.ItemType, C1.ItemType:Equatable> (...) {...}
```

# ARC

ARC 的概念在 Objective-C 就已经有了，强引用、弱引用、引用计数等概念都是一样的。在 Swift 中，weak 必须定义为 Optional 类型。如果引用总是有值，可使用 unowned 关键字，但引用对象被释放时，ARC 不会将其置为 nil。

由于 Swift 引入了闭包，如果一个对象有闭包属性，而闭包也引用了对象内的资源，那么会产生强引用的环路，解决方法是使用闭包捕获列表（closure capture list)。其语法为：
```
var someClosure : () = {
  [unowned self, weak delegate = self.delegate]
  ...
}
```
