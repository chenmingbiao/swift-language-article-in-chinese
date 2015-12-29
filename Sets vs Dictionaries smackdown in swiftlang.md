title: "在Swift语言中集合与字典的角逐"
date: TODO
tags: [ ]
categories: [Swift 进阶]
permalink: Sets_vs_Dictionaries_smackdown_in_swiftlang

---
原文链接=http://ericasadun.com/2015/10/19/sets-vs-dictionaries-smackdown-in-swiftlang/
作者=Erica Sadun
原文日期=2015/10/19
译者=CMB
校对=
定稿=
发布时间=

<!--此处开始正文-->

传统的 `Cocoa` 在使用字典时有个不好的习惯。从用户信息到字体选项再到视频流(`AV`)设置，`NSDictionary` 一直担任 `Cocoa` 传递数据的角色。字典是灵活的，易用的，但它也是一个潜在的危机。

在这篇文章中，我将讨论另一种更快捷的方法。这不是一站式( `turn-key` )的解决方法，但我认为它是一个更好展示应用程序接口(下文中统一简称为 `APIs` )在 `Swift` 快速发展时代的工作思维倾向。

###基于字典的设置工作

下面的代码是随机从我自己项目中抽取出来的。(如果你熟悉我其他的作品或许你会有印象，这是我写的电影制作类)。这几行 `Objective-C` 代码是创建一个名为 `options` 的字典，它是用来构建一个视频流(`AV`)像素缓冲区：

```Objective-C
 NSDictionary *options = @{
    (id) kCVPixelBufferCGImageCompatibilityKey : 
        @YES,
    (id) kCVPixelBufferCGBitmapContextCompatibilityKey : 
        @YES,
 };
```

这个例子中键(`key`)默认为 `(id)` 类型， 并且使用 `Objective-C` 字面量的方式将布尔类型(`Booleans`)的值转换为 `NSNumber` 类型。这种操作在 `Swift` 中使用起来更加简便。而且编译器已经足够智能去把字典中值和类型关联起来(类似脚本语言，赋予值后就会自动声明为该值的类型)。

```swift
let myOptions: [NSString: NSObject] = [
    kCVPixelBufferCGImageCompatibilityKey: true,
    kCVPixelBufferCGBitmapContextCompatibilityKey: true
]
```

即使在 `Swift` 中将值传递给 `APIs` 也是一种很不理想的方式 。

###设置字典的特性

下面的列子中展示了设置字典的基础特性，这些特性都是值得仔细研究的。

 * 他们有一套固定合法的键(`key`)，大概有 `12` 个合法的像素缓冲区属性键在 `AVFoundation` 库里。这个集合里面的键(`key`)很少会被改变，而是用于绑定到一个完善的任务中。

 * 请求的值关联着每个已知类型键的实例。这些类型相比 `NSObject` 显得更加细致，例如描述“图像右边填充像素的大小（ `CFNumber` 类型）。”

 * 类型的安全关系着值的传递，尽管无法通过一个 `NSDictionary` 就可以执行值的传递。前面的例子因为考虑了兼容性，所以键的类型应该是 `Boolean` 类型，而不是 `Int` 或 `String` 或 `Array`，甚至 `NSNumber` 类型。

 * 有效的条目在字典中只会出现一次，键是散列的(哈希)，新条目将会覆盖旧条目。

在 `Swift` 中，上述列出的特性在集合和枚举中显得比字典更加典型。理由如下：

 * 枚举列出了所有给定类型的可能选项。大多数 `Cocoa APIs` 类似于这个例子有固定的，不变的键。

 * 枚举使你在个别情况下可以关联类型值。 `Cocoa APIs` 文档表明，他们希望通过每个键来传递类型。

 * 像字典一样，集合中有成员的限制以避免多个实例的产生。

基于这些原因，我觉得在 `Swift` 中设置集合时使用枚举会比一个 `[NSString: NSObject]` 的字典表达效果更好。

###键的转换

退一步考虑现在最先进的技术。`AVFoundation` 定义了一系列的键，如下所示。 （这并不是一个完整像素缓冲器的键，只显示了一部分）

```swift
const CFStringRef kCVPixelBufferPixelFormatTypeKey;
const CFStringRef kCVPixelBufferExtendedPixelsTopKey;
const CFStringRef kCVPixelBufferExtendedPixelsRightKey;
const CFStringRef kCVPixelBufferCGBitmapContextCompatibilityKey;
const CFStringRef kCVPixelBufferCGImageCompatibilityKey;
const CFStringRef
```

上面都是一些常量字符串，这些字符串都被用于作为字典的索引。调用者通过使用这些键来创建字典，通过键作为值传递任意对象。

在 `Swift` 中，你可以重新设计该基于密匙(`key-based`)的系统为一个简单的枚举，为每个键都关联指定的有效类型。下面的枚举就如上面一样。相关的类型来自现有的像素缓冲区属性文档。

```swift
enum CVPixelBufferOptions {
 case CGImageCompatibility(Bool)
 case CGBitmapContextCompatibility(Bool)
 case ExtendedPixelsRight(Int)
 case ExtendedPixelsBottom(Int)
 case PixelFormatTypes([PixelFormatType])
 // ... etc ...
}
```

重新设计 `options` 为一个可扩展的枚举，每个可能的情况下，严格规定值的类型。这种方法为你提供了一个主要的类型，这种类型相比起 `weak ` 字典类型要显得安全。

此外，在个别枚举案例也会更清晰，更简洁，使用作为数据交互也比名字很长很详细的 `Cocoa` 常量更好，例如 `kCVPixelBufferCGBitmapContextCompatibilityKey` 这个常量名字就显得非常啰嗦。`Cocoa` 称会承担起提醒其作为一个常数（k）去使用，提醒其关联的类（`CVPixelBuffer`）以及提醒其使用的角色（`key`）的责任，所有的一切都可以在此时避免发生。

###创建集合的设置

通过重新设计，你可以通过调用去创建一个集合，这个集合就如下面的例子。我想说的是下面的例子还没被编译的。

```swift
// This does not compile yet
let bufferOptions: Set<CVPixelBufferOptions> = 
    [.CGImageCompatibility(true), 
     .CGBitmapContextCompatibility(true)]
```

`Swift` 不能编译以上的代码，因为 `CVPixelBufferOptions` 的 `options` 还不是遵循 `Hashable` 协议。为了解决这个问题，你可以建立一个数组，但是需要注意的是这个数组要满足不能只有唯一成员属性的条件：

```swift
// This compiles
let bufferOptions: [CVPixelBufferOptions] =
    [.CGImageCompatibility(true),
     .CGBitmapContextCompatibility(true)]
```

数组使用起来是十分友好的，但它却少了独特 `options` 的功能，但是字典是有提供这个功能的，因为这个功能，所以正在推动重新设计数组。

###区分值

`Hashable` 协议使 `Swift` 可以区分不同的实例。集合和字典都使用了哈希来确保成员和键都是唯一的。如果没有哈希，他们不能提供这些保证。

当创建设置集合时，你希望创建集合，但是这个集合不会构建成一个类似于下面的示例，其中有冲突设置的多个成员同时存在：

```swift
[.CGImageCompatibility(true),
 .CGImageCompatibility(false)] // which one?!
```

这显然是两个不同的枚举实例由于有不同的相关值。在这个例子中，你需要一个集合丢弃遵循的第一个成员添加到集合，只留下了 `“true”` 情况下，所有相同的选项。 （字典遵循相反的规则。字典是替换现有成员，而不是丢弃。）

通过实现哈希，使你能够比较枚举的情况。

###实现哈希值

对于这个特定的用例，你需要创建一个哈希函数，该函数只考虑唯一的情况，而不是考虑关联值。目前在 `Swift` 中没有提供此功能的构造函数，所以你需要自己创建这个构造函数。

 `Swift` 的哈希要确保遵从 `Equatable` 协议，因此，你的实现必须解决两组的要求。对于 `Hashable` 协议，你必须返回一个哈希值。对于 `Equatable` 协议 ，必须实现 `==` 函数。

```swift
public var hashValue: Int { get } // hashable
public func ==(lhs: Self, rhs: Self) -> Bool // equatable
```

基本的枚举，例如 `MyEnum {case A, B, C}` 提供了原始值，这个原始值告诉你哪些项你正在使用。这些值都是从零开始，并都使用起来十分方便。不幸的是，枚举的关联值不提供原始值的支持，使这项工作变得更加困难。所以，你必须亲手建立哈希值。

下面是 `CVPixelBufferOptions` 的 `extension` ，它手动为每一种情况增加哈希值。

```swift
extension CVPixelBufferOptions: 
    Hashable, Equatable {
    public var hashValue: Int {
        switch self {
          case .CGImageCompatibility: return 1
          case .CGBitmapContextCompatibility: return 2
          case .ExtendedPixelsRight: return 3
          case .ExtendedPixelsBottom: return 4
          case .PixelFormatTypes: return 5
       }
    }
}

public func ==(lhs: CVPixelBufferOptions, 
    rhs: CVPixelBufferOptions) -> Bool {
    return lhs.hashValue == rhs.hashValue
}
```

从最直观的一面可以看出这些哈希值绝对没有任何意义而且也不会暴露给 `API` 使用者，所以如果你需要坚持添加额外的值，你可以这样做。这就是说，这种做法是丑陋，而且让人感觉起来非常不方便。

一旦你添加这些功能，一切都将开始工作。你可以创建集合的设置，来保证每一个集合只出现一次以及保证它们的值关联的是正确的类型。

###总结

在这篇文章中所描述支持笨重哈希的方法远胜于 `Cocoa` 的 `NSDictionary` 方法。类型安全，枚举和集合都提供了更好的解决方案，否则使用起来就像一个古老而过时的 `API` 。

我想 `Swift` 真正需要的是选项集合与值能更好的关联在一起。至少，在所有的枚举项中添加原始值的支持（不只是基本的那些缺乏相关的或内在价值）将是向前迈进一大步。

Thanks, Erik Little