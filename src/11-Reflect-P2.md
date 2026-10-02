## 11.1 Bevy中的反射

在前面我们说到，元编程的实现类型多种多样，有运行时反射、编译期宏、还有UE5那样的外挂反射和Zig这种独树一帜的`comptime`关键字等等。那么Bevy里主要利用了什么？

答案是前两种。Bevy即为我们提供了运行时反射的能力，又利用宏和Rust强大的泛型能力实现了**编译期单态化与静态分发**。其中前者由bevy_reflect这个crate提供，后者则主要在bevy_ecs中实现。

> [!NOTE]
>
> 想一想，为什么bevy要把他们分开，分别使用运行时反射和编译期宏？（提示：运行时反射的优点和代价是什么）

首先让我们从比较容易理解的运行时反射来研究一下, bevy_reflect该如何使用？他的内部工作原理又是什么？

## 11.2 基本概念

### 11.2.1 Reflect与PartialReflect

bevy_reflect的功能建立在两个`trait`上，即`Reflect`和`PartialReflect`，并且利用类型擦除来存储实现了这些trait并且需要反射的数据，要认识反射就必须先搞懂这两个`trait`到底是用来做什么的。

看到这两个名字你可能会想到Rust标准库中 `Eq` 与 `PartialEq`（以及 `Ord` 与 `PartialOrd`），`Partial` 这个前缀代表着“弱化版的约束”或“不完全的语义”。显然在这里Bevy借鉴了Rust标准库中的这些命名，仅仅是通过这些名字和Rust的传统我们就能意识到这样一件事：**任何一个实现了Reflect的类型肯定也实现了PartialReflect**

为什么要区分二者呢？这是因为在编写游戏时，经常会遇到这样的一个需求：需要根据配置文件（可能是json或者任何序列化的文本）在游戏中生成一个合适的实体或者结构体以供后面的代码使用。然而，Rust是一门编译语言，所有类型的结构体已经在编译期完全确定了，所以**你不可能在不重新编译的前提下，真的生成一个全新的结构体定义并实例化。**

为了能够用代码动态捏造出一个原本在Rust代码里根本不存在的结构体，这时候你就必须拥有一个**能够动态创建并任意添加和修改**的某种东西，而`PartialReflect`的`Partial`含义即在此，一个能够动态捏造的**假类型**。

`Reflect`与`PartialReflect`的区别关键就在于此，`Reflect`代表的是真实Rust类型，你可以把一个`dyn Reflect` 转回（Downcast）原始的真实的结构体，而`PartialReflect`对应的结构并不一定在Rust中拥有真实的定义。

> [!NOTE]
>
> 在以前的早期版本中，其实只有`Reflect`并没有`PartialReflect`，当时遇到的问题是：当你从JSON反序列化或者用代码动态创建一个结构体时，它本质上只是个`HashMap`。如果强行让它实现 `Reflect`，当你尝试把这个`HashMap`转换成真正的 `struct` 时就会崩溃。

顺便一提，你还可以在二者之间进行来回的转换，二者的转换关系如下，不言而喻，把一个`Reflect`转换成`PartialReflect`是没问题的，但是反过来就有可能失败。

- [`PartialReflect::as_partial_reflect`](https://docs.rs/bevy_reflect/latest/bevy_reflect/trait.PartialReflect.html#tymethod.as_partial_reflect) for `&dyn PartialReflect`
- [`PartialReflect::as_partial_reflect_mut`](https://docs.rs/bevy_reflect/latest/bevy_reflect/trait.PartialReflect.html#tymethod.as_partial_reflect_mut) for `&mut dyn PartialReflect`
- [`PartialReflect::into_partial_reflect`](https://docs.rs/bevy_reflect/latest/bevy_reflect/trait.PartialReflect.html#tymethod.into_partial_reflect) for `Box<dyn PartialReflect>`

- [`PartialReflect::try_as_reflect`](https://docs.rs/bevy_reflect/latest/bevy_reflect/trait.PartialReflect.html#tymethod.try_as_reflect) for `&dyn Reflect`
- [`PartialReflect::try_as_reflect_mut`](https://docs.rs/bevy_reflect/latest/bevy_reflect/trait.PartialReflect.html#tymethod.try_as_reflect_mut) for `&mut dyn Reflect`
- [`PartialReflect::try_into_reflect`](https://docs.rs/bevy_reflect/latest/bevy_reflect/trait.PartialReflect.html#tymethod.try_into_reflect) for `Box<dyn Reflect>`

### 11.2.2 \#[derive(Reflect)]

为一个结构体或枚举等实现 `Reflect` 非常简单，直接使用派生宏即可。

```rust
use bevy::reflect::Reflect;

#[derive(Reflect)]
struct MyStruct {
    foo: i32
}
```

当你给一个 `struct` 或 `enum` 打上 `#[derive(Reflect)]` 时，它不仅会自动实现 `Reflect` 和 `PartialReflect`，还会暗中为你自动实现另外几个底层反射`trait`。

- `GetTypeRegistration`：允许类型将自己注册到 Bevy 的类型注册表（`TypeRegistry`）中。

- `Typed`：提供该类型的编译期与运行时元数据（如类型名称、路径、包含哪些字段等）。

- 根据你的类型形态，自动实现 `Struct`、`TupleStruct`或 `Enum`。

并**不是所有**Rust类型都能直接 `#[derive(Reflect)]`，必须满足以下条件。

- 类型必须同时实现 `Any + Send + Sync`。类型本身不能包含非静态的引用（例如不能包含 `&'a str`，必须是 `String`）。这是因为反射系统需要在运行时动态识别类型，不能有随时会失效的临时引用。

- 结构体或枚举里的每一个字段（和子元素），本身也必须实现了 `Reflect`。如果某个字段无法或不需要反射（比如第三方库的复杂类型、指针等），可以在字段上加 `#[reflect(ignore)]` 忽略该字段。

- 如果是在枚举（`enum`）上使用 `#[derive(Reflect)]`，所有字段和子元素必须实现 `FromReflect`。这是因为枚举在 Rust中占用的内存大小取决于最大变体，且带有辨识 Tag。动态构建或还原一个枚举变体时，必须能够把反射数据完全还原为真实的Rust值，因此强制要求 `FromReflect`。

在第二条前面我们提了一嘴`#[reflect(ignore)]`，`#[reflect(...)]`可以作用整个结构体或枚举的顶部，也可以只作用于某些特殊的字段上，在这里我们主要只列举出几个用法并在后文详细给出用法，至于全部的用法读者可以查看[文档](https://docs.rs/bevy_reflect/latest/bevy_reflect/derive.Reflect.html)。

- `#[reflect(TypeData)]`。例如 `#[reflect(Debug, PartialEq)]`，会把该类型的 `ReflectDebug` 和 `ReflectPartialEq` 注册到元数据中。这常配合 `#[reflect_trait]` 宏使用，让系统可以在只有 `dyn Reflect` 的情况下调用具体trait的方法。（在后面讲到反射如何配合trait使用时我们将再次回顾这件事）
- `#[reflect(@expr)]`。注入自定义元数据。接受`@`符号后的任何表达式，该表达式解析为实现该表达式的值`Reflect`。如果注册了两个相同类型的属性，则后注册的属性会覆盖前者。

## 11.3 反射用例

现在，让我们来真正看看到底该如何真正的使用bevy_reflect吧。

### 11.3.1 结构体/枚举/元组反射

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

> [!CAUTION]
>
> 如果你有 C++ Qt 的开发经验，看到这里一定会有强烈的既视感。Qt 为了在静态的 C++ 中实现 UI 编辑器绑定和 QML 交互，打造了 **MOC** 与 `Q_PROPERTY` 宏，如果你仔细看看，你会发现就前面我们讨论的内容而言，两者的核心逻辑完全一致：
>
> 1. **代码生成**：利用外部工具（Qt 的 `moc`）或编译器宏（Bevy 的 `#[derive(Reflect)]`）在编译期提取类型元信息。
> 2. **标签注入**：通过属性注解（Qt 的 `Q_PROPERTY` / Bevy 的 `#[reflect(@expr)]`）为字段挂载额外的运行时属性。
> 3. **动态驱动 UI**：编辑器在运行时通过查询元数据中心（`QMetaObject` / `TypeRegistry`），动态决定如何渲染控件（例如自动将带有范围元数据的 `f32` 渲染为 UI 滑动条）。
>
> 这种跨越语言和年代的设计巧合，恰恰证明了在强类型静态语言中构建现代游戏引擎/GUI 框架时，元数据反射机制是通往动态编辑能力的必然选择。

---

前面我们一直在研究的事情是如何给一个 `struct`贴上元数据并获取。现在，我们来研究反射最核心、最基本的功能：**如何在运行时通过字符串或索引，动态地访问、修改甚至凭空捏造数据的属性**。

在正常的Rust代码中，当你实例化一个结构体`let player: Player = Player { id: 123 };`时，编译器在编译期就已经明确知道它的内存布局和类型大小。我们称之为**具体类型**。

但在反射系统中，为了能把不同的类型统一存放在同一个容器或函数参数里，我们需要通过`trait`对象擦除类型信息，因此首先，我们必须将其转换为一个`Box`。

```rust
let reflected: Box<dyn Reflect> = Box::new(player);
assert!(reflected.downcast_ref::<Player>().is_some());

// 利用Reflect上的方法，你甚至可以直接clone而不需要转为具体的类型
let cloned: Box<dyn Reflect> = reflected.reflect_clone().unwrap();
assert_eq!(cloned.downcast_ref::<Player>(), Some(&Player { id: 123 }));
```

在前面我们说，`Reflect`宏会根据你的类型形态，自动实现`Struct`、`TupleStruct`或 `Enum`。

这里说的`Struct`和其他的到底是什么呢？这是bevy_reflect中的一个用来代表数据类型的`Trait`（“继承”自`PartialReflect`）。通过将某些类型的特定操作包装成一个`Trait`，这样就可以将一个`PartialReflect`当作相应的Rust中的原生类型并提供一些特定的方法，这样就不用在通用的`PartialReflect`进行很多危险的操作了。

具体而言，我们拥有如下的这些`trait`，他们均“继承”自`PartialReflect`。

- [`Tuple`](https://docs.rs/bevy/latest/bevy/reflect/tuple/trait.Tuple.html)
- [`Array`](https://docs.rs/bevy/latest/bevy/reflect/array/trait.Array.html)
- [`List`](https://docs.rs/bevy/latest/bevy/reflect/list/trait.List.html)
- [`Set`](https://docs.rs/bevy/latest/bevy/reflect/set/trait.Set.html)
- [`Map`](https://docs.rs/bevy/latest/bevy/reflect/map/trait.Map.html)
- [`Struct`](https://docs.rs/bevy/latest/bevy/prelude/trait.Struct.html)
- [`TupleStruct`](https://docs.rs/bevy/latest/bevy/prelude/trait.TupleStruct.html)
- [`Enum`](https://docs.rs/bevy/latest/bevy/reflect/enums/trait.Enum.html)
- [`Function`](https://docs.rs/bevy/latest/bevy/prelude/trait.Function.html) 

例如`Struct`这个`trait`看起来就是这样，可以看到这里拥有很多`struct`上才有意义的方法，比如使用字符串访问字段等等，这些在元组或枚举上是无效的操作。

```rust
pub trait Struct: PartialReflect {
    // Required methods
    fn field(&self, name: &str) -> Option<&(dyn PartialReflect + 'static)>;
    fn field_mut(
        &mut self,
        name: &str,
    ) -> Option<&mut (dyn PartialReflect + 'static)>;
    fn field_at(&self, index: usize) -> Option<&(dyn PartialReflect + 'static)>;
    fn field_at_mut(
        &mut self,
        index: usize,
    ) -> Option<&mut (dyn PartialReflect + 'static)>;
    fn name_at(&self, index: usize) -> Option<&str>;
    fn index_of_name(&self, name: &str) -> Option<usize>;
    fn field_len(&self) -> usize;
    fn iter_fields(&self) -> FieldIter<'_> ⓘ;

    // Provided methods
    fn to_dynamic_struct(&self) -> DynamicStruct { ... }
    fn get_represented_struct_info(&self) -> Option<&'static StructInfo> { ... }
}
```

通过`dyn PartialReflect`对象上的`reflect_ref`或者`reflect_mut`方法，我们就可以得到这些对应的值，这些方法返回一个`ReflectRef`类型的枚举，包含了一个转换为上面的`Trait`对象的

```rust
// 每个变体都包含一个特性对象，该特性对象具有特定于某种类型的方法。
// 通过以下方式获得PartialReflect::reflect_ref（返回不可变引用）/PartialReflect::reflect_mut（返回可变引用）。
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

根据我们上面的讨论，为了访问这个`Box<dyn Reflect>`中的字段，我们可以将其转换为一个`dyn Struct`。（按照我们前面介绍的，不可变引用是`reflect_ref`，可变引用时调用`reflect_mut`），这些代码看起来就像下面这样。

```rust
let value = reflected.reflect_ref(); // value这时是一个ReflectRef类型
let value = value.as_struct().unwrap(); // value这时是 dyn Struct
let id = value.field("id").unwrap().try_downcast_ref::<u32>();
assert_eq!(id, Some(&123));
```

---

但是，我们目前仍面临一个难题，`dyn Struct`上并没有类似的任何`insert`之类的方法，那么我们该如何添加自己的字段呢？

首先回忆一下，还记得我们前面讨论的关于`Reflect`和`PartialReflect`的关系吗？为了可以在其之上添加本来并不存在的字段，我们必须以某种方式把一个 `dyn Reflect` 转换成`dyn PartialReflect`，这时我们就创造了一个**动态类型**。

通过这里是`to_dynamic`方法，我们可以把一个`Box<dyn Reflect>`转换为一个 `Box<dyn PartialReflect>`，但是，这远不是简单的转换，他不同于`as_partial_reflect`，这时我们的这个对象现在变成了“动态的”。

```rust
// 注意⚠️，这里不是PartialReflect::as_partial_reflect，这里是to_dynamic方法
let mut dynamic: Box<dyn PartialReflect> = reflected.to_dynamic().unwrap();
assert!(dynamic.is_dynamic());
// 一旦转换为了动态类型，就无法在变回reflect了
assert!(dynamic.try_as_reflect().is_none());
```

**这里的关键是，为什么需要动态类型？他内部又是怎样工作的？**

前面我们在`pub trait Struct: PartialReflect`中并没有看到任何能够添加或者删除字段的方法，但有一个方法格外的显眼：`fn to_dynamic_struct(&self) -> DynamicStruct { ... }`。

这说明，bevy_reflect其实为前面我们说的每一种类型的`trait`都实现了对应的动态类型，所以自然而然的，我们拥有下面这些：

- [`DynamicTuple`](https://docs.rs/bevy/latest/bevy/reflect/tuple/struct.DynamicTuple.html)
- [`DynamicArray`](https://docs.rs/bevy/latest/bevy/reflect/array/struct.DynamicArray.html)
- [`DynamicList`](https://docs.rs/bevy/latest/bevy/reflect/list/struct.DynamicList.html)
- [`DynamicMap`](https://docs.rs/bevy/latest/bevy/reflect/map/struct.DynamicMap.html)
- [`DynamicStruct`](https://docs.rs/bevy/latest/bevy/reflect/structs/struct.DynamicStruct.html)
- [`DynamicTupleStruct`](https://docs.rs/bevy/latest/bevy/reflect/tuple_struct/struct.DynamicTupleStruct.html)
- [`DynamicEnum`](https://docs.rs/bevy/latest/bevy/reflect/enums/struct.DynamicEnum.html)

 通过获得这些动态类型，我们就可以真正的进行 [insert](https://docs.rs/bevy/latest/bevy/reflect/structs/struct.DynamicStruct.html#method.insert) 、[remove_at](https://docs.rs/bevy/latest/bevy/reflect/structs/struct.DynamicStruct.html#method.remove_at)、 [remove_by_name](https://docs.rs/bevy/latest/bevy/reflect/structs/struct.DynamicStruct.html#method.remove_by_name)等操纵结构体的方法了。

```rust
// 通过这个动态类型代理，我们可以动态地读取字段
// dyn Struct上有一个fn to_dynamic_struct(&self) -> DynamicStruct { ... }
let mut dynamic_mut = dynamic
    .reflect_mut()
    .as_struct()
    .unwrap()
    .to_dynamic_struct();

dynamic_mut.insert("x", 1u32);
dynamic_mut.insert("y", 2u32);
dynamic_mut.insert("z", 3u32);
```

或者，我们可以干脆直接创建一个`DynamicStruct`并使用，

```rust
#[derive(Reflect, Default, Debug, PartialEq)]
struct MyStruct {
    x: u32,
    y: u32,
    z: u32,
}

let mut dynamic_struct = DynamicStruct::default();
dynamic_struct.insert("x", 1u32);
dynamic_struct.insert("y", 2u32);
dynamic_struct.insert("z", 3u32);

let mut my_struct = MyStruct::default();
// apply方法可以把一个 &dyn PartialReflect类型其中的数据插入到一个实现了Reflet特性的对象上
my_struct.apply(&dynamic_struct);
assert_eq!(my_struct, MyStruct { x: 1, y: 2, z: 3 });
```

对于元组、枚举等数据，和结构体的使用方法是类似的，这里就不再赘述了。

### 11.3.2 函数反射

//TODO

### 11.3.3 trait反射

//TODO

### 11.3.4 序列化

//TODO

## 11.4 补充

//TODO