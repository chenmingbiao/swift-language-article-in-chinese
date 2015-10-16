title: 捕获上下文
date: 2015-10-14
tags: [Swift]
categories: [ericasadun]
permalink: capturing_context

---
> 原文链接: [Capturing context](http://ericasadun.com/2015/08/27/capturing-context-swiftlang/)
> 原文日期: 2015/08/27

> 译者：[chenmingbiao](https://github.com/chenmingbiao)
> 校对：[xxx](xxx)
> 审核：[xxx](xxx)

#捕获上下文

---


假设你正在使用一个类型，当有错误时发生时你想要抛出冲突在文中出现的位置，你可以做到这一点，主要是使用一些内置的编译器关键字：`__FUNCTION__` ， `__LINE__` 和 `__FILE__` ，这些关键词提供文字插值有关调用函数的详细信息

Say you’re working with a type and you want to throw an error that reflects the context of where it originates. You can do that, mostly, using a few built-in compiler keywords. __FUNCTION__, __LINE__, and __FILE__ provide literal interpolation about the details of a calling function:

```swift
public struct Error: ErrorType {
    let source: String; let reason: String
    public init(_ reason: String, source: String = __FUNCTION__,
        file: String = __FILE__, line: Int = __LINE__) {
            self.reason = reason; self.source = "\(source):\(file):\(line)"
    }
}
```

一行典型的错误输出提示就像这个例子一样:

> Error(source: "myFunction():<EXPR>:14", reason: "An important reason")

虽然这种结构使你能够捕获错误的函数名，文件名和行序，但你无法捕捉没有类型参数的原始父类型。抓取该类型，扩展 `Error` 初始化为包括“源类型”，和从构造传递 `self.dynamicType` 。

While this struct enables you to capture the error’s function, file, and line, you can’t capture the originating parent type without a type parameter. To fetch that type, extend the Error initializer to include a “source type”, and pass self.dynamicType from the constructor.

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

我找了额外的类型参数深深讨厌。这种方法的整个点是简化误差生成。

I find the extra type parameter deeply annoying. The entire point of this approach is to simplify error generation.


```swift
public struct Parent {
    func myFunction() throws {
        throw Error("An important reason", type: self.dynamicType)}
}

do {try Parent().myFunction()} catch{print(error)}
// Error(source: "myFunction():<EXPR>:14:Parent", reason: "An important reason")
```

使用协议自动拿起型环境。在以下 `Contextualizable` 分机默认实现指 `self.dynamicType` 的方式，方法签名不能。

Use a protocol to automatically pick up on type context. The default implementation in the following Contextualizable extension refers to self.dynamicType in a way that a method signature cannot.

```swift
protocol Contextualizable {}
extension Contextualizable {
    func currentContext(file : String = __FILE__, function : String = __FUNCTION__, line : Int = __LINE__) -> String {
        return "\(file):\(function):\(line):\(self.dynamicType)"
    }
}
```

结合的方法，可以简化整个事情。共享错误类型简化为简单的让和环境责任的错误初始化移动到符合要求的类型，自动继承 `currentContext` 方法。

Combining approaches enables you to simplify the whole shebang. The shared Error type reduces to simple lets and context responsibility moves from the Error initializer to a conforming type, automatically inheriting the currentContext method.

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

更新后的结果现在包括在字符串输出原始类型。

The updated results now includes the originating type in the string output.

正如读者 `Kametrixom` 所指出的，你还可以扩展 `Contextualizable` 代表您构建一个错误。 （他还写了选择性增加方面一个非常好的错误类型。）

As reader Kametrixom points out,  you can also extend Contextualizable to construct an error on your behalf. (He’s also written a really nice error type that optionally adds context.)

>	@ericasadun How about this? pic.twitter.com/1ijQD5uN3j

>	— Kametrixom (@Kametrixom) August 27, 2015

这里有一个链接到一个要点，我已经抛出了一些这方面的在一起。

Here’s a link to a gist where I’ve thrown up some of this all together.



