title: 捕获语境
date: 2015-11-09
tags: [Swift]
categories: [ericasadun]
permalink: capturing_context

---
> 原文链接: [Capturing context](http://ericasadun.com/2015/08/27/capturing-context-swiftlang/)
> 原文日期: 2015/08/27

> 译者：[CMB](https://github.com/chenmingbiao)
> 校对：[xxx](xxx)
> 审核：[xxx](xxx)

#捕获语境

---

假设你正在使用一个类型，当有错误时发生时你想要抛出冲突在文中出现的位置，主要是通过使用一些内置的编译器关键字：`__FUNCTION__` ， `__LINE__` 和 `__FILE__` ，这些关键词提供了有关函数调用详细的文本插值：

```swift
public struct Error: ErrorType {
    let source: String; let reason: String
    public init(_ reason: String, source: String = __FUNCTION__,
        file: String = __FILE__, line: Int = __LINE__) {
            self.reason = reason; self.source = "\(source):\(file):\(line)"
    }
}
```

一行典型的 `Error` 输出提示就跟这个例子一样:

> Error(source: "myFunction():<EXPR>:14", reason: "An important reason")

虽然这种结构使你能够捕获错误的函数，文件和行序，但你无法捕捉没有类型参数的原始父类型。为了抓取该类型的，需要在 `Error` 结构体构造器中带上“源类型”的参数，还有需要从构造器中传递 `self.dynamicType` 参数。

```swift
public struct Error: ErrorType {
    let source: String; let reason: String
    public init(_ reason: String, type: Any = "", 
        source: String = __FUNCTION__,
        file: String = __FILE__, 
        line: Int = __LINE__) {
            self.reason = reason; self.source = "\(source):\(file):\(line):\(type)"
    }
}
```

我对这种额外添加类型参数的方式感到深深讨厌。这种方法只是简化了错误的产生。

```swift
public struct Parent {
    func myFunction() throws {
        throw Error("An important reason", type: self.dynamicType)}
}

do {try Parent().myFunction()} catch{print(error)}
// Error(source: "myFunction():<EXPR>:14:Parent", reason: "An important reason")
```

使用一个协议来自动选择语境类型。将以一种默认引用 `self.dynamicType` 的 `Contextualizable` 扩展方式来实现，这种方式用在方法签名是不可行的。

```swift
protocol Contextualizable {}
extension Contextualizable {
    func currentContext(file : String = __FILE__, function : String = __FUNCTION__, line : Int = __LINE__) -> String {
        return "\(file):\(function):\(line):\(self.dynamicType)"
    }
}
```

结合两种方法可以简化整个工作。共享 `Error` 类型可以减少一些常亮的定义，并且合理地把 `Error` 构造器移至一个符合的类型当中，这种类型自动继承了 `currentContext` 方法。

```swift
public struct Error: ErrorType {
    let source: String; let reason: String
    public init(_ source: String = __FILE__, _ reason: String) {
        self.reason = reason; self.source = source
    }
}
public struct Parent: Contextualizable {
    func myFunction() throws {
        throw Error(currentContext(), "An important reason")}
```

修改了输出结果的句子后，原始类型也包含在字符中进行输出。

正如读者 `Kametrixom` 所指出的，你还可以继承 `Contextualizable` 类去创建一个属于你自己的错误类型。（他还写了一个非常好的错误类型，这种错误类型可以随意添加到程序中。）`Kametrixom` 所写的错误类型如下图所示：

![](http://img-storage.qiniudn.com/15-11-9/61176604.jpg)
