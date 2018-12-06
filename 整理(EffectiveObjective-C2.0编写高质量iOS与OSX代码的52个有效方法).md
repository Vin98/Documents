> 本文摘自《Effective Objective-C 2.0  编写高质量iOS与OS X代码的52个有效方法》  
> 下文的【要点】均为原书中每点之后的【要点】栏摘抄，注释为本文作者注

# 第1章 熟悉Objective-C
## 4.多用类型常量，少用#define预处理指令
**原因:**  以`#define ANIMATION_DURATION 0.3`为例

1. 这样定义出来的常量没有类型信息
2. 假设此指令声明在某头文件中，那么所有引入了这个头文件的代码，其中的`ANIMATION_DURATION`都会被替换
3. \#define可以重新定义此常量值，编译器给出警告但是允许更改

**更好的写法:**   

	static const NSTimeInterval kAnimationDuration = 0.3;
> **static作用:**该变量仅在定义此变量的编译单元可见。编译器每收到一个编译单元，就会输出一份目标文件(.o)，OC中编译单元通常指每个类的实现文件(.m)，上述声明的变量，作用域仅限于其所在的编译单元生成的目标文件中。如果不加static，编译器会为它创建一个外部符号。此时若是另一个编译单元中也声明了同名变量，那么编译器就会抛出一条错误信息：

>
	duplicate symbol _kAnimationDuration in :
		xxxxxx.o
		xxxxxx.o
> 实际上，如果一个变量既声明为static，又声明为const，编译器便不会创建符号，而是会像#define预处理指令一样，把所有遇到的变量都替换为常值

**常量的命名法:**

1. 若常量局限于某编译单元(也就是实现文件)之内，则在前面加字母k
2. 若常量在类外可见，则通常以类名为前缀

--
> 要点
> 
* 不要用预处理指令定义常量。这样定义出来的常量不含类型信息，编译器只是会在编译前据此执行查找与替换操作。即使有人重新定义了常量值，编译器也不会产生警告信息(==①==)，这将导致应用程序中的常量值不一致。
* 在实现文件中使用static const来定义“只在编译单元内可见的常量”（translation-unit-specific constant）。由于此类常量不在全局符号表中，所以无需为其名称加前缀(==②==)。
* 在头文件中使用extern来声明全局常量，并在相关实现文件中定义其值。这种常量要出现在全局符号表中，所以其名称应加以区隔，通常用与之相关的类名做前置。  

> ==①==:本文写作时Xcode 10会给出警告信息`'xxxxx' macro redefined`  
> ==②==:局限于某编译单元的常量在前面加字母k

## 5.用枚举表示状态、选项、状态码

> 凡是需要以按位或操作来组合的枚举都应使用`NS_OPTIONS`定义。若是枚举不需要互相组合，则应使用`NS_ENUM`来定义。  

**原因:**根据是否要将代码按C++模式编译，`NS_OPTIONS`宏的定义方式也有所不同。如果不按C++编译，那么其展开方式就和`NS_ENUM`相同。若按C++编译，则展开后代码略有不同。原因在于，用按位或运算来操作两个枚举值时，C++编译模式的处理办法和非C++模式不一样。在用或运算操作两个枚举值时，C++认为运算结果的数据类型应该是枚举的底层数据类型。而且C++不允许将这个底层类型“隐式转换”为枚举类型本身，这样在使用的时候就会因为数据类型不匹配而报错。如果想正确编译，就要将结果显示转换，这无疑会使代码变得复杂。因此有以上结论。

> 与一般习惯不同，若是用枚举来定义状态机(switch语句)，则最好不要有default分支。这样的话，如果之后又加了一种状态，那么编译器就会发出警告信息，提示新加入的状态并未在switch分支中处理。    

--
> 要点  
>
* 应该用枚举来表示状态机的状态、传递给方法的选项以及状态码等值，给这些值起个易懂的名字。   
* 如果把传递给某个方法的选项表示为枚举类型，而多个选项又可同时使用，那么就将各选项值定义为2的幂，以便通过按位或操作将其组合起来。
* 用`NS_ENUM`与`NS_OPTIONS`宏来定义枚举类型，并指明其底层数据类型。这样做可以确保是用开发者所选的底层数据类型实现出来的，而不会采用编译器所选的类型。
* 在处理枚举类型的switch语句中不要实现default分支。这样的话，加入新枚举之后，编译器就会提示开发者：switch语句并未处理所有枚举。

#第2章 对象、消息、运行期 
## 6.理解“属性”这一概念
### 属性特质
1. 原子性。默认情况下，由编译器所合成的方法会通过锁定机制确保其原子性。如果属性具备nonatomic特质，则不使用同步锁。  
   
	> 具备atomic特质的获取方法会通过锁定机制来确保操作的原子性。当两个线程读写同一属性，那么不论何时，总能看到有效的属性值。若是不加锁（使用nonatomic语义），那么当其中一个线程正在改写某属性值时，另外一个线程也许会突然闯入，把尚未修改好的属性值读取出来。   
	但是在iOS开发中，所有属性均声明为nonatomic。因为在iOS中使用同步锁的开销较大，带来性能问题。一般情况下并不要求属性必须是原子的，因为这并不能保证“线程安全”。若要实现线程安全的操作，还需采用更为深层的锁定机制才行。例如，一个线程在连续多次读取某属性值的过程中有别的线程同时改写该值，那么即便声明为atomic，也还是会读到不同的属性值。但是在开发Mac OS X程序时，使用atomic属性通常都不会有性能瓶颈。
2. 读写权限   
	* 具备readwrite特质的属性拥有getter与setter。若该属性由@synthesize实现，则编译器会自动生成这两个方法。
	* 具备readonly特质的属性仅拥有getter方法，只有当该属性由@synthesize实现时，编译器才会为其合成获取方法。可以把某个属性对外公开为只读属性，然后在“class-continuation”中将其重新定义为读写属性（27条）。
3. 内存管理语义  
	下面这一组特质仅会影响setter。
	* assign: setter方法只会针对“纯量类型”（scalar type，例如CGFloat或NSInteger等）的简单赋值操作。
	* strong: 此特质表明该属性定义了一种“拥有关系”。为这种属性设置新值时，设置方法会先保留新值，并释放旧值，然后再将新值设置上去。
	* weak: 此特质表明该属性定义了一种“非拥有关系”。为这种属性设置新值时，设置方法既不保留新值，也不释放旧值。此特质同assign类似，然而在属性所指的对象遭到摧毁时，属性值也会清空。
	* unsafe_unretained: 此特质的语义和assign相同，但是它适用于“对象类型”，该特质表达一种“非拥有关系”（不保留，unretained），当目标对象遭到摧毁时，属性值不会自动清空（不安全，unsafe），这一点与weak有区别。
	* copy: 此特质所表达的所属关系与strong类似，然而设置方法不保留新值，而是将其copy。一般用于NSArray、NSDictionary、NSString、block等。  
4. 方法名
	* getter = \<name\>: 指定获取方法的方法名。
	* setter = \<name\>: 指定设置方法的方法名，不常见。  

--
> 要点  
>
* 可以用@property语法来定义对象中所封装的数据。
* 通过“特质”来指定存储数据所需的正确语义。
* 在设置属性所对应的实例变量时，一定要遵从该属性所声明的语义。
* 开发iOS程序时应该使用nonatomic，因为atomic会严重影响性能。

## 7.在对象内部尽量直接访问实例变量
> 除了几种特殊情况外，读取实例变量时采用直接访问的形式，而在设置实例变量的时候通过属性来做。
现有以下两种实现方法；

1. 使用点语法，通过存取方法来访问相关实例变量
	
	``` objc  
	- (NSString *)fullName {
	    return [NSString stringWithFormat:@"%@ %@", self.firstName, self.lastName];
	}
	
	- (void)setFullName:(NSString *)fullName {
	    NSArray *components = [fullName componentsSeparatedByString:@" "];
	    self.firstName = [components objectAtIndex:0];
	    self.lastName = [components objectAtIndex:1];
	}
	
	```

2. 不经由存取方法，直接访问实例变量

	``` objc
	- (NSString *)fullName {
	    return [NSString stringWithFormat:@"%@ %@", _firstName, _lastName];
	}
	
	- (void)setFullName:(NSString *)fullName {
	    NSArray *components = [fullName componentsSeparatedByString:@" "];
	    _firstName = [components objectAtIndex:0];
	    _lastName = [components objectAtIndex:1];
	}
	```

两种写法的区别：  
 
* 由于不经过Objective-C的“方法派发”（method dispatch，参见11条）步骤，所以直接访问实例变量的速度当然比较快。在这种情况下，编译器所生成的代码会直接访问保存对象实例变量的那块内存。
* 直接访问实例变量时，不会调用其“设置方法”，这就绕过了为相关属性所定义的“内存管理语义”。比方说，如果在ARC下直接访问一个声明为copy的属性，那么并不会拷贝该属性，知会保留新值并释放旧值。
* 如果直接访问实例变量，那么不会触发“键值观测”（Key-Value Observing， KVO）通知。这样做是否会产生问题，还取决于具体的对象行为。
* 通过属性来访问有助于排查与之相关的错误，因为可给“获取方法”和/或“设置方法”中新增断点，监控该属性的调用者及其访问时机。

合理的折中方案：在写入实例变量时，通过其“设置方法”来做，而在读取实例变量时，则直接访问。此办法既能提高读取操作的速度，又能控制对属性的写入操作。之所以要通过“设置方法”来写入实例变量，其首要原因在于，这样做能够确保相关属性的”内存管理语义“得以贯彻。

--
> 要点
> 
* 在对象内部读取数据时，应该直接通过实例变量来读，而写入数据时，则应通过属性来写。
* 在初始化方法及dealloc方法中，总是应该直接通过实例变量来读写数据。
* 有时会使用惰性初始化技术配置某份数据，这种情况下，需要通过属性来读取数据。

## 8.理解“对象等同性”这一概念

比较对象时==操作符比较的是两个指针本身，因此结果未必是我们想要的。  
以NSString对象为例：  

``` objc
NSString *foo = @"Badger 123";
NSString *bar = [NSString stringWithFormat:@"Badger %i", 123];
BOOL equalA = (foo == bar); //equalA = NO
BOOL equalB = [foo isEqual:bar]; //equalB = YES
BOOL equalC = [foo isEqualToString:bar]; //equalC = YES
```
NSString类对象实现了一个自己独有的等同性判断方法isEqualString:。传递给该方法的对象必须是NSString，否则结果未定义（undefined）。调用该方法比调用“isEqual：”方法快，后者还要执行额外的步骤，因为不知道首测对象类型。

NSObject协议中有两个用于判断等同性的关键方法： 
 
``` objc 
- (BOOL)isEqual:(id)object;
- (NSUInteger)hash;
```
这两个方法的默认实现是：当且仅当其“指针值”完全相等时，这两个对象才相等。若想在自定义对象中正确覆写这些方法，就必须先理解其约定。如果“isEqual：”方法判定两个对象相等，那么其hash方法也必须返回同一个值。但是，如果两个对象的hash方法返回同一个值，那么“isEqual：”方法未必会认为两者相等。（哈希碰撞）

#### 特殊类所具有的等同性判定方法
如果需要经常判断等同性，那么可能会自己来创建等同性判定方法，因为无需检测参数类型，所以能大大提升检测速度。另一方面，我们想令代码看上去更美观、易读，此动机与NSString类isEqualToString：方法的创建缘由相似，纯粹为了装点门面。   
在编写判定方法时，也应一并覆写isEqual：方法。后者的常见实现方式为：如果受测的参数与接收该消息的对象都属于同一个类，那么就调用自己编写的判定方法，否则就交由超类来判断。
#### 等同性判定的执行深度
NSArray的检测方式为先看两个数组所含对象个数是否相同，若相同，则在每个对应位置的两个对象上调用其“isEqual：”方法。如果对应位置上的对象均相等，那么这两个数组就相等，这叫做“深度等同性判定”。不过有时无须将所有数据逐个比较，只根据其中部分数据即可判明二者是否等同。尤其在实际项目中，需要判断的两个对象一般都具有唯一标识符，此时只需判断唯一标识符即可。
#### 容器中可变类的等同性
把某个对象放入collection之后，就不应该再改变其hash了。collection会把各个对象按照其hash分装到不同的“箱子数组”中。如果某对象在放入“箱子”之后hash变了，那么其现在所处的这个箱子对它来说就是错误的。因此需要确保hash不是根据对象的可变部分计算出来的，或是保证放入collection之后就不再改变对象内容了。   
> 用一个NSMutableSet和几个NSMutableArray对象测试   
> 
> 1. 把一个数组加入到set中：    
> 
	``` objc
	NSMutableSet *set = [NSMutableSet new];
	NSMutableArray *arrayA = [@[@1, @2] mutableCopy];
	[set addObject:arrayA];
	NSLog(@"set = %@", set);
	//Output: set = {((1, 2))}
	```
	现在set里含有一个数组对象，数组中包含两个对象。
> 2. 再向set中加入一个数组，此数组与前一个数组所含的对象相同，顺序也相同，于是，待加入的数组与set中已有的数组是相等的：
> 
	``` objc
	NSMutableArray *arrayB = [@[@1, @2] mutableCopy];
   [set addObject:arrayB];
   NSLog(@"set = %@", set);
   //Output: set = {((1, 2))}}
	```
	此时set里仍然只有一个对象：因为刚才要加入的那个数组对象和set中已有的数组对象相等，所以set不会改变。
> 3. 添加一个和set中已有对象不同的数组：
>
	``` objc
	NSMutableArray *arrayC = [@[@1] mutableCopy];
   [set addObject:arrayC];
   NSLog(@"set = %@", set);
   // Output: set = {((1), (1, 2))}
	```
	由于arrayC与set里已有的对象不相等，所以set里有两个数组了。
> 4. 改变arrayC的内容，令其和最早加入set的那个数组相等：
>
	``` objc
	[arrayC addObject:@2];
    NSLog(@"set = %@", set);
    // Output: set = {((1, 2), (1, 2))}
	```
>	set中出现了两个彼此相等的数组。根据set的语义是不允许出现这种情况的。如果此时copy此set：
>	
	``` objc
	NSSet *setB = [set copy];
   NSLog(@"setB = %@", setB);
   // Output: setB = {((1, 2))}
	```
	复制过来的set又只剩一个对象了。如果把某对象放入set之后又修改其内容，那么后面的行为将很难预料。  

> 结论：把某对象放入collection之后改变其内容将会造成何种后果。并不是说绝对不能这么做，而是说如果真要这么做，那就得注意其隐患，并用相应的代码处理可能发生的问题。

--
> 要点
> 
* 若想检测对象的等同性，请提供“isEqual：”与hash方法。
* 相同的对象必须具有相同的hash，但是两个hash相同的对象却未必相同。
* 不要盲目地逐个检测每条属性，而是应该按照具体需求来指定检测方案。
* 编写hash方法时，应该使用计算速度快而且hash碰撞率低的算法。

## 9.以“类族模式”隐藏实现细节
“类族”是一种很有用的模式，可以隐藏“抽象基类”背后的实现细节。该模式可以灵活应对多个类，将它们的实现细节隐藏在抽象基类后面，以保持接口简洁。用户无须自己创建子类实例，只需调用基类方法来创建即可。
### 创建类族
假设有一个处理雇员的类，每个雇员都有“名字”和“薪水”这两个属性，管理者可以命令其执行日常工作。但是，雇员的工作内容却不同。经理在带领雇员做项目时，无须关心每个人如何完成其工作，仅需指示其开工即可。   
首先定义抽象基类

``` objc
#import <Foundation/Foundation.h>

typedef NS_ENUM(NSUInteger, LJLEmployeeType) {
    LJLEmployeeTypeDeveloper,
    LJLEmployeeTypeDesigner,
    LJLEmployeeTypeFinance,
};

@interface LJLEmployee : NSObject

@property (nonatomic, copy) NSString *name;
@property (nonatomic, assign) NSUInteger salary;

+ (LJLEmployee *)employeeWithType:(LJLEmployeeType)type;

- (void)doADaysWork;

@end

@implementation LJLEmployee

+ (LJLEmployee *)employeeWithType:(LJLEmployeeType)type {
    switch (type) {
        case LJLEmployeeTypeDeveloper:
            return [LJLEmployeeTypeDeveloper new];
            break;
        case LJLEmployeeTypeDesigner:
            return [LJLEmployeeTypeDesigner new];
            break;
        case LJLEmployeeTypeFinance:
            return [LJLEmployeeTypeFinance new];
            break;
    }
}

- (void)doADaysWork {
    //Subclass implement this
}

@end
```
每个实体子类都从基类继承而来。例如

``` objc
@interface LJLEmployeeDeveloper : LJLEmployee

@end

@implementation LJLEmployeeDeveloper

- (void)doADaysWork {
    [self writeCode];
}

@end
```
Objective-C无法指明某个基类是抽象的。于是，开发者通常会在文档中写明类的用法。这种情况下，基类接口一般都没有名为init的成员方法，这暗示该类的实例也许不应该由用户直接创建。还有一种方法可以确保用户不会使用基类实例，那就是在基类的“DoADaysWork：”方法中抛出异常。然而这种做法相当极端，很少有人用。  
如果对象所属的类位于某个类族中，那么在查询其类型信息时就要注意（参见14条）。在employee这个例子中 `[employee isMemberOfClass:[LJLEmployee class]]` 似乎为返回YES，但实际返回为NO，因为employee并非Employee类的实例，而是其某个子类的实例。   
我们经常需要向类族中新增实体子类，不过这么做的时候得留心。在Employee这个例子中，若是没有“工厂方法”的源代码，那就无法向其中新增雇员类别了。然而对于Cocoa中NSArray这样的类族来说，还是有办法新增子类的，但是需要遵守几条规则。

* 子类应该继承自类族中的抽象基类    
	若要编写NSArray类族中的子类，则需令其继承自不可变数组的基类或可变数组的基类。
* 子类应该定义自己的数据存储方式   
	开发者编写NSArray子类时，经常在这个问题上受阻。子类必须用一个实例变量类存放数组中的对象。这似乎与大家预想的不同，我们以为NSArray自己肯定会保存那些对象，所以在子类中就无需再存一份了。但是NSArray本身只不过是包在其他隐藏对象外面的壳，它仅仅定义了所有数组都需要具备的一些接口。对于这个自定义的数组子类来说，可以用NSArray来保存其实例。
* 子类应该覆写超类文档中指明需要覆写的方法。   
	在每个抽象基类中，都有一些子类必须覆写的方法。比如说想要编写NSArray的子类，就需要实现count及“objectAtIndex：”方法。像lastObject这种方法则无需实现，因为基类可以根据前两个方法实现出这个方法。

--
> 要点
> 
* 类族模式可以把实现细节隐藏在一套简单的公共接口后面。
* 系统框架中经常使用类族。
* 从类族的公共抽象基类中继承子类时要当心，若有开发文档，则应首先阅读。

## 10.在既有类中使用关联对象存放自定义数据
有时需要在对象中存放相关信息。这时我们通常会从对象所属的类中继承一个子类，然后改用这个子类对象。然而并非所有情况下都能这么做，有时候类的实例可能是由某种机制所创建的，而开发者无法令这种机制创建出自己所写的子类实例。Objective-C中有一项强大的特性可以解决此问题，这就是“关联对象”（Associated Object）。   
可以给某些对象关联许多其他对象，这些对象通过键来区分。存储对象值时，可以指明“存储策略”，用以维护相应的“内存管理语义”。存储策略由名为objc_AssociationPolicy的枚举所定义，下表列出了该枚举的取值，同时还列出了与之等效的@property属性:假如关联对象成为了属性，那么它就会具备相应的语义。   

<center>

| 					关联类型      				|    等效的@property属性    |
|:--------------------------------------|:-----------------------:|
| OBJC\_ASSOCIATION\_ASSIGN 				|         assign          |
| OBJC\_ASSOCIATION\_RETAIN\_NONATOMIC 	|    nonatomic,retain     |
| OBJC\_ASSOCIATION\_COPY\_NONATOMIC 	|    nonatomic,copy       |
| OBJC\_ASSOCIATION\_RETAIN 				|         retain          |
| OBJC\_ASSOCIATION\_COPY 					|         copy            |

</center>

> 下列方法可以管理关联对象：
>
* void objc\_setAssociatedObject (id object, void * key, id value, objc\_AssociationPolicy policy)  
	此方法以给定的键和策略为某对象设置关联对象值
* id objc\_getAssociatedObject (id object, void * key)   
	此方法根据给定的键从某对象中获取相应的关联对象值。
* void objc\_removeAssociatedObjects (id object)   
	此方法移除指定对象的全部关联对象。

可以把某对象想象成NSDictionary，把关联到该对象的值理解为字典中的条目，于是，存取关联对象的值就相当于在NSDictionary对象上调用[object setObject:valueforKey:key]与[object objectForKey:key]方法。然而两者之间有个重要差别：设置关联对象时用的键是个“不透明的指针”（apaque pointer， 其所指向的数据结构不局限于某种特定类型的指针，参见[此处](https://en.wikipedia.org/wiki/Opaque_pointer)）。如果在两个键上调用“isEqual：”方法的返回值为YES，那么NSDictionary就认为二者相等；然而在设置关联对象值时，若想令两个键匹配到同一个值，则二者必须是完全相同的指针才行。鉴于此，在设置关联对象值时，通常使用静态全局变量做键。

### 关联对象用法举例




