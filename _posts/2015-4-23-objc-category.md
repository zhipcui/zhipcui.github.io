---
layout: post
title:  "objc 自定义已存在的类：Category、Extension"
date:   2015-4-23
categories: IOS
tag: IOS

---


objc 提供了自定义已存在类的语言特性：category和extension。


###category

自定义已存在的类，意味着你要修改原来类的行为：添加、修改方法。在一些情况下，你可以通过继承的方式来实现，但是有时候继承并不是很好的方式，比如你不仅想修改该类的行为，还想同时修改该类的子类的行为。如你通过某种方式修改了`NSString`的行为，同时`NSMutableString`也拥有了该行为，这种方式就是category。

#####例子

原来的类为`XYZPerson.h`,添加一个方法:`lastNameFirstNameString:"`

category 头文件：`XYZPerson+XYZPersonNameDisplayAdditions.h`

	#import "XYZPerson.h"
 
	@interface XYZPerson (XYZPersonNameDisplayAdditions)
		- (NSString *)lastNameFirstNameString;
	@end


category 实现文件：`XYZPerson+XYZPersonNameDisplayAdditions.m`
	
	#import "CategoryName.h"
 
	@implementation ClassName (CategoryName)

	- (NSString *)lastNameFirstNameString {
    	return [NSString stringWithFormat:@"%@, %@", self.lastName, self.firstName];
	}	

	@end
	
使用时只需要`#import "XYZPerson+XYZPersonNameDisplayAdditions.h"`




#####重点：
1. 通过category方式修改类行为，该类的子孙类同时获得该行为；
2. category适用于声明类方法、实例方法，但不适用于声明新的变量。在category中，声明新的变量是没有语法问题的，但是编译器并不会合成声明的变量和变量访问方法，即使你提供自己的访问方法，也是没有办法使用声明的变量。

#####category方法名冲突

因为在category声明的方法会添加到原类中，所以你需要特别注意方法名冲突的问题。

如果category声明的方法名与原类中某个方法名相同，或则该类另外一个category的方法名（甚至是子孙类），那么就会发生冲突，在运行时中该方法会被定义为`未定义`。给你自己的类定义category可能不会出现这样的问题，但是使用Cocoa、Cocoa Touch类就要注意了。

下面2种情况要注意了：

1. 你使用的第三方库可能使用了具有相同方法名的category;
2. apple下一代更新中可能提供了具有相同方法名的category;

解决冲突的方法就是给方法提供前缀：

	@interface NSSortDescriptor (XYZAdditions)
	
	+ (id)xyz_sortDescriptorWithKey:(NSString *)key ascending:(BOOL)ascending;
	
	@end




###extension

extentison与category有点类似，但是extension只可以添加到你拥用源代码的类中，这意味这你不可以在Cocoa、Cocoa Touch类中使用extension。

声明类extension在语法上与category类似：

	@interface ClassName ()
 
	@end
	
与category相比，括号中没有提供名字，所以extension也被称为`匿名category`。

与category不同的是，extension可以添加自己属性和实例变量[^1]，如果你在类extension中声明了一个属性，如下：

	@interface XYZPerson ()
	@property NSObject *extraProperty;
	@end

那么编译器就会自动合成相关的访问方法。

你在extension中声明的任何方法，都必须在实现文件中实现。

你在extension中可以声明实例变量：

	@interface XYZPerson () {
    	id _someCustomInstanceVariable;
	}
	...
	@end



#####利用extension来隐藏消息

objc没有提供类似c++的访问控制(private,public,protected),所以在头文件中声明的方法和变量都是`公开`的。

但是我们可以利用extension的特性来达到隐藏信息的目的。

例子：

`XYZPerson`类有一个属性`uniqueIdentifier`,储存了某人的身份证ID. 明显`uniqueIdentifier`应该是`readonly`类型的。

	@interface XYZPerson : NSObject
	...
	@property (readonly) NSString *uniqueIdentifier;
	- (void)assignUniqueIdentifier;
	@end
	
其他的对象是没有办法直接修改`uniqueIdentifier`的。如果一个人还没有身份证ID的话，需要一个方法来获取`uniqueIdentifier`－－`assignUniqueIdentifier：`

为了实现在`XYZPerson`类内部修改`uniqueIdentifier`，需要在extension中重新声明一下该变量

	@interface XYZPerson ()
	@property (readwrite) NSString *uniqueIdentifier;
	@end
 
	@implementation XYZPerson
	...
	@end
	
这样，编译器就会合成该变量的方法方法，在该类的任何方法里面就可以自己修改该属性了。

NOTE: 通过上面的方式可以达到对外隐藏信息的目的，如果类外部尝试直接使用extension的方法会出现编译器错误，但是可以通过其他方式来调用，如`performSelector:...`,如果外面的世界尝试这样的做法，是没有办法避免外部调用的。


[^1]: 属性、实例变量的理解：属性更多的面向外部，而实例变量是面向内部.
