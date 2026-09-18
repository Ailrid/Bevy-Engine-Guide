## 11.1 Bevy中的反射

在前面我们说到，元编程的实现类型多种多样，有运行时反射、编译期宏、还有UE5那样的外挂反射和zig这种独树一帜的`comptime`关键字等等。那么Bevy里主要利用了什么？

答案是前两种。Bevy即为我们提供了运行时反射的能力，又利用宏和Rust强大的泛型能力实现了**编译期单态化与静态分发**。其中前者由bevy_reflect这个crate提供，后者则主要在bevy_ecs中实现。

> [!NOTE]
>
> 想一想，为什么只使用运行时反射或编译期宏？（提示：运行时反射的优点和代价是什么）

首先让我们从比较容易理解的运行时反射来研究一下，bevy_reflect该如何使用？他的内部工作原理又是什么？

## 11.2 基本概念

### Reflect与PartialReflect

bevy_reflect的功能建立在两个trait上，即`Reflect`和`PartialReflect`，并且利用类型擦除来存储实现了这些trait并且需要反射的数据，要认识反射就必须先搞懂这两个trait到底是用来做什么的。

看到这两个名字你可能会想到Rust标准库中 `Eq` 与 `PartialEq`（以及 `Ord` 与 `PartialOrd`），`Partial` 这个前缀代表着“弱化版的约束”或“不完全的语义”。显然在这里Bevy借鉴了Rust标准库中的这些命名，仅仅是通过这些名字和Rust的传统我们就能意识到这样一件事：**任何一个实现了Reflect的类型肯定也实现了PartialReflect**

为什么要区分二者呢？这是因为在编写游戏时，经常会遇到这样的一个需求：需要根据配置文件（可能是json或者任何序列化的文本）在游戏中生成一个合适的实体或者结构体以供后面的代码使用。然而，Rust是一门编译语言，所有类型的结构体已经在编译期完全确定了，所以**你不可能在不重新编译的前提下，真的生成一个全新的结构体定义并实例化。**

为了能够用代码动态捏造出一个原本在Rust代码里根本不存在的结构体，这时候你就必须拥有一个**能够动态创建并任意添加和修改**的某种东西，而`PartialReflect`的`Partial`含义即在此，一个能够动态捏造的**假类型**。

`Reflect`与`PartialReflect`的区别关键就在于此，`Reflect`代表的是真实Rust类型，你可以把一个`dyn Reflect` 转回（Downcast）原始的真实的结构体。

> [!NOTE]
>
> 在以前的早期版本中，其实只有`Reflect`并没有`PartialReflect`，当时遇到的问题是：当你从JSON反序列化或者用代码动态创建一个结构体时，它本质上只是个HashMap。如果强行让它实现 `Reflect`，当你尝试把这个 HashMap转换成真正的 `struct` 时就会崩溃。

顺便一提，你还可以在二者之间进行来回的转换，二者的转换关系如下，不言而喻，把一个Reflect转换成PartialReflect是没问题的，但是反过来就有可能失败。

- [`PartialReflect::as_partial_reflect`](https://docs.rs/bevy_reflect/latest/bevy_reflect/trait.PartialReflect.html#tymethod.as_partial_reflect) for `&dyn PartialReflect`
- [`PartialReflect::as_partial_reflect_mut`](https://docs.rs/bevy_reflect/latest/bevy_reflect/trait.PartialReflect.html#tymethod.as_partial_reflect_mut) for `&mut dyn PartialReflect`
- [`PartialReflect::into_partial_reflect`](https://docs.rs/bevy_reflect/latest/bevy_reflect/trait.PartialReflect.html#tymethod.into_partial_reflect) for `Box<dyn PartialReflect>`

- [`PartialReflect::try_as_reflect`](https://docs.rs/bevy_reflect/latest/bevy_reflect/trait.PartialReflect.html#tymethod.try_as_reflect) for `&dyn Reflect`
- [`PartialReflect::try_as_reflect_mut`](https://docs.rs/bevy_reflect/latest/bevy_reflect/trait.PartialReflect.html#tymethod.try_as_reflect_mut) for `&mut dyn Reflect`
- [`PartialReflect::try_into_reflect`](https://docs.rs/bevy_reflect/latest/bevy_reflect/trait.PartialReflect.html#tymethod.try_into_reflect) for `Box<dyn Reflect>`

### \#[derive(Reflect)]

为一个结构体或枚举等实现 `Reflect` 非常简单，直接使用派生宏即可。

```rust
use bevy::reflect::Reflect;

#[derive(Reflect)]
struct MyStruct {
    foo: i32
}
```

当你给一个 `struct` 或 `enum` 打上 `#[derive(Reflect)]` 时，它不仅会自动实现 `Reflect` 和 `PartialReflect`，还会暗中为你自动实现另外几个底层反射Trait。

- **`GetTypeRegistration`**：允许类型将自己注册到 Bevy 的类型注册表（`TypeRegistry`）中。

- **`Typed`**：提供该类型的编译期与运行时元数据（如类型名称、路径、包含哪些字段等）。

- 根据你的类型形态，自动实现 `Struct`、`TupleStruct`或 `Enum`。

注意！这里说的`Struct`和另外两个，是bevy_reflect中的一个用来代表数据类型的枚举，用来将一个`PartialReflect`当作相应的Rust中的原生类型并提供一些特定的方法，这样就不用在通用的`PartialReflect`进行很多危险的操作了。

```rust
// 对反射类型的“种类”进行不可变枚举。
// 每个变体都包含一个特性对象，该特性对象具有特定于某种类型的方法。
// 通过以下方式获得PartialReflect::reflect_ref。

pub enum ReflectRef<'a> {
    Struct(&'a dyn Struct),
    TupleStruct(&'a dyn TupleStruct),
    Tuple(&'a dyn Tuple),
    List(&'a dyn List),
    Array(&'a dyn Array),
    Map(&'a dyn Map),
    Set(&'a dyn Set),
    Enum(&'a dyn Enum),
    #[cfg(feature = "functions")]
    Function(&'a dyn Function),
    Opaque(&'a dyn PartialReflect),
}
```

并**不是所有**Rust类型都能直接 `#[derive(Reflect)]`，必须满足以下条件。

- 类型必须同时实现 `Any + Send + Sync`。类型本身不能包含非静态的引用（例如不能包含 `&'a str`，必须是 `String`）。这是因为反射系统需要在运行时动态识别类型，不能有随时会失效的临时引用。

- 结构体或枚举里的每一个字段（和子元素），本身也必须实现了 `Reflect`。如果某个字段无法或不需要反射（比如第三方库的复杂类型、指针等），可以在字段上加 `#[reflect(ignore)]` 忽略该字段。

- 如果是在枚举（`enum`）上使用 `#[derive(Reflect)]`，所有字段和子元素必须实现 `FromReflect`。这是因为枚举在 Rust中占用的内存大小取决于最大变体，且带有辨识 Tag。动态构建或还原一个枚举变体时，必须能够把反射数据完全还原为真实的Rust值，因此强制要求 `FromReflect`。

在第二条前面我们提了一嘴`#[reflect(ignore)]`，`#[reflect(...)]`可以作用整个结构体或枚举的顶部，也可以只作用于某些特殊的字段上，在这里我们主要只列举出几个用法并在后文详细给出用法，至于全部的用法读者可以查看[文档](https://docs.rs/bevy_reflect/latest/bevy_reflect/derive.Reflect.html)。

- `#[reflect(TypeData)]`。例如 `#[reflect(Debug, PartialEq)]`，会把该类型的 `ReflectDebug` 和 `ReflectPartialEq` 注册到元数据中。这常配合 `#[reflect_trait]` 宏使用，让系统可以在只有 `dyn Reflect` 的情况下调用具体trait的方法。（在后面讲到反射如何配合trait使用时我们将再次回顾这件事）
- `#[reflect(@expr)]`。注入自定义元数据。接受`@`符号后的任何表达式，该表达式解析为实现该表达式的值`Reflect`。如果注册了两个相同类型的属性，则后注册的属性会覆盖前者。

## 11.3 反射用例

现在，让我们来真正看看到底该如何真正的使用bevy_reflect吧。

### 结构体/枚举/元组反射

首先，让我们来看看如何利用反射来给结构体和枚举等添加一些自定义属性，利用这个功能我们可以方便的添加一些附加元数据在结构体上，观看以下示例：

```rust
#[derive(Reflect)]
struct Slider {
    #[reflect(@RangeInclusive::<f32>::new(0.0, 1.0))]
    // 使用#[reflect(@expr)]属性注入元数据
    #[reflect(@0.0..=1.0_f32)]
    value: f32,
}

// 获得反射的编译时类型信息
let TypeInfo::Struct(type_info) = Slider::type_info() else {
    panic!("expected struct");
};
// 获得对应的字段
let field = type_info.field("value").unwrap();
// 将我们前面存进去的元数据信息重新拿出来
let range = field.get_attribute::<RangeInclusive<f32>>().unwrap();

assert_eq!(*range, 0.0..=1.0);
```

乍一看，如果你习惯使用诸如`js`或者`python`这样的动态类型语言，你可能觉得这些代码有些面熟感觉。没错，就是有一种qt里的属性的既视感，其实qt里的`Q_PROPERTY`宏，以及它背后的元对象系统，本质上和这些代码所做的事情没有区别。而在动态类型语言里，这些完全就是天生的功能（比如python的装饰器语法或js的反射可以直接操纵函数或者类定义）。

对于枚举，我们可以这样使用`#[reflect(@expr)]`，将一些额外的结构体元数据添加到对应的枚举上，不过要记住，这些结构体元数据也必须是`#[derive(Reflect)]`的。

```rust
// 可以是实现Reflect的任何类型：
#[derive(Reflect)]
struct Required;
#[derive(Reflect, PartialEq, Debug)]
struct Tooltip(String);
impl Tooltip {
    fn new(text: &str) -> Self {
        Self(text.to_string())
    }
}

#[derive(Reflect)]
#[reflect(@Required, @Tooltip::new("An ID is required!"))]
struct Id(u8);

let TypeInfo::TupleStruct(type_info) = Id::type_info() else {
    panic!("expected struct");
};

// 可以检查该结构体上是否拥有该类型的元数据属性信息
assert!(type_info.has_attribute::<Required>());

// 动态的获得相应的id
let some_type_id = TypeId::of::<Tooltip>();

let tooltip: &dyn Reflect = type_info.get_attribute_by_id(some_type_id).unwrap();
assert_eq!(
    // 这里我们尝试直接下转为真正的类型
    tooltip.downcast_ref::<Tooltip>(),
    Some(&Tooltip::new("An ID is required!"))
);
```

另外，让我们再补充一下，所谓的`TypeInfo`和`TypeId`到底是什么？

对于`TypeInfo`，这是bevy_reflect里用于标识元数据属性信息的一个枚举，就类似于之前我们看到的`ReflectRef`那样，让我们能够将一个类型的元数据属性信息转换成正确的形式。

```rust
pub enum TypeInfo {
    Struct(StructInfo),
    TupleStruct(TupleStructInfo),
    Tuple(TupleInfo),
    List(ListInfo),
    Array(ArrayInfo),
    Map(MapInfo),
    Set(SetInfo),
    Enum(EnumInfo),
    Opaque(OpaqueInfo),
}
```

对于任何给定的类型，我们可以通过以下四种方式之一检索此值。在前面我们正是使用了第一种方式来获得相应的`TypeInfo`，同样如果我们拥有`TypeRegistry`或对应实例，也可以获得相应的元数据属性。

- [`Typed::type_info`](https://docs.rs/bevy_reflect/latest/bevy_reflect/trait.Typed.html#tymethod.type_info)

-  [`DynamicTyped::reflect_type_info`](https://docs.rs/bevy_reflect/latest/bevy_reflect/trait.DynamicTyped.html#tymethod.reflect_type_info)

- [`PartialReflect::get_represented_type_info`](https://docs.rs/bevy_reflect/latest/bevy_reflect/trait.PartialReflect.html#tymethod.get_represented_type_info)

- [`TypeRegistry::get_type_info`](https://docs.rs/bevy_reflect/latest/bevy_reflect/struct.TypeRegistry.html#method.get_type_info)

而`TypeId` 是Rust标准库提供的一个极其底层的机制，大多数时候你都不用接触他，但一旦要用到它时就变得非常重要。简而言之，`TypeId` 就是Rust编译器在编译期为每一个类型生成的唯一身份证号（u128类型）。比如在Bevy中，所有的Component都存放在`World`里。`World` 底层的哈希表就是以`TypeId`作为Key的。

而上面的 `tooltip.downcast_ref::<Tooltip>()`这一行代码，实际上就是在比较泛型参数 `Tooltip` 的 `TypeId::of::<Tooltip>()`和内部的Id是否一致。

---

//TODO: 动态字符串访问和属性修改

### 函数反射

//TODO

### 动态类型反射

//TODO

### trait反射

//TODO