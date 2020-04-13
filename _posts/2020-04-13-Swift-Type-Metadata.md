
[TOC]
# Swift Type Metadata

>最近在捣腾一个的小工具, 不得不把丢弃多年的 Swift 重新捡起来看了看. 中间用到一点反射, 但 Swift 在这方面羸弱的支持实在不堪入目, 但是这篇文章并不想对如何扩展reflection功能做过多阐述, 仅仅提供一点关于`Type Metadta`介绍.

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
`Struct Metadata`在`Common Metadata Layout`的基础上添加了一家三个字端
- **nominal type descriptor** 
  偏移量为1. 
- **[generic argument vector](file:///Users/infiq/Documents/markdown/docs/#toc_3)**
  偏移量为2.
- **A vector of field offsets**
  紧接着`generic argument vector`. 按照结构体中声明的字段顺序, 以字节为单位,存储相对于结构体开始的偏移量(指针大小的整数).

按官方文档的描述应该如下图, 
![](media/15865148686427/15866853684559.jpg)
但是其中红色部分(`nominal type descriptor`)已经标记过时, 无奈只能把Swift源码翻出来看看.
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
官方文档中有个例子, 比如给定一个范型参数<T,U,V>, 那么用C语言结构体表示出的`metadata records`如下
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
除了[common metadata layout](file:///Users/infiq/Documents/markdown/docs/#toc_1)外, 枚举的`metadata records`还包含以下两个字段:
- nominal type descriptor. 偏移量为1
- [**generic argument vector**](file:///Users/infiq/Documents/markdown/docs/#toc_3). 偏移量为2
我们知道Swift中Optional类型实现就是一种枚举类型, 所以`Optional Metadata`和`Enum Metadata`公用相同的基本布局. 但是由于可选型在反射和动态铸造(dynamic-casting)中的重要性, 有自己单独的类型区分.

![](media/15865148686427/15867663532326.jpg)

同样红色部分已经out of date. 
    
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
同样地除了[common metadata layout](file:///Users/infiq/Documents/markdown/docs/#toc_1)外, 还有以下字段:
- **number of elements**. 
  一个指针大小的整数, 偏移量为1.
- **labels string**.
  偏移量2, 一个指向元组元素标签的指针, 标签以UTF-8编码, 空格分隔空字符终止. 比如(x: Int, z: Int)将被encode为字符数组`"x z \0"`.
- **element vector**
  偏移量3, 是一个类型-偏移(type-offset)对数组.第n个元素的type metadata指针的偏移量为 `3+2*n`, 该元素的相对tuple起始位置偏移值存储在`3+2*n+1`位置.
  
## Function Metadata
- [**common metadata layout**](file:///Users/infiq/Documents/markdown/docs/#toc_1)
- **function flags**
  偏移量1. 这个字段包含了诸如语义约定、是否throws、参数数量等信息.
- **result type reference**
  返回结果`type metadata`的引用存储在偏移量`2*`处.如果有多个返回, 该引用一个tuple Metadata record.
- **parameter type vector **
  紧跟着`result type`之后, 包含类型参数对应的 `NumParameters` 类型元数据指针
- **parameter flags vector**
  紧跟着`parameter type vector`之后的可选字段(如果没有任何非默认标志, 则不设置), 包含`NumParameters`的32-bit标志. 这个字段包含了参数参数信息比如是否是 `inout`或者可变的.



