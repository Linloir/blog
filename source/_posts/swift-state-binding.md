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
    var myPropertyUnderlyValue: MyValueType

    init(myPropertyUnderlyValue: MyInitialValueType) {
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
        set {
            self = newValue
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

### projectedValue 和 wrappedValue

## State 的魔法

### 可能的 State 结构体

### 展开后的 @State 代码

### 为什么使用 Binding 作为 projectedValue

## Binding 的魔法

### 可能的 Binding 结构体

### 展开后的 @Binding 代码
