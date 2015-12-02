title: "捕获上下文信息"
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

In this post, I’m going to discuss an alternative approach, one that’s far more Swift-y. It’s not yet a completely turn-key solution but I think its one that showcases a much better mindset for how APIs should be working in a post-Swift world.

###Working with Dictionary-based Settings

Here is some code I pulled out of a random project of mine. (If you’re familiar with my other writings, it’s from my Movie Maker class.) This Objective-C code builds an option dictionary, which is used to construct a AV pixel buffer:

```swift
 NSDictionary *options = @{
    (id) kCVPixelBufferCGImageCompatibilityKey : 
        @YES,
    (id) kCVPixelBufferCGBitmapContextCompatibilityKey : 
        @YES,
 };
```

This example casts the option keys to (id) and uses Objective-C literal notation to transform Booleans to NSNumber instances. The Swift version of this is much simpler. The compiler is smart enough to associate  values with the declared dictionary type.

```swift
let myOptions: [NSString: NSObject] = [
    kCVPixelBufferCGImageCompatibilityKey: true,
    kCVPixelBufferCGBitmapContextCompatibilityKey: true
]
```

Even in Swift, this is far from an ideal way to pass values to APIs.

###Characteristics of Setting Dictionaries

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

###Transitioning from Keys

Stepping back a second, consider the current state of the art. AVFoundation defines a series of keys like the following. (This is not a complete set of the pixel buffer keys, by the way)

```swift
const CFStringRef kCVPixelBufferPixelFormatTypeKey;
const CFStringRef kCVPixelBufferExtendedPixelsTopKey;
const CFStringRef kCVPixelBufferExtendedPixelsRightKey;
const CFStringRef kCVPixelBufferCGBitmapContextCompatibilityKey;
const CFStringRef kCVPixelBufferCGImageCompatibilityKey;
const CFStringRef
```

Each of these items is a  constant string and each string is used to index dictionaries. Callers build dictionaries using these keys, passing arbitrary objects as values.

In Swift, you can re-architect this key-based system into a simple enumeration, specifying the valid type associated with each key case. Here’s what this looks like for the five cases listed above. The  associated types are pulled from the existing pixel buffer attribute key docs.

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

In addition, the individual enumeration cases are also clearer, more succinct, and communicate use better than exhaustively long Cocoa constants. Compare these cases with, for example,  kCVPixelBufferCGBitmapContextCompatibilityKey. The Cocoa names take responsibility for mentioning their use as a constant (k), their related class (CVPixelBuffer), and their usage role (key), all of which can be dropped here.

###Building Settings Collections

With this redesign, your call constructing a set should look like the following example. I say should because this example will not yet compile.

```swift
// This does not compile yet
let bufferOptions: Set<CVPixelBufferOptions> = 
    [.CGImageCompatibility(true), 
     .CGBitmapContextCompatibility(true)]
```

Swift cannot compile this because the CVPixelBufferOptions options are not yet Hashable. At this point, all you can build is an array, which does not guarantee unique membership:

```swift
// This compiles
let bufferOptions: [CVPixelBufferOptions] =
    [.CGImageCompatibility(true),
     .CGBitmapContextCompatibility(true)]
```

Arrays are all well and lovely but they lack the unique options feature that dictionaries offer and that is driving this redesign.

###Differentiating Values

The Hashable protocol enables Swift to differentiate instances which are basically the same thing from instances which are different things. Both sets and dictionaries use hashing to ensure that members and keys are unique. Without hashing, they cannot provide these assurances.

When establishing settings collections, you want to build sets that won’t construct an example like the following, where multiple members with conflicting settings are present at the same time:

```swift
[.CGImageCompatibility(true),
 .CGImageCompatibility(false)] // which one?!
```

These are clearly two distinct enumeration instances due to the different associated values. In this example, you’ll want a set to discard all identical options that follow the first member added to the set, leaving just the “true” case. (Dictionaries follow the opposite rule. Subsequent additions replace existing members instead of being discarded.)

You achieve this by implementing hashing, which enables you to compare enumeration cases.

###Implementing Hash Values

For this specific use-case, you need to create a hashing function that considers  case and only case, and not associated values. There is currently no native construct in Swift that offers this functionality, so you need to build this on your own.

Swift’s Hashable protocol conforms to Equatable, so your implementation must address both sets of requirements. For Hashable, you must return a hash value. For Equatable, you must implement ==.

```swift
public var hashValue: Int { get } // hashable
public func ==(lhs: Self, rhs: Self) -> Bool // equatable
```

Basic enumerations, e.g. enum MyEnum {case A, B, C} provide raw values that tell you which item you’re working with. These are numbered from zero, and are super handy to use. Unfortunately, enumerations with associated values don’t provide raw value support, making this exercise a lot harder than it might otherwise be. So you have to build hash values by hand, which kind of sucks.

Here’s an extension of CVPixelBufferOptions, which manually adds hash values for each case. Ugh.

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

Once you add these features, however, everything starts to work. You can build sets of settings, with the assurance that each setting appears just once and their associated values are well typed.

###Final Thoughts

The approach described in this post, clunky hashing support and all, is far better than the Cocoa NSDictionary approach. Type safety, enumerations, and set, all provide a better solution for what otherwise feels like an archaic API dinosaur.

What Swift really needs here, though, is something closer to option sets with associated values. At a minimum, adding raw value support to all enumeration cases (not just basic ones that lack associated or intrinsic values) would be a major step forward.

Thanks, Erik Little