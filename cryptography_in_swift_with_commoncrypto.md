#Cryptography in Swift with CommonCrypto

CommonCrypto密码类在Swift中的应用

![](http://img-storage.qiniudn.com/15-8-23/34909577.jpg)

Many developers don’t have to deal with cryptography in their Apps nowadays. Even if you are working with a REST API from a remote server, usually sticking to HTTPS solves most secure communication issues, and Apple “Protected mode” and the hardware/software encryption combo do the rest. However, there are many situations where you need to deal with encrypted communications or documents. Perhaps you are porting an existing solution to iOS and the communication involves ciphered documents/messages, perhaps you are working in an App where security is important, or perhaps you just like to add an extra security level to your data (which is always a good thing).

现在许多开发者不必在他们的Apps中进行加密处理。即使你在远程服务器上使用了REST API，通常情况下使用HTTPS可以解决大多数的安全通信问题，其余可以使用苹果提供的“保护模式”和硬件/软件加密组合方式来解决。然而在很多情况下，你还需要对通信或文件进行加密。也许你正在移植一个iOS现有的解决方案在通信涉及到加密文件/信息，也许你在制作一个要求保密性高的App，或者也许你只是想为数据添加一个额外的安全级别（这是一件好事）。

Whatever the case may be, CommonCrypto has been the reference framework for Cocoa (both in iOS and OS X) for these duties. However, CommonCrypto uses an old C-Style API that’s been showing signs of aging for years , and that now with Swift is even more unintuitive to use. Specially, the strongly typed nature of Swift makes dealing with the different data types of CCCrypt (the main encryption/decryption function of the framework for symmetric cryptography) less than optimal. Let’s have a look at the definition of CCCrypt:

不管是什么情况，CommonCrypto有义务一直为Cocoa提供参考框架（在iOS和OS X系统中），然而CommonCrypto一直使用着旧的C风格（C-Style）的API，这种API风格多年来显示出老化的迹象，现在Swift是更直观的使用。特别地，Swift中强类型属性处理来CCCrypt中不同类型的数据（对称式加密框架的主要加密/解密功能）是不太理想的方式。让我们看一下CCCrypt的定义：

```
CCCrypt(op: CCOperation, alg: CCAlgorithm, options: CCOptions, key: UnsafePointer<Void>, keyLength: Int, iv: UnsafePointer<Void>, dataIn: UnsafePointer<Void>, dataInLength: Int, dataOut: UnsafeMutablePointer<Void>, dataOutAvailable: Int, dataOutMoved: UnsafeMutablePointer<Int>)
```

The signature of the Objective-C function (more precisely, the C version of the function) was different from the point of view of types involved:

在Objective-C函数的签名（更准确来说是C版本的函数）是从不同角度涉及的类型：

```
CCCryptorStatus CCCrypt(
 CCOperation op,          // operation: kCCEncrypt or kCCDecrypt
 CCAlgorithm alg,         // algorithm: kCCAlgorithmAES128... 
 CCOptions options,       // operation: kCCOptionPKCS7Padding...
 const void *key,         // key
 size_t keyLength,        // key length
 const void *iv,          // initialization vector (optional)
 const void *dataIn,      // input data
 size_t dataInLength,     // input data length
 void *dataOut,           // output data buffer
 size_t dataOutAvailable, // output data length available
 size_t *dataOutMoved)    // real output data length generated
 ```

In the good old Objective-C days, one would simply use the predefined constants for operation, algorithm and options, like “kCCAlgorithm3DES” for instance, and pass the different arrays and sizes without worrying much about their exact types (i.e: ints to size_t variables, or”char *” buffers where “void *” ones where expected). It was not the best of practices, but it worked and only required (at most) some castings.

在过去Objective-C的日子，简单地使用预定义常数来操作，算法和选项，如“kccalgorithm3des”为例可以通过不同的数组和大小而不必担心它们的确切类型（即size_t整型变量或“void *”所期望的“char *”缓冲器）。这不是最好的做法，但它的工作（在大多数情况下）只需要一些造型。

But now with Swift we have taken the “C” of Objective-C out of the equation, and we need some more work to prepare our Swift and Cocoa variables for CommonCrypto:

但是现在在Swift中我们采取了"C"中的Objective-C所没有的方程式，还有我们需要对CommonCrypto做一些Swift和Cocoa变量的准备工作：

##Of operations, algorithms and options

操作，算法和选项

Symmetric cryptography is one of the simplest forms used for sending and receiving ciphered data in Apps. There is only one key that’s used both to cipher the clear text and decipher the encrypted one (as opposed to asymmetric cryptography, where usually a public-private pair of keys are used). There are many different algorithms for symmetric cryptography, and all of them can have different options. These three main concepts: operation used (encrypt/decrypt), algorithm (AES, DES, RC4…) and options are mapped to the CommonCrypto concepts CCOperation, CCAgorithm and CCOptions respectively.

在Apps中对称密码体制是一种用于发送和接收加密数据的最简单形式。这种形式只有一个密钥，它用于加密明文和解密它们（相对于非对称加密，它通常是一对公－私密钥的使用）。对称密码有许多不同的算法，所有的算法都可以有不同的选择。这三个主要概念分别是：操作使用（加密/解密），算法（DES，AES，RC4……）和选择映射到CommonCrypto的概念CCOperation，CCAgorithm和CCOptions。

CCOperation, CCAlgorithm and CCOption are mere wrappers for uint32_t (that’s it, an unsigned int stored using 32 bits), so we just can build them and pass the proper CommonCrypto constant we are after:

CCOperation，CCAgorithm和CCOptions仅仅是用来包装uint32_t（一个占32位存储的unsigned int），所以我们可以通过适当的CommonCrypto常量来构造它们：

```
let operation = CCOperation(kCCEncrypt)
let algorithm = CCAlgorithm(kCCAlgorithmAES)
let options = CCOptions(kCCOptionPKCS7Padding | kCCOptionECBMode)
```

##Unsafe pointers

unsafe指针

Unsafe pointers are the abstraction used in Swift to manage C-pointers. As Swift tries to abstract itself from any kind of pointers or C-style memory management, they are not specially promoted or showcased. You should not need to use them unless you are accessing old-style APIs (like CommonCrypto), but in case you find yourself in that situation, here’s a quick overview of what you need to know to deal with them:

在Swift中Unsafe指针被抽象地使用在管理C语言指针(C-Pointers)。Swift试图从任何类型的指针或C语言风格的内存管理器抽象自己。它们不是特别的提升或展示。你不需要使用它们，除非你访问旧风格(old-style)的APIs(如CommonCrypto)，但如果你发现自己在这种情况下,下面将简要概述如果处理它们：

There are two types of pointers defined in swift: UnsafePointers and UnsafeMutablePointers. The first ones are used for constant buffers, i.e: pointers to memory areas that are constant and not subject to change. The second ones are for the mutable memory areas. So strictly from a C perspective, you would translate an UnsafePointer to a “const type *” buffer and an UnsafeMutablePointer to a “type *” buffer (excuse the use of the word “buffer” in this context, it’s just an old habit ;). The type of the pointer is declared in <> signs immediately after the pointer type declaration, so if you want to declare a pointer to a “void *”, you would do it with UnsafeMutablePointer<Void>. If you were to declare a pointer to an old “const unsigned char *” buffer, you would use UnsafePointer<UInt8>. While Apple does offer a translation from pure C-types to Swift types here, it’s important to note that those types (like CChar, CInt, CUnsignedLongLong…) are not used in the definition of UnsafePointers, using the native swift types instead. This leave us with some ambiguity as to where one type should be used or not. Fortunately, a little delving inside Swift’s types definitions give us some handful insights:

在Swift中定义两种类型的指针：分别为UnsafePointers和UnsafeMutablePointers类型。第一个被用于常量寄存器，如：内存空间上的指针是恒定的不受改变的。第二个是用于可变的内存空间。所以严格从C语言的角度来看，你要把UnsafePointer类型转化成"const type *"缓冲类型，把UnsafeMutablePointer转化成"type *"缓冲类型（这里的"缓冲"一词只是过去习惯的叫法）。在指针类型声明之后马上加上<>符号，所以如果你想去声明一个"void *"类型的指针，你需要这样做：UnsafeMutablePointer<Void>。如果你一个老的"const unsigned char *"缓冲类型的指针，你需要使用：UnsafePointer<UInt8>。当苹果公司提供纯粹的C类型(C-types)转化成Swift类型,这时记录这些类型(像 CChar，CInt， CUnsignedLongLong…)是非常重要的，因为这些类型将不能用于定义UnsafePointers类型，需要使用天然的Swift类型来替代它们。这会留给我们一些歧义，到底这些类型是否能被使用。幸运地，只要我们稍微深入理解Swift的类型定义就会了解到一些有用的见解：

```
typealias CShort = Int16
typealias CSignedChar = Int8
typealias CUnsignedChar = UInt8
typealias CUnsignedInt = UInt32
typealias CUnsignedLong = UInt
typealias CUnsignedLongLong = UInt64
typealias CUnsignedShort = UInt16
```

Thankfully we don’t need to take memory management of UnsafePointers and UnsafeMutablePointers into account when dealing with them (as long as you get the memory from a valid Cocoa object like NSData). This is automatically managed (and bridged) by Swift. If you have your raw data to encrypt/decrypt and your keys in NSData, it is easy to get a UnsafePointer or UnsafeMutablePointer by calling data.bytes or data.mutableBytes respectively.

值得庆幸的是我们不需要去考虑UnsafePointers和 UnsafeMutablePointers类型的内存管理（只要你得到一个像NSData一样的有效Cocoa对象的内存空间时）。这是Swift的自动管理（或桥接）。如果你有一个加密/解密和NSData密钥的原始数据，这是非常容易从调用data.bytes或data.mutableBytes中得到一个UnsafePointer或UnsafeMutablePointer类型的对象。

Another way of getting the UnsafePointer for a variable, useful when you are dealing with output variables (that need the memory address) is by means of the & sign (like you did in C) for getting from a Int to a Unsafe(Mutable)Pointer<Int>. We can use this technique in CCCrypt by passing the address of an “Int” variable to the last parameter: “dataOutMoved”. Note that a let-defined variable will produce an UnsafePointer<Type> whereas a var-defined one will produce an UnsafeMutablePointer<Type>.

另一种得到UnsafePointer变量的方式,当你在处理输出变量时(需要内存的地址)就是通过这个&符号(就像在C语言里面一样)用于从Int去得到Unsafe(Mutable)Pointer<Int>，我们可以在CCCrypt中使用这技术，通过把"Int"变量地址传给最后一个参数 ："dataOutMoved" 。要注意，一个用let定义的变量就产生一个UnsafePointer<Type>类型而var变量将产生一个UnsafeMutablePointer<Type>类型。

So now we have everything we need to call CCCrypt with our everyday Swift objects.

所以现在，我们有很多需要去调用的含有基本对象的CCCrypt。

##Bridging

桥接

CommonCrypto has not been exposed to pure Swift (yet?), so in order to use it, we need to build a bridging header and make an import of CommonCrypto the old Objective-C way:

CommonCrypto尚未接触到纯粹的Swift，所以为了使用它，我们需要建立一个桥接的头文件使得能导入旧Objective-c方式的CommonCrypto。


```
#import <CommonCrypto/CommonCrypto.h>
```

##SymmetricCryptor

SymmetricCryptor类

Recently I needed to work with symmetric cryptography for the project I’m currently working on.  In order to make it easier for me to encrypt and decrypt data, I built a SymmetricCryptor class (please excuse the terrible name) to help me from having to “translate” all my data to the proper CommonCrypto types. You can use it to encrypt or decrypt with a single, convenient call:

最近我需要做对称加密的项目，为了更容易的加密和解密数据，我建了一个symmetriccryptor类（不要在意这个可怕的名字）帮我把数据转化到适当的commoncrypto类型中。你可以使用这种单一，方便的调用去加密或解密数据。

```
let sc = SymmetricCryptor(algorithm: .AES128, options: CCOptions(kCCOptionPKCS7Padding))
cypher.setRandomIV()
do { let cypherText = try sc.crypt(string: clearText, key: key) } catch { print("Error while encrypting: \(error)") }
```

While CommonCrypto provides several algorithms and options, I wanted to address the common encryption configurations usually found when dealing with ciphering/deciphering. For example, when using RC4, you can use 40 or 128 bits keys (known as RC4_40 and RC4_128 respectively). Same goes for the different variants of AES (128b, 256b…). That’s why I defined an enum called SymmetricCryptorAlgorithm, that defines configurations (like AES 256) rather than just mere algorithms, as defined in the cipher specs by Apple, and so it contains information about the length of the key or the block size, for instance.

当我想要去解决常见的加密配置时，发现CommonCrypto提供几种算法和选项用于加密/解密处理。例如，在使用RC4时，可以使用40位或128位密钥（分别称为RC4_40和RC4_128）。也用于不同的AES变量（128B，256B...）。我定义了名为SymmetricCryptorAlgorithm一个枚举的原因，它定义了配置（如AES256），而不仅仅是单纯的算法，如由苹果公司提供的密码规范定义，它包含了密钥的长度或块大小，例如信息。

You can find the SymmetricCryptor class alongside a demo project showcasing how easy it’s to use the class for symmetric ciphering/deciphering here, in my github repository.

在我的GitHub的资源库中SymmetricCryptor类旁边，你可以找到一个对称加密/解密类示范项目，它展示了对称加密/解密类的使用是十分简单的。

![](http://img-storage.qiniudn.com/15-8-23/36052539.jpg)

And that’s all for now. I will write a continuation of this post dealing with asymmetric cryptography and the public-private key pair scheme. Let me know your thoughts in the comments below.

现在，我会持续写涉及到非对称加密技术和公私密钥对的方案。请留下你的评论，让我从下面的评论中知道你的看法。