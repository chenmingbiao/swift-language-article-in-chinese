title: "在Swift语言中集合与字典的角逐"
date: TODO
tags: [Erica Sadun]
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

Traditional Cocoa has a bad dictionary habit. From user info to font options to AV settings, NSDictionary has long acted as the Cocoa workhorse for passing data. Dictionaries are flexible, easy to use, and a minefield of potential disasters.

传统的 `Cocoa` 有一个坏的字典习惯。从用户信息到字体选项AV设置， `NSDictionary` 一直担任 `Cocoa` 传递数据的角色。字典是灵活的，易用的，但它也是一个深水炸弹。

In this post, I’m going to discuss an alternative approach, one that’s far more Swift-y. It’s not yet a completely turn-key solution but I think its one that showcases a much better mindset for how APIs should be working in a post-Swift world.

在这篇文章中，我将讨论另一种更快捷的方法。这还不是一个完全交钥匙解决方案，但我认为它的一个展示如何API应该在后雨燕的世界是工作一个更好的心态。

###Working with Dictionary-based Settings

###与基于字典的环境中工作

Here is some code I pulled out of a random project of mine. (If you’re familiar with my other writings, it’s from my Movie Maker class.) This Objective-C code builds an option dictionary, which is used to construct a AV pixel buffer:

下面是一些代码，我拿出我的一个随机的项目。 （如果你熟悉我的其他作品，这是从我的电影制作类）。这Objective-C代码生成选项的字典，它是用来构建一个AV像素缓冲区：

```swift
 NSDictionary *options = @{
    (id) kCVPixelBufferCGImageCompatibilityKey : 
        @YES,
    (id) kCVPixelBufferCGBitmapContextCompatibilityKey : 
        @YES,
 };
```

This example casts the option keys to (id) and uses Objective-C literal notation to transform Booleans to NSNumber instances. The Swift version of this is much simpler. The compiler is smart enough to associate  values with the declared dictionary type.

这个例子铸选项键（ID），并使用Objective-C的文字符号转换为布尔值的NSNumber实例。这样做的夫特版本要简单得多。编译器是足够聪明的价值观与声明的字典类型相关联。

```swift
let myOptions: [NSString: NSObject] = [
    kCVPixelBufferCGImageCompatibilityKey: true,
    kCVPixelBufferCGBitmapContextCompatibilityKey: true
]
```

Even in Swift, this is far from an ideal way to pass values to APIs.

即使在Swift，这是很不理想的方式来传递值的API。

###Characteristics of Setting Dictionaries

###设置词典的特点

Settings dictionaries like the examples you just saw exhibit common characteristics, which are worth examining carefully.

 * They have a fixed set of legal keys. There are about a dozen legal pixel buffer attribute keys in AVFoundation. This collection rarely  changes and those keys are tied to a well-established task.
 
 * The requested values associated with each key instance are of known types. These types are more nuanced than just NSObject, for example “The number of pixels padding the right of the image (type CFNumber).”
 
 * Type safety does matter for the passed values, even though this cannot be enforced through an NSDictionary. Both compatibility keys in the preceding example should be Boolean, not Int or String or Array or even NSNumber.
 
 * Valid entries appears just once in the dictionary, as keys are hashed and new entries overwrite older ones.

In Swift, the characteristics in the above bullets list are far more typical of sets and enumerations than dictionaries. Here are some reasons why.

 * An enumeration expresses a full range of possible options for a given type. Most Cocoa APIs similar to this example have fixed, unchanging keys.
 
 * Enumerations enable you to associate typed values with individual cases. Cocoa APIs document the types they expect to be passed for each key.
 
 * Like dictionaries, sets restrict membership to avoid multiple instances.

For these reasons, I think these settings collections are better expressed in Swift as sets of enumerations instead of an [NSString: NSObject] dictionary.

设置字典一样的例子，你刚才看到具有共同的特点，这是值得仔细研究。

 *他们有固定的一套法律钥匙。有关于AVFoundation十几法律像素缓冲区属性键。这个集合很少改变那些键被绑定到一个完善的任务。
 
 *与每个键实例关联的请求的值是已知的类型。这些类型不仅仅是NSObject的更细致，例如“像素填充图像（类型CFNumber）的权数。”
 
 *类型的安全关系了传递的值，尽管这无法通过一个NSDictionary执行。前面的例子在兼容的键应该是布尔值，而不是int或字符串或数组，甚至NSNumber的。
 
 *有效条目在字典中出现一次就好，因为密钥散列和新条目将覆盖旧的。

在夫特，在上述列表中的子弹的特性是更加典型的套和枚举比字典。这里有一些原因。

 *枚举表达了全方位的给定类型的可能选项。大多数Cocoa API的类似于这样的例子有固定的，不变的钥匙。
 
 *枚举使您可以键入值与个别案例关联。可可的API文档，他们希望传递的每个键的类型。
 
 *如字典，组成员资格限制，以避免多个实例。

基于这些原因，我觉得这些设置的集合在斯威夫特更好的表现为集枚举，而不是[的NSString：NSObject的]的字典。

###Transitioning from Keys

###从按键过渡

Stepping back a second, consider the current state of the art. AVFoundation defines a series of keys like the following. (This is not a complete set of the pixel buffer keys, by the way)

退一步第二，考虑在本领域的当前状态。 AVFoundation定义了一系列类似以下键。 （这并不是一个完整的像素缓冲器键的方式）

```swift
const CFStringRef kCVPixelBufferPixelFormatTypeKey;
const CFStringRef kCVPixelBufferExtendedPixelsTopKey;
const CFStringRef kCVPixelBufferExtendedPixelsRightKey;
const CFStringRef kCVPixelBufferCGBitmapContextCompatibilityKey;
const CFStringRef kCVPixelBufferCGImageCompatibilityKey;
const CFStringRef
```

Each of these items is a  constant string and each string is used to index dictionaries. Callers build dictionaries using these keys, passing arbitrary objects as values.

所有这些项目是一个常量字符串，每个字符串被用于索引字典。来电者使用这些按键，通过任意对象作为值构建词典。

In Swift, you can re-architect this key-based system into a simple enumeration, specifying the valid type associated with each key case. Here’s what this looks like for the five cases listed above. The  associated types are pulled from the existing pixel buffer attribute key docs.

在Swift，你可以重新设计该键为基础的系统进入了一个简单枚举，指定与每个键的情况下相关的有效的类型。以下是这看起来像上面列出的五个案件。相关的类型从现有的像素缓冲区拉属性关键文档。

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

Re-designing options this way produces an extensible enumeration with strict value typing for each possible case. This approach offers you a major type safety victory compared to weak dictionary typing.

重新设计的选择这种方式会产生一种可扩展的枚举严格的价值类型为每个可能的情况下。这种方法为您提供比较弱词典打字的主要类型安全的胜利。

In addition, the individual enumeration cases are also clearer, more succinct, and communicate use better than exhaustively long Cocoa constants. Compare these cases with, for example,  kCVPixelBufferCGBitmapContextCompatibilityKey. The Cocoa names take responsibility for mentioning their use as a constant (k), their related class (CVPixelBuffer), and their usage role (key), all of which can be dropped here.

此外，个别枚举箱子也更清晰，更简洁，与通信使用比详尽长可可常量更好。比较这些情况下与例如，kCVPixelBufferCGBitmapContextCompatibilityKey。可可名称承担责任，提到自己作为一个常数（k）的使用，其相关的类（CVPixelBuffer）中，其使用的角色（密钥），所有这一切都可以在这里删除。

###Building Settings Collections

###大厦设置的集合

With this redesign, your call constructing a set should look like the following example. I say should because this example will not yet compile.

有了这个重新设计，您的通话构建了一套看起来应该像下面的例子。我说应该，因为这个例子没有编译。

```swift
// This does not compile yet
let bufferOptions: Set<CVPixelBufferOptions> = 
    [.CGImageCompatibility(true), 
     .CGBitmapContextCompatibility(true)]
```

Swift cannot compile this because the CVPixelBufferOptions options are not yet Hashable. At this point, all you can build is an array, which does not guarantee unique membership:

斯威夫特不能编译这一点，因为CVPixelBufferOptions选项尚未哈希。在这一点上，你可以建立是一个数组，这并不能保证唯一成员：

```swift
// This compiles
let bufferOptions: [CVPixelBufferOptions] =
    [.CGImageCompatibility(true),
     .CGBitmapContextCompatibility(true)]
```

Arrays are all well and lovely but they lack the unique options feature that dictionaries offer and that is driving this redesign.

数组都好可爱，但他们缺乏独特的选项功能，词典提供，并正在推动这一重新设计。

###Differentiating Values

###鉴别价值

The Hashable protocol enables Swift to differentiate instances which are basically the same thing from instances which are different things. Both sets and dictionaries use hashing to ensure that members and keys are unique. Without hashing, they cannot provide these assurances.

该哈希协议使雨燕区分情况下它们基本上是从实例这是不同的东西同样的事情。这两套和字典使用散列，以确保成员和钥匙都是独一无二的。如果没有哈希，他们不能提供这些保证。

When establishing settings collections, you want to build sets that won’t construct an example like the following, where multiple members with conflicting settings are present at the same time:

在建立设置的集合，要打造集不会构成类似以下内容，其中多个成员与冲突的设置都存在在同一时间的一个例子：

```swift
[.CGImageCompatibility(true),
 .CGImageCompatibility(false)] // which one?!
```

These are clearly two distinct enumeration instances due to the different associated values. In this example, you’ll want a set to discard all identical options that follow the first member added to the set, leaving just the “true” case. (Dictionaries follow the opposite rule. Subsequent additions replace existing members instead of being discarded.)

这些显然是由于不同的相关值两个不同的枚举实例。在这个例子中，你需要一组丢弃遵循的第一个成员添加到组，只留下了“真实”情况下，所有相同的选项。 （字典遵循相反的规则。随后补充更换，而不是被丢弃现有的成员。）

You achieve this by implementing hashing, which enables you to compare enumeration cases.

通过实施散列，使您能够比较枚举的情况下，你做到这一点。

###Implementing Hash Values

###实现哈希值

For this specific use-case, you need to create a hashing function that considers  case and only case, and not associated values. There is currently no native construct in Swift that offers this functionality, so you need to build this on your own.

对于这个特定的用例，你需要创建一个散列函数，考虑的情况下，只有情况下，并没有相关的值。目前，提供这种功能的斯威夫特没有原生结构，所以你需要自己建这个。

Swift’s Hashable protocol conforms to Equatable, so your implementation must address both sets of requirements. For Hashable, you must return a hash value. For Equatable, you must implement ==.

斯威夫特的哈希协议符合Equatable，让您的实现必须解决这两组的要求。对于哈希，你必须返回一个哈希值。对于Equatable，必须实现==。

```swift
public var hashValue: Int { get } // hashable
public func ==(lhs: Self, rhs: Self) -> Bool // equatable
```

Basic enumerations, e.g. enum MyEnum {case A, B, C} provide raw values that tell you which item you’re working with. These are numbered from zero, and are super handy to use. Unfortunately, enumerations with associated values don’t provide raw value support, making this exercise a lot harder than it might otherwise be. So you have to build hash values by hand, which kind of sucks.

基本枚举，例如枚举MyEnum{情况下，A，B，C}规定，告诉你哪些项目你正在使用原始值。这些都是从零开始编号，并都超方便的使用。不幸的是，枚举与关联值不提供原始值的支持，使这项工作变得更加困难比它否则可能。所以，你必须建立由专人哈希值，哪一种吮吸。

Here’s an extension of CVPixelBufferOptions, which manually adds hash values for each case. Ugh.

下面是CVPixelBufferOptions的延伸，它增加了手动哈希值的每一种情况。 啊。

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

On the (slightly) bright side, these hash values have absolutely no meaning and are never exposed to API consumers, so if you need to stick in extra values, you can do so. That said, this approach is ugly and hacky and feels very un-Swift.

在（略）光明的一面，这些哈希值绝对没有任何意义，不会暴露给API消费者，所以如果你需要额外的价值坚持，你可以这样做。这就是说，这种做法是丑陋，哈克，感觉非常不斯威夫特。

Once you add these features, however, everything starts to work. You can build sets of settings, with the assurance that each setting appears just once and their associated values are well typed.

一旦你添加这些功能，但是，一切都开始工作。你可以打造集的设置，与每个设置出现了一次，并且它们的相关值以及输入的保证。

###Final Thoughts

###最后的思考

The approach described in this post, clunky hashing support and all, is far better than the Cocoa NSDictionary approach. Type safety, enumerations, and set, all provide a better solution for what otherwise feels like an archaic API dinosaur.

在这篇文章中，笨重的散列支持和全体描述的方法，是远远超过了可可的NSDictionary方法要好。安全型，枚举和设置，都提供了什么，否则感觉就像一个古老的API恐龙一个更好的解决方案。

What Swift really needs here, though, is something closer to option sets with associated values. At a minimum, adding raw value support to all enumeration cases (not just basic ones that lack associated or intrinsic values) would be a major step forward.

斯威夫特什么真正需要的这里，不过，是更接近于选项集与关联值。至少，将原始值的支持，以列举的所有案件（不只是基本的那些缺乏相关的或内在价值）将是向前迈进一大步。

Thanks, Erik Little