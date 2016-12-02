title: '[转]NSObject class和NSObject protocol的关系（抽象基类与协议）'
date: 2016-11-25 18:02:38
tags: [iOS, NSObject, Protocol]
---

主要讲解NSObject Class和NSObject Protocol的联系，以及NSObject和NSProxy的联系。

<!-- more -->

## 1、接口的实现

对于接口这一概念的支持，不同语言的实现形式不同。Java中，由于不支持多重继承，因此提供了一个Interface关键词。而在C++中，通常是通过定义抽象基类的方式来实现接口定义的。

Objective-C既不支持多重继承，也没有使用Interface关键词作为接口的实现（Interface作为类的声明来使用），而是通过抽象基类和协议（protocol）来共同实现接口的。

 

## 2、接口的意义

面向对象编程中一条重要的经验法则是：对接口编程，而不是对实现编程。即一个对象想要调用另一个对象的方法，往往不会直接采取直接调用的形式。为降低耦合度考虑，通常会在调用者和被调用者中间增加一层抽象的（通常不会变动的）中间层，接口就是中间层的最通用的形式。

 

## 3、Objective-C中的接口与协议protocol

如前所述，Objective-C中使用协议protocol作为支持接口实现的关键词。

如下类A的对象想要调用类B的对象的方法：

Class A中：

```objc
- (void)doSth:(B *)b
{
　　[b doSth];
}
```

增加一个抽象的（通常不会变动的）中间层作为中介，如下：

定义一个protocol：

```objc
@protocol doSthDelegate

(void)doSth;

@end
```

Class A中：

```objc
- (void)doSth:(id<doSthDelegate>) delegate
{
　　if (delegate)

　　[delegate doSth];
}
```

 

## 4、Objective-C中的接口与抽象基类

协议protocol其实是足以支持接口的语法实现的，但对于需要频繁调用的方法来说，未免不够简洁。如在NSObject协议中声明的alloc、dealloc、retain、release和autoRelease等几乎出现在iOS开发各个角落的方法，如果都需要

```objc
if (delegate && delegate respondsToSelector:@selector(func)
{
　　[delegate func];
}
```

那就太过麻烦了。因此NSObject Class出现了。

在NSObject中对NSObject protocol中的方法都做了基本的实现，因而保证了处于NSObject派生链中的子类在调用NSObject protocol中的方法时的可靠性。写法上也变得极为简洁。

 

## 5、抽象基类和协议共存

如4中所述，抽象基类相比于协议，一方面提供了一些方法的基本实现，使得子类不需要重复实现，另一方面能够保持语法的简洁。因此，只使用协议而不使用抽象基类，是可行的，但极不方便。

而单独使用抽象基类支持接口的语法，是基本上不可行的。如SDK中的UIView及其子类，由于Objective-C的单继承限制，这些已经有基类的类，就不能使用接口了。当然，开发者自行定义的类，是可以完全依靠抽象基类进行组织的。

 

## 6、抽象基类和协议如何协作

经典范例就是标题中的NSObject class和NSObject protocol。

在Cocoa Touch中，并非所有的类都派生自NSObject，如NSProxy类(它本身是一个root class)。但是，对NSProxy和NSproxy的派生类的对象的内存控制，仍然采用alloc、dealloc、retain、release和autoRelease是很自然的想法。在不改变NSProxy基类的情况下，就只能通过协议来支持。

在NSObject协议中，声明了retain、release和autoRelease方法（alloc和dealloc在NSProxy有定义），而NSProxy实现了NSObject协议（conform to protocol NSObject），因此可以做到

[proxyObj release];

这样的调用。

 

## 7、总结

在Objective-C中，接口的支持主要由协议protocol来实现，抽象基类用于简化语法、提供通用方法的基础实现。

 

##参考：

【1】Objective-C编程之道

【2】NSObject class and protocol

http://objectmix.com/c/177917-nsobject-class-protocol.html

【3】NSProxy Class Reference

https://developer.apple.com/library/ios/documentation/Cocoa/Reference/Foundation/Classes/NSProxy_Class/Reference/Reference.html

转自：http://www.cnblogs.com/lexingyu/p/3351996.html