---
title: Swift 学习笔记 - 从 Property Wrapper 视角探索 State 与 Binding 如何工作
date: 2025-05-12 23:30:52
tags:
  - Swift
  - iOS
  - 编程语言
categories:
  - 技术
---

## 背景

基本学完 Swift 语法后，就开始跟随 [Apple 关于 Swift UI 的官方教程](https://developer.apple.com/tutorials/swiftui-concepts) 来学习 Swift 如何应用在 iOS App 开发中。

在 [Driving Changes in your UI with State and Bindings](https://developer.apple.com/tutorials/swiftui-concepts/driving-changes-in-your-ui-with-state-and-bindings) 这一小节中，教程首次引入了 Swift UI 中关于状态的概念。

教程中使用了简单的代码来介绍 `@State` 和 `@Binding` 属性，以及如何在工程中使用这两个属性:

```swift
import Foundation

// 省略部分代码 ...
struct RecipeEditorConfig {
    var recipe = Recipe.emptyRecipe()
    var shouldSaveChanges = false
    var isPresented = false
    // 省略部分代码 ...
}
```

```swift
import SwiftUI


struct ContentListView: View {
    // 省略部分代码 ...
    @State private var recipeEditorConfig = RecipeEditorConfig()

    var body: some View {
        // 省略部分代码 ...
                        RecipeEditor(config: $recipeEditorConfig)
        // 省略部分代码 ...
    }
    
    // 省略部分代码 ...
}
```

```swift
import SwiftUI

struct RecipeEditor: View {
    @Binding var config: RecipeEditorConfig
    
    var body: some View {
        NavigationStack {
            RecipeEditorForm(config: $config)
            // 省略部分代码 ...
        }
    }
    // 省略部分代码 ...
}
```

由于之前也接触过其他的前端语言，例如 Dart 或是 JavaScript，自然地就会好奇：Swift 这样的语法设计下，是怎么实现状态的刷新的呢？

## 一些猜测

比较直接的猜测就是，传入的 `$config` 类似闭包语法，实则是传递了一个 `{ config in return config }` 闭包给 `RecipeEditor`

这看起来好像解答了为什么 `RecipeEditor` 可以直接修改 `ContentListView` 中 `recipeEditorConfig` 的值，但实则有几个关键的问题没有解答：

1. `@State` 存在的意义是在状态变更的时候通知渲染层进行重绘，类似 `setState` 方法，如果直接传入变量闭包，虽然可以修改，但是如何通知 UI 层重绘？
2. 如果传入的是闭包，闭包是如何赋值给 `RecipeEditorConfig` 这样的变量的？为什么没有 `init` 函数依然可以完成这样的赋值？
3. 在 `RecipeEditor` 中，是可以直接修改 `config` 的值的，例如直接给 `config` 赋新值，如果原本的引用通过闭包传入后解析出来赋值给 `config`，那这样修改后不就丢失了对原先的对象的引用吗？

几番提问下来，基本可以肯定应该不是直接通过闭包传递的，而是 `@State` 和 `@Binding` 做了更复杂的处理

而具体做了哪些处理，则要从 Property Wrapper 开始讲起

## Swift 中的 Property Wrapper

在一篇关于 `@State` 和 `@Binding` 的问答中，我了解到，这两个属性并非宏 (至少不是库中按照 Macro 描述编写的那种宏)，而是受语言支持的两个 Property Wrapper

关于 Property Wrapper 的内容，在 [Swift Evolution - Property Wrapper](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0258-property-wrappers.md) 中有提到整个 Property Wrapper 设计的背景、提案和应用场景

简单说来，Property Wrapper 可以分为 3 步：

1. 编写由 `@propertyWrapper` 标识的结构体，这个结构体用于存储包装属性所需要的所有额外信息以及可以提供的方法（例如一个 lazy 包装属性可能会想要存储当前变量是否已经被初始化过、初始化后的值等），这个结构体会暴露 `wrappedValue` 和 `projectValue` 两个属性供外部访问
2. (猜想): 编译器识别到 `@propertyWrapper`，将后续结构体对应的 AST 转化为包含该结构体定义的，更复杂的宏定义
3. 编译器识别到 `@MyPropertyWrapper`，将后续对应的声明 AST 转化为更复杂的表达式，包括一个私有 `MyPropertyWrapper<Value>` 结构体对象和对应的访问器

### PropertyWrapper 结构体

Property Wrapper 这一方案的提出意图解决的问题就是，当访问一个变量时，能够在不增加越来越多且复杂的编译器属性的同时，支持将通用的、可复用的额外逻辑作用在这个访问过程中

例如，关键词 `lazy` 本质上就是在试图解决这类问题中的一个: 当访问一个变量时，先判断是否初始化，如果初始化则返回初始化后的值，否则进行初始化

最初的想法可能就是，写一些重复的代码 (boilerplate code) 来实现这个功能:

```swift
// Irrelevant code omitted
struct MyClass {
    var _myDefaultValue: Int = 123
    var _myValue: Int? = nil
    var myValue: Int {
        get {
            if _myValue == nil {
                _myValue = _myDefaultValue
            }
            guard let v = _myvalue else {
                raise fatalError()
            }
            return v
        }
        set {
            _myValue = newValue
        }
    }
}
```

但是如果每一个需要用到这个逻辑的地方都要这么些，未免有些复杂

考虑到宏可以扩展代码，也许可以定义一个宏 `@Lazy` 来展开这些内容，从而避免每次都重复编写

```swift
// Irrelevant code omitted
struct MyClass {
    // 也就是，将
    @Lazy var myValue: Int = 123
    // 转换为
    var _myValue: Int? = nil
    var myValue: Int {
        get {
            if _myValue == nil {
                _myValue = 123
            }
            guard let v = _myvalue else {
                raise fatalError()
            }
            return v
        }
        set {
            _myValue = newValue
        }
    }
}
```

然而，具体的宏想想就已经非常复杂了

也就是说，如果想要直接用宏来实现给属性访问添加额外逻辑这一能力，虽然可以做到，但那样的话似乎更多的关注会在如何实现宏本身而不是实际的逻辑上了 (Make 的困境幻视)

那么，从语言设计的角度考虑，是不是可以让开发者更关注于具体逻辑的实现而隐藏宏转换的细节呢？类似 CMake 那样，用一套规范来约束开发者对逻辑的定义，再在这一规范之上构建编译器属性，使得能够将具体逻辑转换成某种编译器能识别的宏，从而开发者只需要遵循规范来定义逻辑，而不再需要编写具体的宏定义了

虽然不知道 Swift 具体是怎么实现的，但我想 Property Wrapper 的出现大概率是相似的思路

在 Property Wrapper 的描述中，Swift 约定了一个编译器属性 `@propertyWrapper`，其可以用于修饰自定义的结构体，结构体的一般结构约定如下：

```swift
@propertyWrapper
struct MyPropertyWrapper {
    private var myPropertyUnderlyValue: MyValueType

    init(myPropertyUnderlyValue: MyValueType) {
        self.myPropertyUnderlyValue = myPropertyUnderlyValue
    }

    var wrappedValue: MyValueType {
        get {
            // Add logic when get
            myPropertyUnderlyValue
        }
        set {
            // Add logic when set
            myPropertyUnderlyValue = newValue
        }
    }

    var projectedValue: SomeProjectedType {
        get {
            self
        }
    }
}
```

我想，之所以这么设计，是因为通过 `struct` 可以提供一个统一的抽象语法树节点来描述整个额外逻辑所需要的所有内容。想象如果不使用 `struct` 结构来包装，那为了实现 `@Lazy` 的能力，需要找一个地方写 `_myValue` 变量，另找一个地方写 `myValue` 需要的 `getter` 和 `setter` 逻辑，并且编译器要能够直到这些分散的表达式节点都是 `@Lazy` 的一部分，这想想就不好实现

通过 `struct` 的包装，虽然使得实际值的存储多了一层结构体的封装，但简化了编译器的实现，并且也统一了可能的表现形式

现在，编译器只需要在将这个结构体转换为一个宏，其在标记了对应属性的声明处进行如下操作即可:

1. 将原先的声明替换为一个 `_originalVarName: MyPropertyWrapper` 的值
2. 声明 `var originalVarName: MyPropertyWrapper { get, set }` 访问器，通过 `MyPropertyWrapper.wrappedValue` 进行访问
3. 声明 `var $originalVarName: SomeProjectedType { get, set }` 访问器，通过 `MyPropertyWrapper.projectedValue` 进行访问
4. 生成 synthesized initializer (如果必须)

看起来就很好实现了，不是吗？

最后生成出来的代码大致可以理解成下面这样:

```swift
// Original class declaration
// struct MyClass {
//     @MyPropertyWrapper var myPropertyValue: MyValueType
// }

// Converted class declaration
struct MyClass {
    private var _myPropertyValue: MyPropertyWrapper
    
    var myPropertyValue: MyValueType {
        get {
            _myPropertyValue.wrappedValue
        }
        set {
            _myPropertyValue.wrappedValue = newValue
        }
    }

    var $myPropertyValue: SomeProjectedType {
        get {
            _myPropertyValue.projectedValue
        }
        set {
            _myPropertyValue.projectedValue = newValue
        }
    }

    init(myPropertyValue: MyValueType) {
        _myPropertyValue = MyPropertyWrapper(myPropertyUnderlyValue: myPropertyValue)
    }
}
```

### projectedValue 和 wrappedValue

在 `MyPropertyWrapper` 的实现中，`wrappedValue` 的作用比较显而易见: 提供了一个对实际存储内容的访问器，在访问器中实现了需要对目标属性添加的额外访问逻辑。而通过将 `myPropertyValue` 改为对 `_myPropertyValue.wrappedValue` 的访问器，可以满足在当前类内对目标属性的访问经过所需要的额外逻辑

然而，只暴露 `wrappedValue` 往往不能满足需求，因为在类内访问 `self.myPropertyValue` 会直接经过 `_myPropertyValue.wrappedValue.get` 解析到 `_myPropertyValue.myPropertyUnderlyValue` 这一实际的底层存储值，这对变量传递就不太友好了: 如果我试图将这个属性传递到其他的函数中，它就会作为实际存储值进行传递，而不会再传递对应的访问器了！

虽然 `@autoclosure` 看似可以解决这个问题，但是这需要被调函数进行主动适配，显然不能够满足所有的情况。我猜想，Swift 语言的开发团队就是为此类情况而额外支持了一种访问器: `projectedValue`

`projectedValue` 可以由开发者自由指定需要返回的对象，并且在类内以 `$myPropertyValue` 的形式暴露，一个简单的设计就是:

```swift
var projectedValue: MyPropertyWrapper {
    self
}
```

提供一个对 `MyPropertyWrapper` 实例对象的 `get` 访问器，从而当使用 `$myPropertyValue` 进行传参时，依然可以通过传入参数的 `wrappedValue` 来访问并且修改底层存储值

## State 的魔法

在 SwiftUI 中，`@State` 就是依赖了 Property Wrapper 这一语言特性

很显然，对于显示内容状态的存储，完全符合了 Property Wrapper 的适用场景:

- View 的数据需要以成员变量形式存储并且加以访问和修改 (对象为类的成员变量)
- 对 View 所引用数据的修改需要通知 UI 框架进行重绘 (变量需要在被赋值时添加额外的通知逻辑)
- 数据需要能够沿着控件树向下传递，并且子树也能够对存储数据进行修改 (包含额外逻辑的变量访问器需要能够作为参数传递)
- 对于所有的 View 需要的数据，虽然类型不同，但依赖的逻辑相同 (需要多处复用的额外逻辑)

### 可能的 State 结构体

要实现对状态的存储和访问，`@State` 包装主要需要做的就是在变量被赋值时，触发 UI 更新逻辑

因此，一个可能的简化版的 `@State` 大概会是如下的样子:

```swift
@propertyWrapper
struct State<T> {
    private state: T

    var wrappedValue: T {
        get {
            state
        }
        set {
            state = newValue
            // call UI update logic
        }
    }

    var projectedValue: State<T> {
        get {
            self
        }
    }

    init(state: T) {
        self.state = state
    }
}
```

### 展开后的 @State 代码

参考 Property Wrapper 展开的逻辑，对于使用了 `@State` 的成员变量，其简化后的展开代码大概会是下面的样子:

```swift
class MyView: View {
    // Original declaration
    // @State var counter: Int = 5

    // Expanded declaration
    var _counter: State<Int> = State(5)

    var counter: Int {
        get {
            _counter.wrappedValue
        }
        set {
            _counter.wrappedValue = newValue
        }
    }

    var $counter: State<Int> {
        get {
            _counter.projectedValue
        }
    }

    var body: some View {
        // View hierarchy
    }
}
```

对于当前的 View 来说，直接使用例如 `counter = counter + 1` 可以更新状态值并且触发 UI 更新逻辑；而通过传递 `$counter` 则可以允许其他 View 通过 `State<Int>` 提供的接口来更新状态值并且触发 UI 更新逻辑

### 使用 Binding 作为 projectedValue

这看起来已经解决了大半的问题，但还有一些情况需要考虑: 例如有时可能会需要将已有的属性外再添加一些逻辑后再传递给子控件

对于这种情况，如果直接传递 `$state` 所对应的 `State<Int>` 对象，显然是不满足要求的，而如果只是传递一个 `@autoclosure`，又没办法实现 `set` 能力

因此，可以想到的就是创建一个额外的类，来包含需要新增的逻辑，并提供访问接口 (就像另一个 Property Wrapper 那样):

```swift
class WierdCounter {
    let get: () -> Int
    let set: (Int) -> Void

    init(get: () -> Int, set: (Int) -> Void) {
        self.get = get
        self.set = set
    }

    var wrappedValue: Int {
        get {
            self.get()
        }
        nonmutating set {
            self.set(newValue)
        }
    }
}

class MyView: View {
    @State var counter: Int = 5

    var wrappedCounter: WierdCounter = WierdCounter {
        counter
    }, set: { newValue in
        counter = newValue * 2
    }

    var body: some View {
        MyAnotherView(counter: wrappedCounter)
    }
}
```

此时，更进一步地思考，目前子控件想要访问父控件传入的状态的话，多少还是有些复杂的: 不仅有时候传入 `State<T>` 有时候传入额外的类，访问实际状态的时候还要经过额外的一层访问器去访问

相比 Swift 设计团队也是这么想的，于是他们首先解决了第一个问题: 先让传入的状态参数类型统一

这其实还比较好实现，不难发现，`WierdCounter` 和 `State<T>` 其实有着相似的结构，而且他们的核心目标都是需要访问 `counter.wrappedValue`，因此不妨在库中就提供一个类似的类，例如 `MyBinding<T>`:

```swift
struct MyBinding<T> {
    let get: () -> T
    let set: (T) -> Void

    var wrappedValue: T {
        get {
            self.get()
        }
        nonmutating set {
            self.set(newValue)
        }
    }

    init(@escaping get: () -> T, @escaping set: (T) -> Void) {
        self.get = get
        self.set = set
    }
}
```

对应的，`State<T>` 的设计也进行相应的调整：

```swift
@propertyWrapper
struct State<T> {
    private state: T

    var wrappedValue: T {
        get {
            state
        }
        set {
            state = newValue
            // call UI update logic
        }
    }

    var projectedValue: MyBinding<T> {
        MyBinding {
            self.wrappedValue
        }, set { newValue in
            self.wrappedValue = newValue
        }
    }

    init(state: T) {
        self.state = state
    }
}
```

这样，展开后的 `MyView` 类也会有相应的变化:

```swift
class MyView: View {
    // Original declaration
    // @State var counter: Int = 5

    // Expanded declaration
    var _counter: State<Int> = State(5)

    var counter: Int {
        get {
            _counter.wrappedValue
        }
        set {
            _counter.wrappedValue = newValue
        }
    }

    var $counter: MyBinding<Int> {
        _counter.projectedValue
    }

    var body: some View {
        // View hierarchy
    }
}
```

统一了状态传参的类型，接下来，对于第二个问题，就是 `@Binding` 展现魔法的时刻了

## Binding 的魔法

其实读到这里，难免会有一种不由自主的冲动:

既然状态参数的类型都统一了，那要简化子控件访问状态的逻辑，不就是要打包 `incomingState.wrappedValue` 这个逻辑嘛！让每个对 `incomingState` 的访问都以 `incomingState.wrappedValue` 的方式进行，不就可以了嘛？

没错！还记得 Property Wrapper 就是干这事的吧！

想必 Swift 开发团队也是这么想的，于是就有了 `@Binding`

### 可能的 Binding 结构体

有了 `@State` 的经验，猜想 `@Binding` 的实现就简单多了

不妨猜测简化后的 `@Binding` 如下:

```swift
@propertyWrapper
struct Binding<T> {
    let _binding: MyBinding<T>

    let wrappedValue: T {
        get {
            self._binding.wrappedValue
        }
        nonmutating set {
            self._binding.wrappedValue = newValue
        }
    }

    let projectedValue: Binding<T> {
        self
    }

    init(@escaping get: () -> T, @escaping set: (T) -> Void) {
        self.get = get
        self.set = set
    }
}
```

这看起来和 `MyBinding<T>` 的定义也太像了！不妨试试合二为一:

```swift
@propertyWrapper
struct Binding<T> {
    let get: () -> T
    let set: (T) -> Void

    let wrappedValue: Binding<T> {
        get {
            self.get()
        }
        nonmutating set {
            self.set(newValue)
        }
    }

    let projectedValue: Binding<T> {
        self
    }

    init(@escaping get: () -> T, @escaping set: (T) -> Void) {
        self.get = get
        self.set = set
    }
}
```

太神奇了，这样甚至不知不觉中统一了 `Binding<T>` 和 `MyBinding<T>` 的实现！接下来，只要让编译器生成 `synthesized initializer` 的时候适配 `Binding<T>` 的初始化方式 (使用 `get` 和 `set` 参数而不是 `Binding<T>` 对象) 似乎就完成了

### 展开后的 @Binding 代码

那么，根据 Property Wrapper 的展开逻辑，我们来推测一下 `@Binding` 展开后的代码:

```swift
struct MySubView: View {
    // Original declaration
    // @Binding var parentState: Int

    // Expanded declaration
    var _parentState: Binding<Int>

    var parentState: Int {
        get {
            _parentState.wrappedValue
        }
        set {
            _parentState.wrappedValue = newValue
        }
    }

    var $parentState: Binding<Int> {
        get {
            _parentState.projectedValue
        }
    }

    // Synthesized initializer
    init(parentState: Binding<Int>) {
        self._parentState = Binding {
            parentState.wrappedValue
        }, set: { newValue in
            parentState.wrappedValue = newValue
        }
    }
}
```

## 一些奇怪的问题记录

至此，基本完成了对 Swift 中 `@State` 和 `@Binding` 实现原理的深入探索了，但难免还会有一些未解的疑惑

感谢如今大模型的能力，使得部分奇怪的问题得以解答

以下列出部分与大模型问答的摘要

### 手动指定 initializer

#### 提问

如果 `@State` 和 `@Binding` 的生成涉及 `synthesized initializer` 的生成，那么如果我自己手动指定了 initializer，是否与前述中自动生成的 initializer 产生冲突?

#### 回答

会产生影响，如果手动指定了 initializer，需要手动添加对应的初始化逻辑，例如:

```swift
init(parentState: Binding<Int>, someOtherParams: MyType) {
    // Add the initialization of bindings
    self._parentState = Binding {
        parentState.wrappedValue
    }, set: { newValue in
        parentState.wrappedValue = newValue
    }

    // Other logic
}
```
