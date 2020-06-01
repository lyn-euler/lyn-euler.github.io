---
layout:     post
title:      Swift Type Metadata
subtitle:   -
date:       2020-04-13
author:     Lyzz
header-img: img/post-bg-swift-metadta.jpg
catalog: true
tags:
    - Swift
    - iOS
---

# Swift Type Metadata

[TOC]

>最近在捣腾一个的小工具, 不得不把丢弃多年的 Swift 重新捡起来看了看. 中间用到一点反射, 但 Swift 在这方面羸弱的支持实在不堪入目, 但是这篇文章并不想对如何扩展reflection功能做过多阐述, 仅仅提供一点关于官方`Type Metadta`介绍和基于5.x版本的更新.

Swift运行时会给每一个程序中的类型(包括每个范型实例)提供**metadata record**. 我们可以基于元数据信息开发一些debugger工具展示类型信息, 或者扩展reflection功能(像HandyJSON). 其实在官方提供的文档中也描述了metadata record可以用于`reflection`, 但目前还是TODO的状态, 所以也期待后续apple官方能提供更好的支持.

Swift中类型的`metadata record`是由编译期和runtime共同决定的. 对于普通的非范型类型`metadata record`是由编译器静态生成的; 而对于范型实例, 以及固有类型(intrinsic types)比如元组、函数、协议组合等, `metadata record`就要到运行时延迟创建.每种类型有且只有唯一的`metadata record`与之对应, 类型相同那么`metadata pointer`也必然相等, 反之亦然.

## Common Metadata Layout
所有的 `metadata records` 有一个通用的header, 包含以下两个字端.
- **value witness table pointer**
  该指针指向一个vtable函数表, 实现了类型值语意. 这个表里提供了一些基本操作比如allocating, copying以及类型值的destroying, 也记录了类型的size、alignment、stride, 以及一些其他基本属性.该指针地址相对于元数据指针偏移量为-1.
  
- **kind**
  该字段用于区别描述元数据描述的类型, 是一个占用一个指针内容大小的integer值, 相对于元数据指针的偏移量为0.
  
    | kind | 类型 | 描述 |
    | --- | --- | --- |
    | 0 | Class metadata | 如果Class被声明成要与Objective-C协作,那么这个字段会被一个指向OC元类型的isa指针替代. 区别于kind的枚举类型, 这个指针肯定是大于2047的|
    | 1 | Struct metadata |  |
    | 2 | Enum metadata |  |
    | 3 | Optional metadata |  |
    | 8 | Opaque metadata | 用于编译器内置原语, 不提供附加的运行时信息 |
    | 9 | Tuple metadata  |  |
    | 10 | Function metadata |  |
    | 12 | Protocol metadata | This is used for protocol types, for protocol compositions, and for the Any type. |
    | 13 | Metatype metadata |  |
    | 14 | Objective C class wrapper metadata |  |
    | 15 | Existential metatype metadata |  |

## Struct Metadata
`Struct Metadata`在`Common Metadata Layout`的基础上添加了一家三个字段
- **nominal type descriptor** 
  偏移量为1.
   
- **[generic argument vector](./#toc_3)**
  偏移量为2.
  
- **A vector of field offsets**
  紧接着`generic argument vector`. 按照结构体中声明的字段顺序, 以字节为单位,存储相对于结构体开始的偏移量(指针大小的整数).

按官方文档的描述应该如下图, 
![截屏2020-04-1516.47.56](https://ftp.bmp.ovh/imgs/2020/04/cd1c2569cac87929.jpg)

但是其中红色部分(`nominal type descriptor`)已经标记过时, 无奈只能把Swift源码翻出来看了看.
```swift
struct TargetValueMetadata : public TargetMetadata<Runtime> {
  ConstTargetMetadataPointer<Runtime, TargetValueTypeDescriptor>
  getDescription() const {
    return Description;
  }
};

struct TargetContextDescriptor {
  /// Flags describing the context, including its kind and format version.
  ContextDescriptorFlags Flags;
  
  /// The parent context, or null if this is a top-level context.
  TargetRelativeContextPointer<Runtime> Parent;

}

class TargetTypeContextDescriptor
    : public TargetContextDescriptor<Runtime> {
public:
  /// The name of the type.
  TargetRelativeDirectPointer<Runtime, const char, /*nullable*/ false> Name;

  /// A pointer to the metadata access function for this type.
  ///
  /// The function type here is a stand-in. You should use getAccessFunction()
  /// to wrap the function pointer in an accessor that uses the proper calling
  /// convention for a given number of arguments.
  TargetRelativeDirectPointer<Runtime, MetadataResponse(...),
                              /*Nullable*/ true> AccessFunctionPtr;
  
  /// A pointer to the field descriptor for the type, if any.
  TargetRelativeDirectPointer<Runtime, const reflection::FieldDescriptor,
                              /*nullable*/ true> Fields;
      
  
};

class TargetStructDescriptor final
    : public TargetValueTypeDescriptor<Runtime> {
  /// The number of stored properties in the struct.
  /// If there is a field offset vector, this is its length.
  uint32_t NumFields;
  /// The offset of the field offset vector for this struct's stored
  /// properties in its metadata, if any. 0 means there is no field offset
  /// vector.
  uint32_t FieldOffsetVectorOffset;
  ...
    };
    
```
最后整理了下, 这部分最新的结构应该是如下
```C
struct _StructContextDescriptor {
    Int32 Flags, // describing the context , including kind & format version
    Pointer Parent, //
    Pointer Name, // name of the type
    Pointer AccessFunctionPtr, // A pointer to the metadata access function for this type.
    Pointer<_FieldDescriptor> Fields,  // A pointer to the field descriptor for the type, if any.
    Int32 NumFields, //
    Int32 FieldOffsetVectorOffset, // 
}

struct _FieldDescriptor {
    Pointer MangledTypeName,
    Pointer Superclass,
    Int Kind,
    Int16 FieldRecordSize, 
    Int32 numFields,
    [] FieldRecords,
}
```

## generic argument vector
对于范型实例的`Metadata records`都会包含范型参数的信息. 每个范型类型对应一个Type Metadata 引用, 如果范型声明要符合protocol的要求, 还会以声明的顺序存储它需要符合的每个protocol的`witness table`.
官方文档中有个例子, 比如给定一个范型参数`<T,U,V>`, 那么用C-Type结构体表示出的`metadata records`如下
```C
struct GenericParameterVector {
  TypeMetadata *T, *U, *V;
};
```
如果加上一些协议约束比如:`<T: Runcible, U: Fungible & Ansible, V>`, 那么相对的vector会增加对应的witness table的引用.
```C
struct GenericParameterVector {
  TypeMetadata *T, *U, *V;
  RuncibleWitnessTable *T_Runcible;
  FungibleWitnessTable *U_Fungible;
  AnsibleWitnessTable *U_Ansible;
};
```

## Enum Metadata & Optional Metadata
除了[common metadata layout](./#toc_1)外, 枚举的`metadata records`还包含以下两个字段:
- nominal type descriptor. 偏移量为1
- [**generic argument vector**](./#toc_3). 偏移量为2

我们知道Swift中Optional类型实现就是一种枚举类型, 所以`Optional Metadata`和`Enum Metadata`公用相同的基本布局. 但是由于可选型在反射和动态铸造(dynamic-casting)中的重要性, 有自己单独的类型区分.

![](https://ftp.bmp.ovh/imgs/2020/04/ab07839099b4edb3.jpg)

同样红色部分已经out of date. 下面是5.x版本最新的C-Type结构体表示
    
```C
struct _EnumDescriptor {
    Int32 Flags, // describing the context , including kind & format version
    Pointer Parent, //
    Pointer Name, // name of the type
    Pointer AccessFunctionPtr, // A pointer to the metadata access function for this type.
    Pointer<_FieldDescriptor> Fields,  // A pointer to the field descriptor for the type, if any.
    Int32 NumPayloadCasesAndPayloadSizeOffset, // The number of non-empty cases in the enum are in the low 24 bits; the offset of the payload size in the metadata record in words, if any, is stored in the high 8 bits.
    Int32 NumEmptyCases; // The number of empty cases in the enum.
}
```

## Tuple Metadata
同样地除了[common metadata layout](./#toc_1)外, 还有以下字段:
- **number of elements**. 
  一个指针大小的整数, 偏移量为1.
  
- **labels string**.
  偏移量2, 一个指向元组元素标签的指针, 标签以UTF-8编码, 空格分隔空字符终止. 比如(x: Int, z: Int)将被encode为字符数组`"x z \0"`.
  
- **element vector**
  偏移量3, 是一个类型-偏移(type-offset)对数组.第n个元素的type metadata指针的偏移量为 `3+2*n`, 该元素的相对tuple起始位置偏移值存储在`3+2*n+1`位置.
  
## Function Metadata
- [**common metadata layout**](./#toc_1)

- **function flags**
  偏移量1. 这个字段包含了诸如语义约定、是否throws、参数数量等信息.
  
- **result type reference**
  返回结果`type metadata`的引用存储在偏移量`2*`处.如果有多个返回, 该引用一个tuple Metadata record.
  
- **parameter type vector **
  紧跟着`result type`之后, 包含类型参数对应的 `NumParameters` 类型元数据指针
  
- **parameter flags vector**
  紧跟着`parameter type vector`之后的可选字段(如果没有任何非默认标志, 则不设置), 包含`NumParameters`的32-bit标志. 这个字段包含了参数参数信息比如是否是 `inout`或者可变的.
  
  目前Swift官方runtime中提供了以下几个ABI endpoint, 可以获取函数类型的metadata
  
```Swift
const FunctionTypeMetadata *
swift_getFunctionTypeMetadata(FunctionTypeFlags flags,
                              const Metadata *const *parameters,
                              const uint32_t *parameterFlags,
                              const Metadata *result);

SWIFT_RUNTIME_EXPORT
const FunctionTypeMetadata *
swift_getFunctionTypeMetadata0(FunctionTypeFlags flags,
                               const Metadata *result);

SWIFT_RUNTIME_EXPORT
const FunctionTypeMetadata *
swift_getFunctionTypeMetadata1(FunctionTypeFlags flags,
                               const Metadata *arg0,
                               const Metadata *result);

SWIFT_RUNTIME_EXPORT
const FunctionTypeMetadata *
swift_getFunctionTypeMetadata2(FunctionTypeFlags flags,
                               const Metadata *arg0,
                               const Metadata *arg1,
                               const Metadata *result);

SWIFT_RUNTIME_EXPORT
const FunctionTypeMetadata *swift_getFunctionTypeMetadata3(
                                                FunctionTypeFlags flags,
                                                const Metadata *arg0,
                                                const Metadata *arg1,
                                                const Metadata *arg2,
                                                const Metadata *result);
```

## Protocol Metadata
- [**common metadata layout**](./#toc_1)

- **layout flags**
  
  偏移量1, 结构如下
  
  | 字段 | 占位 | 描述 |
  | --- | --- | --- |
    | number of witness tables | 0-24, 24 bits&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;| 协议类型的值在其布局中包含witness table的指针数 |
    | special protocol kind | 24-30, 6 bits &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; | 目前只有Error 协议的值为 1. |
    | superclass constraint indicator | 30, 1 bits | 如果有值, 表示协议类型有父类约束 |
    | class constraint | 31, &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;| 如果有设置值, 则表示不是class-constrained,即协议类型可以用于存储结构体, 枚举, 类. 反之, 只有类值(class values)能存储在类型中, 但类型在使用上会更高效 |

- **number of protocols**
  协议组中协议的数量, 偏移量为2. 对于`Any`或`AnyObject`类型的协议, 这个值为0;对于single-protocol类型的协议值为1; 对于协议组`P & Q &...`为协议组中协议的数量.
  
- **superclass type metadata**
  如果layout flags中 `superclass constraint indicator` 有值, 下一个偏移量对应superclass的`type metadta`.
  
- **protocol vector**
  这是一个内联的指针数组, 用于描述每个协议组中的每个协议. 每个指针引用的是一个 Swift [protocol descriptor](./#toc_8)或者Objective-C协议; 如果指向Objective-C协议, 低位会设置标示.对于`Any`或`AnyObject`类型, 没有protocol descriptor数组
  
### Protocol Descriptor
`Protocol Descriptor` 用于描述协议本身的限制(requirements), 同时充当协议本身的句柄. 会被`Protocol metadata`、[`Protocol Conformance Records`](./#toc_9)和`generic requirements`所引用. `Protocol Descriptor`只有在非 @objc 的协议中才会被传家, @objc的协议会以Objective-C元数据呈现.
- context descriptor metadata(目前还没有文档化)
- 协议的16-bit kind-specific 标记, 如下

    | 位置  | 描述 |
    | --- | --- | --- |
    | Bit 0 | 类约束位(class constraint bit) |
    | Bit 1 | 是否具有弹性(resilient) |
    | Bits 2-7 | 特殊协议类型, 目前只有Error被有值1. |
    
- 指向协议名的指针.
- generic requirements number

  协议`requirement signature`中`generic requirements`的数量
  
- number of protocol requirements
  `requirement signature` 中 `protocol requirements` 在 `generic requirements`之后
- associated type names
  一个C字符串, 包含本协议中所有关联类型的名称, 用空格分隔, 按`protocol requirements`中的顺序排列.
- generic requirements
- protocol requirements

### Protocol Conformance Records
用来表示给定类型是否遵循特定的协议. Swift 运行时可以使用该信息, 比如`swift_conformsToProtocol`来查看是否遵循某个协议.包含如下内容:
- protocol descriptor 
  表示相对字段32-bit的偏移量(可能是间接的).低位表示是否为间接偏移, 第二个最低位是保留位,供将来使用
-  conforming type reference
   表示为相对于字段的32位偏移量, 最低2位表示conforming type的内容
   0. `nominal type descriptor`直接引用;
   1. `nominal type descriptor`间接引用;
   2. 预留位;
   3. Objective-C 类对象的指针引用;
-  witness table
  提供对描述一致性本身的`witness table`的访问, 直接的32-bit相对偏移.低位2bits用于描述witness table引用的内容:
  1. 引用一个witness table;
  2. 对无条件一致性的witness table accessor函数的引用;
  3. 对条件一致性的witness table accessor函数的引用;
  4. 预留位;

## Class Metadata
`Class Metadata`被设计成和Objective-C交互(interoperate)的, 所有的`Class Metadata`对 Objective-C 类对象也是有效的. 类的元数据指针被用作元类的值, 所以一个派生类的`metadata record`也可以作为其所有祖先类(ancestor class)有效的元类值.
![Class Metadata](https://ftp.bmp.ovh/imgs/2020/04/ae71b3b80b045180.jpg)
NOTE: 同样红色部分已经过时了, 最新的见下面struct表示.
- **destructor pointer**
  相对元数据指针偏移量为-2的位置, 在`value witness table`之后.当类实例被销毁时, Swift的deallocator会调用该函数.
  
- **isa pointer** 
  偏移量0处, 替换kind字段, 是一个指向兼容Objective-C元类记录的指针.
  
- **super pointer**
  父类指针, 偏移量1处.如果该类是根类, 这个值为null.
  
- 两个为Objective-C运行时保留的字, 偏移量2和3处.

- **rodata pointer**
  偏移量4, 指向一个Objective-C兼容的`rodata record`.该指针的值包含一个tag. 如果是Swift类低位被设置为1, 如果是Objective-C类则为0.
  
- **class flags**
  一个32-bit的字段, 偏移量为5.
  
- **instance address point**
  也是32-bit的字段, 指向类实例对象的指针
  
- **instance size**
  32-bit, 类对象存储的字节数
  
- **instance alignment mask**
  16-bit, 实例对齐方式
  
- **runtime-reserved**
  16-bit, 编译器会将这个值初始化为0.
  
- **class object size**
  32-bit, 类元数据对象存储的总字节数
  
- **class object address point**
 32-bit, This is the number of bytes of storage in the class metadata object.
 
- **nominal type descriptor**
 64位平台上偏移量为8, 32位平台上偏移量为11. 结构如下:
 
    ```C
    struct _ClassContextDescriptor {
        Int32 Flags, // describing the context , including kind & format version
        Pointer Parent, //
        Pointer Name, // name of the type
        Pointer AccessFunctionPtr, // A pointer to the metadata access function for this type.
        Pointer<_FieldDescriptor> Fields,  // A pointer to the field descriptor for the type, if any.
        Pointer SuperclassType, // The type of the superclass, expressed as a mangled type name that can refer to the generic arguments of the subclass type.
        union {
            Int32 MetadataNegativeSizeInWords, //If this descriptor does not have a resilient superclass, this is the negative size of metadata objects of this class (in words)
            Pointer ResilientMetadataBounds, // If this descriptor has a resilient superclass, this is a reference to a cache holding the metadata's extents.
        },
        union {
            Int32 MetadataPositiveSizeInWords, //If this descriptor does not have a resilient superclass, this is the positive size of metadata objects of this class (in words).
            Pointer ExtraClassFlags, // Otherwise, these flags are used to do things like indicating the presence of an Objective-C resilient class stub.
        },
        Int32 NumImmediateMembers, // The number of additional members added by this class to the class metadata.  This data is opaque by default to the runtime, other than as exposed in other members;it's really just  NumImmediateMembers * sizeof(void*) bytes of data. Whether those bytes are added before or after the address point  depends on areImmediateMembersNegative().
        Int32 NumFields, // The number of stored properties in the class, not including its superclasses. If there is a field offset vector, this is its length.
        Int32 FieldOffsetVectorOffset,
        
    }
    ```
 
- 对于swift类的继承结构, 按照根类到派生类的顺序, 存在以下字段:

  1. 父metadta引用, 对于包含标称类型的成员, 这是一个包含类型元数据的引用.对于top-level则为null. ps:目前parent pointer 还都是null, 官方文档还是todo标记.
  2. 如果是范型类, 还会内联它的 generic argument vector.
  3. 内联vtable, 包含按声明顺序实现的函数实现.
  4. field offset vector, 按声明顺序,每个字段的偏移量数组, 跟Objective-C ivar偏移量一样, 对于固定布局的类可以从全局变量静态访问.
  注意:继承结构中Objective-C基类不存在这些字段

## Type Metadata递归依赖
Swift的类型系统是通过应用其他现有类型的构造函数(比如tuple、function、用户自定义的范型)而产生的.对于inductive system来说这是“最小不动点”(least fixed point), 也就意味着不能包含使用自身定义基本信息的无限递归点类型(µ-types).
比如
```swift
typealias IntDict = Dictionary<String, Int>
```
是被允许的, 但是
```swift
typealias RecursiveDict = Dictionary<String, RecursiveDict>
```
就不能直接这么使用, 但是Swift确实允许以基本身份以外的方式表达具有递归依赖的类型.比如class A可以继承`Base<A>`,或者它可以包含一个`(A,A)`的属性.为例能将类型动态重新组合成类型元数据, 以支持这些类型的动态布局, Swift的metadata运行时支持metadata依赖和迭代初始化(iterative initialization)系统.

### Metadata States
整个编译运行阶段一个类型的元数据有可能是处于下面状态中的一个, 而且状态是不可回溯的:
- **abstract**
  此时仅仅存储一些定义类型身份的基本信息: 名字、kind、所包含的特定种类身份信息(比如`nominal type descriptor`和范型参数).
  
- **layout-complete** 
  该状态下metadata还纯粹类型“外部布局”(external layout)的组件, 这是计算任意直接存储类型的值的类型布局所必须的.这个状态的metadata具有有意义的`value witness table`.
  
- **non-transitively complete**
  该状态的metadata已经经历了额外的初始化过程, 用以支持对该类型的基本操作.比如该状态下的metadata将经历必要的“内部布局”(internal layout), 这可能是为了创建类型的值所必须的, 而不是分配内存空间来储存所必要的.比如一个类元数据有一个实例布局, 这不是计算额外布局(external layout)所必须的, 但是在分配实例和创建子类时是必须的.
  
- **complete**
  一个完整的元数据还可以保证从元数据引用的元数据的传递完整性.举个例子, 一个complete状态的 Array<T> 的元数据, 存储在`generic arguments vector`中T的元数据也是complete的.
  
### 瞬态完整性保证(Transitive Completeness Guarantees)
一个complete状态的class的元数据满足以下条件:
- 如果有父类的话, 其父类的元数据状态也是complete的
- 它的范型参数也是complete
- 同样的超类的范型参数是complete的

一个complete状态struct, enum, 或者 optional的元数据满足以下条件:
- 它的范型参数也是complete

tuple 的元数据满足以下条件:
- 每个元素的类型元数据是complete

其他种类的类型元数据不具备完整性保证.具备可传递完整性的类型可能需要两阶段初始化(two-phase initialization). 其他类型的metadata绘制分配是立即声明自己完成, 所以为了可传递完整性保证将显著正价运行时接口和实现的复杂度, 以及可能在分配过程中添加无法恢复的内存开销.

### 完整性保证(Completeness Requirements)
对于绝大部分代码类型元数据必须是`transitively complete`的, 这允许该代码在不显式检查其完整性的情况下使用元数据.其他状态的元数据通常只有在初始化或建立元数据时才会遇到。 特别地，类型元数据状态在以下情况下必须是complete的:
- 作为函数范型参数(元数据访问函数、witness table访问函数、元数据初始化函数除外).
- 用于元类的值,包括作为static或class方法的Self参数、包括initializers
- 用于构建一个opaque已存在的值.

### Metadata Requests and Responses
当要调用一个Metadata获取函数的时候需要提供以下信息:
- metadata需要满足的状态
- 在满足该状态之前是否阻塞callee
access函数返回:
- metadata
- 当前metadata的状态
access函数不会返回null指针, 如果函数调用时元数据还没有被allocated, 那么runtime会阻塞当前线程直到allocation完成.所以该函数至少会返回abstract状态的元数据, 也就是说使用非阻塞方式获取abstract状态的元数据是没意义的.

### Metadata Allocation and Initialization
 为了支持类型元数据之间的递归依赖关系，类型元数据的创建分为两个阶段
 - **allocation** 创建一个abstract metadata.
    `allocation`不能失败, 它应该快速返回, 不做任何的元数据请求.
 - **initialization** 通过状态过程推进metadata.
 `initialization`过程会重复执行, 直到它的状态到达complete.这个过程一次只能由一个线程执行.编译器发送的初始化函数被赋予一定数量的划痕空间(scratch space)，传递给所有执行;在以后的重新执行的过程中, 这可以用来跳过昂贵的或不可重复的步骤.
 `initialization`阶段的任何特定执行都可能由于不满足的依赖而失败。它通过返回一个元数据依赖项(metadata dependency这是一对元数据和该元数据的必需状态)来实现。初始化阶段预期只会对元数据提出非阻塞请求.如果响应不满足要求，则应将返回的元数据和要求作为依赖项提交给调用方. 运行时利用这个依赖项做以下两件事:
 
 1. 它试图将initialization添加到依赖元数据的completion queue中。 如果成功，初始化被认为是阻塞的；一旦依赖的元数据达到所需的状态，它就会被解除。 但是，如果依赖关系由于并发初始化而已经解决，它也可能失败；如果是，初始化将立即恢复。
 2. 如果它成功阻塞一个依赖的初始化、它将检查是否存在无法解决的依赖环, 如果有, 它将上报一个stderr并终止进程, 这取决于正确使用非阻塞请求；由于阻塞请求的环，运行时不会做出任何努力来检测死锁。
 
初始化不能基于有关元数据动态状态的陈旧信息重复报告失败. 例如，它不能在初始化划痕空间中缓存先前执行的元数据状态, 如果发生这种情况，运行时可能会旋转(spin)，反复执行初始化过程，但由于相同的陈旧依赖关系，它会在同一地方失败。
 编译发布的初始化函数只负责确保元数据处于`non-transitively complete`状态.运行时通过返回一个空依赖来代表这一点. 然后运行时将确保传递的完整性.
 如果编译器发送的初始化函数返回依赖项，则元数据的当前状态（abstract与layout-complete）将通过检查**value witness table**标志中的不完整位来确定。 因此，编译器发送的初始化函数要确保此位设置正确。


## 参考
- [TypeMetadata](https://github.com/apple/swift/blob/master/docs/ABI/TypeMetadata.rst)
- [Swift 5 Type Metadata 详解](https://juejin.im/post/5c7513e7e51d451ac30154aa)
- [Swift Source Code](https://github.com/apple/swift)