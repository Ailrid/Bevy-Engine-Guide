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

通过`dyn PartialReflect`对象上的`reflect_ref`或者`reflect_mut`方法，我们就可以得到这些对应的值，这些方法返回一个`ReflectRef`类型的枚举，包含了一个对应的`Trait`对象。

```rust
// 每个变体都包含一个trait对象，该特性对象具有特定于某种类型的方法。
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

最后，这里不得不提到的一件事就是，**当我们使用`#[derive(Reflect)]`来给结构体/枚举添加反射功能时，所有的字段都的生命周期必须都是'static，因此，非静态借用数据无法实现反射。**

```rust

#[derive(Reflect, Default, Debug, PartialEq)]
struct Pos {
    x: u32,
    y: u32,
    z: u32,
}

#[derive(Reflect, Default, Debug, PartialEq)]
struct MyStruct<'a> {
    pos: &'a Pos,// 不能这样写❌
}
```

### 11.3.2 函数反射

前面我们讲了这么几件事：**如何使用反射给一个结构体/枚举本身/某个字段添加属性、如何在运行时通过字符串或索引，动态地访问、修改甚至凭空捏造数据。**

现在我们来看看反射在函数上有什么用。要启用函数反射，首先要在cargo.toml中启用该功能。

```toml
bevy_reflect = {version = "1.95.0",features=["functions"]}
```

在开始之前，先让我们自己想一想，对一个函数而言，反射应该具备什么功能呢？很显然，对于一个函数来说，我们最关心的无非是下面几件东西：**函数标识、每个参数的类型、每个参数的位置、返回值类型。**稍后我们就会看到，完全如我们所料，这些在bevy_reflect里都有正确的对应对象或者获取方法。而且这也是在函数反射中，最为常用的功能之一。

在前面我们说到，创建一个不存在的结构体，我们需要一个动态类型，而对于函数，我们也需要一个[`DynamicFunction`](https://docs.rs/bevy/latest/bevy/reflect/func/struct.DynamicFunction.html)，不过好消息是，大多数时候我们只需要对一个函数简单的调用`into_function`，在背后，这是通过一个[`IntoFunction`](https://docs.rs/bevy/latest/bevy/prelude/trait.IntoFunction.html) `trait`来实现的。

```rust
fn add(a: i32, b: i32) -> i32 {
  a + b
}

// bevy_reflect为0-15个参数的函数、闭包均实现了 IntoFunction trait
// 在这里必须强调的一点是，函数的所有参数的生命周期必须和返回值相同
let mut func: DynamicFunction = add.into_function();
```

这个`DynamicFunction`类型上有一个重要的方法`pub fn info(&self) -> &FunctionInfo`，其返回一个对于[FunctionInfo](https://docs.rs/bevy/latest/bevy/reflect/func/struct.FunctionInfo.html)类型的引用，而这个`FunctionInfo`类型上，就拥有描述一个函数所需要的全部信息。前面我们说到的四种信息，可以通过`FunctionInfo`类型上的这几个方法来获得。

- `pub fn name(&self) -> Option<&Cow<'static, str>>`

- `pub fn arg_count(&self) -> ArgCount`

- `pub fn signatures(&self) -> &[SignatureInfo]`

```rust
// 实际上DynamicFunction上也有name方法，因此你可以直接这样func.name()
let name: Option<&Cow<'static, str>> = func.info().name() ;
let arg_count:ArgCount = func.info().arg_count();
let signatures:&[SignatureInfo] = func.info().signatures();
```

让我们一个个来看看这些东西，首先是`name`，为何返回的是一个`Option`呢？这是因为，当`add`是闭包函数、匿名函数或函数指针是，`name`将会是`None`，对于重载函数，这是基础签名的名称（对于函数重载的讨论我们下面再进行）。

`arg_count`的类型是[ArgCount](https://docs.rs/bevy/latest/bevy/reflect/func/args/struct.ArgCount.html)，`ArgCount`又是一个什么样的类型呢？为什么不直接返回一个`usize`类型呢？答案是，这还是和函数重载有关。由于一个函数可能有多个实现，这时他们的参数个数也会发生变化，因此你应该可以猜到，`ArgCount`的用法和一个二维数组非常相似，类似于这样。

```rust
for arg_len in arg_count.iter(){
    // 这时，arg_len才是一个usize类型
}
```

`signatures`的类型是`&[SignatureInfo]`，你可能也已经猜到了为何这是一个数组切片了，原因同样是因为重载。**不过，有意思的是，对比二者，我也想问，为何上面的`args`不是一个`&[usize]`类型呢？**

[SignatureInfo](https://docs.rs/bevy/latest/bevy/reflect/func/struct.SignatureInfo.html)是一个非常重要的数据类型，其中含有了对于我们最重要的参数和返回值相关的信息。于是我们为了获取这些信息，还需要再往内部嵌套一层，因此同样我们就有了下面的这些方法。

- `pub fn args(&self) -> &[ArgInfo]`

- `pub fn return_info(&self) -> &ReturnInfo`

```rust
let args:&[ArgInfo] =signatures.args();
let return_info:&ReturnInfo = signatures.return_info();
// 实际上 SignatureInfo 也有 pub fn arg_count(&self) -> usize 方法
// 而且，他还有pub fn name(&self) -> Option<&Cow<'static, str>>方法
// 于是我们可以发现一件事，DynamicFunction上的这两个方法实际上都只是简单的调用了 SignatureInfo 而已
```

 [ArgInfo](https://docs.rs/bevy/latest/bevy/reflect/func/args/struct.ArgInfo.html)类型代表了一个确切的函数参数的信息，通过他，我们终于可以获得一个参数的类型信息、名称、在参数表中的位置了。bevy_reflect为我们提供了非常多有用的方法，包括下面这些。

```rust
for arg in args.iter(){
    // pub fn index(&self) -> usize 返回参数在参数列表中的位置索引
    arg.index();
    // fn ty(&self) -> &Type 返回参数的Rust底层类型
    arg.ty();
    // fn type_id(&self) -> TypeId 返回参数类型的TypeId 
    arg.type_id();
    // 检查给出的类型是否匹配,这只比较了类型之间的 TypeId，并未验证它们是否共享相同的类型路径 
    arg.is::<T>();
    // 检查参数是引用/可变引用还是所有权转移
    args.ownership();
}

// pub enum Ownership {
//     Ref,
//     Mut,
//     Owned,
// }
```

对于[ReturnInfo](https://docs.rs/bevy/latest/bevy/reflect/func/struct.ReturnInfo.html)，和`ArgInfo`的区别主要在于，其没有`index()`方法，因为函数的返回值一般只能有一个。

---

在rust中，对于闭包的区分我们有`Fn`和 `FnMut`和`FnOnce`，你可能已经猜到了，我们前面的`DynamicFunction`对应的其实就是`Fn`，而`FnMut`，当然也有一个`DynamicFunctionMut`！

他们的区别和`Fn`和 `FnMut`的区别几乎是一样的，当我们的闭包需要修改外部的变量时，我们就需要使用`into_function_mut()`方法生成一个`DynamicFunctionMut`。

```rust
 let mut count = 0;
 let increment = |amount: i32| count += amount;
 let closure: DynamicFunctionMut = dbg!(increment.into_function_mut());
```

对于其余的获取参数和各种信息的方法，二者基本上是没有区别的，这里就不再赘述。

---

现在，我们还要做另一件事。既然我们已经知道了怎么通过反射查询一个函数的参数信息，那我们便可以通过这些信息组装成一个合适的参数列表，然后动态调用函数。可是，**我们该怎么组装这样一个列表然后调用这个反射的函数对象呢？**

bevy_reflect为我们提供了一个[ArgList](https://docs.rs/bevy/latest/bevy/reflect/func/struct.ArgList.html)类型，这就是我们需要的，一份可以传递给 `DynamicFunction`或`DynamicFunctionMut`的参数列表。我们可以像这样，创建一个参数列表。

```rust
let foo = 123;
let bar = 456;
let mut baz = 789;
let args = ArgList::new()
  // 移动所有权
  .with_owned(foo)
  // 移动所有权并创建一个堆指针
  .with_boxed(Box::new(foo))
  // 不可变引用
  .with_ref(&bar)
  // 可变引用参数
  .with_mut(&mut baz)
  // 一个ArgValue参数
  .with_arg(ArgValue::Ref(&3.14));

// ArgValue参数是一个枚举类型，包含下列一些变体
// pub enum ArgValue<'a> {
//     Owned(Box<dyn PartialReflect>),
//     Ref(&'a (dyn PartialReflect + 'static)),
//     Mut(&'a mut (dyn PartialReflect + 'static)),
// }
```

在`DynamicFunction`和`DynamicFunctionMut`上，我们有一个`call`方法，其正好接受一个`ArgList`类型的引用。

- `pub fn call<'a>(&self, args: ArgList<'a>) -> Result<Return<'a>, FunctionError>`

而在`DynamicFunctionMut`上，我们还有一个额外的`call_once`方法。

- `pub fn call_once<'a>(&self, args: ArgList<'a>) -> Result<Return<'a>, FunctionError>`

二者的区别在于，`DynamicFunctionMut`的`call`，会可变借用被捕获的变量，所以在手动`drop`掉`DynamicFunctionMut`之前，这些被闭包捕获的变量就无法再次使用了。如果你需要在调用函数之后继续使用被捕获的变量，那么就需要`call_once`来自动`drop`掉该对象。

```rust
let mut count = 0;
let increment = |amount: i32| count += amount;
let closure: DynamicFunctionMut = dbg!(increment.into_function_mut());
let args = dbg!(ArgList::new().with_owned(5_i32));
dbg!(closure.call_once(args).unwrap());
```

---

现在，让我们补充介绍一下，**对于函数重载和泛型函数这些情况下该如何正确使用反射呢？**

其实按照正确的说法，rust本身是**不支持函数重载**的。但是在bevy_reflect中，通过一些魔法，我们可以做到函数重载的功能。即一个`DynamicFunction`对象，能够拥有多个`SignatureInfo`和相应的实现，这和C++中的函数重载是相同的。

`DynamicFunction`和`DynamicFunctionMut`上有一个构造函数方法，其签名如下。这里的F是一个实现了`IntoFunction`的类型。

```rust
pub fn with_overload<'a, F, Marker>(self, function: F) -> DynamicFunction<'a>
where
    'env: 'a,
    F: IntoFunction<'a, Marker>


pub fn with_overload<'a, F, Marker>(self, function: F) -> DynamicFunctionMut<'a>
where
    'env: 'a,
    F: IntoFunctionMut<'a, Marker>
```

前面我们说到，bevy_reflect为0-15个参数的函数均实现了`IntoFunction`，也就是说，我们完全可以把任何一个函数拿过来，将两个函数完全**合二为一**，哪怕他们在rust里根本不是一个函数！于是我们就可以写出下面这样**魔鬼般的代码。**

```rust
fn add<T: Add<Output = T>>(a: T, b: T) -> T {
    a + b
}

let mut func = add::<i32>.into_function();

// 直接把泛型化的两个函数单子合并塞到同一个DynamicFunction里！
func = func.with_overload(add::<f32>);

// 在调用call时候我们直接传入ArgList即可，bevy_reflect会为我们决定使用哪个变体
let args = ArgList::default().with_owned(25_i32).with_owned(75_i32);
let result = func.call(args).unwrap().unwrap_owned();
assert_eq!(result.try_take::<i32>().unwrap(), 100);

let args = ArgList::default().with_owned(25.0_f32).with_owned(75.0_f32);
let result = func.call(args).unwrap().unwrap_owned();
assert_eq!(result.try_take::<f32>().unwrap(), 100.0);



fn add_2(a: i32, b: i32) -> i32 {
    a + b
}

fn add_3(a: i32, b: i32, c: i32) -> i32 {
    a + b + c
}

let mut func = add_2.into_function();

// 或者直接合并两个完全不同的函数！
func = func.with_overload(add_3);

let args = ArgList::default().with_owned(25_i32).with_owned(75_i32);
let result = func.call(args).unwrap().unwrap_owned();
assert_eq!(result.try_take::<i32>().unwrap(), 100);

let args = ArgList::default()
    .with_owned(25_i32)
    .with_owned(75_i32)
    .with_owned(100_i32);
let result = func.call(args).unwrap().unwrap_owned();
assert_eq!(result.try_take::<i32>().unwrap(), 200);
```

而对于泛型函数，遗憾的是bevy_reflect无法直接对泛型函数进行反射，泛型函数在 Rust 里不是一个函数，而是一个**函数模板**。它没有独立的 `TypeId`，也没有唯一的签名。不过这并不是完全无法做到的，实际上正如我们能前面的代码中写的那样，**通过手动单态化，仍然可以对单态化之后的泛型函数进行反射。**

###  11.3.3 trait反射

`trait`在编译后有两种形态，这两种形态决定了对于`trait`的反射是更加棘手的。

在静态分发时，`trait`的约束在编译期用掉了，对于编译之后的代码而言，所有的调用都直接内联成了具体的函数地址，在这一点上他和普通的函数是一致的，我们在这里丢掉了`trait`本身的信息。

在动态分发时，编译器会生成一张`vtable`表，里面保存了`trait`所有方法的函数指针，但是这个`vtable`只包含自己以及父`trait`的方法，并没有别的`trait`相关的信息。这就意味着你根本无法从一个`dyn trait`对象上得到任何其他`trait`的信息。

为了把这些信息留到运行时，我们必须要采取一些额外的措施。

假设我们有如下的一个`trait`，并为`Zombie`结构体实现了这个`trait`。

```rust
trait Damageable {
    type Health: Sized + core::ops::Mul<i32, Output = Self::Health>;
    fn damage(&mut self, damage: Self::Health);
}

#[derive(Reflect, PartialEq, Debug)]
struct Zombie {
    health: i32,
}

impl Damageable for Zombie {
    type Health = i32;
    fn damage(&mut self, damage: Self::Health) {
        self.health -= damage;
    }
}
```

一旦我们想要将其变为`dyn Reflect`来使用反射的功能，我们就会丢失掉该`trait`相关的信息。

```rust
 let mut value: Box<dyn Reflect> = Box::new(Zombie { health: 100 });

// 你根本写不出来这些代码,编译器会直接阻止你，因为这是一个dyn Reflect对象，而不是一个dyn Damageable对象
// value.damage(10);
```

如何才能曲线救国呢？不管怎么样，我们必须通过某种方式把这个`trait`相关的信息保存到和类型`Zombie`相关的某种数据结构上。bevy_reflect为我们提供了这样一种数据结构，即[TypeRegistry](https://docs.rs/bevy/latest/bevy/reflect/struct.TypeRegistry.html)。然而我们在前面说到，`trait`并不是一个具体的结构，甚至在静态分发下编译后直接消失了，我们该怎么保存这些数据呢？

聪明的你肯定能想到，如果我们能够编写一个结构体，并为结构体实现一些方法，然后在结构体的这些方法内把调用转发给`trait`，我们就可以做到把**无形变有形**，这样以来就能够保存这些`trait`数据了。而**这个用曲线保存trait信息的结构体，在这里就叫做`TypeData`。顾名思义，这是一个用来保存类型的数据结构，虽然其是一个结构体。**

bevy_reflect为我们提供了这样一个泛型，只要我们实现他，bevy_reflect就可以为我们自动从一个struct里提取出一个

```rust
pub trait CreateTypeData<T, Input = ()>: TypeData {
    /// 创建这个类型的TypeData
    fn create_type_data(input: Input) -> Self;
}
// TypeData只是一个能够下转为其他类型的记号，并没有什么很奇异的方法
pub trait TypeData: Downcast + Send + Sync {
    fn clone_type_data(&self) -> Box<dyn TypeData>;
}
```



---

那么，让我们来尝试一下，应该如何来写这些代码呢？

我们可不想为每个用到`Damageable`的`struct`都编写一个相应的适配，所以该怎么让才能让rust自己为我们来编写呢？在前面我们说到，一个`dyn Reflect`可以被`downcast_mut`为一个具体的类型，而这个方法正好是一个泛型，或许我们能够利用泛型参数来解决这个问题。

让我们先拼凑出这些代码。当使用时，我们让rust编译器为我们根据泛型参数自动推断出应该转换为什么类型。可是这里的T应该是个什么类型才能让我们调用`trait`上的方法呢？嗯，没错，这里的T应该是我们上面说到的`TypeData`，我们先把这件事记下。

```rust
// 这里的T应该是什么类型呢？他应该是我们的TypeData，这个TypeData应该能够转发damage方法调用给真正的trait
let damageable = reflect.downcast_mut::<T>().unwrap();
damageable.damage(damage);
```

由于我们需要转发这个调用，那我们就必须得把这些代码封装成一个闭包，考虑到闭包的参数，我们至少要有两个类型参数：`dyn Reflect`对象和真正的`trait`方法的参数，到这里我们发现我们要面临一个非常棘手的问题，这里的T应该怎么限定呢？

```rust
#[derive(Clone)]
struct ReflectDamageable {
    damage: fn(&mut dyn Reflect, damage: Box<dyn Reflect>),
}

impl<T> CreateTypeData<T> for ReflectDamageable
where
    T: ?,// 这里的T应该是什么约束？
{
    fn create_type_data(_input: ()) -> Self {
        Self {
            damage: |reflect, damage| {
                let damageable = reflect.downcast_mut::<T>().unwrap();
                let damage = damage.take::<T::Health>().unwrap(); 
                damageable.damage(damage);
            },
        }
    }
}

impl ReflectDamageable {
    pub fn damage(&self, reflect: &mut dyn Reflect, damage: Box<dyn Reflect>) {
        (self.damage)(reflect, damage);
    }
}
```

让我们来细细思索一下，假如我们已经能够通过一个`TypeRegistry`和`Zombie`拿到了`ReflectDamageable`这个结构体，这个T应该是一个什么呢？

正确的答案是，这是一个实现了`Reflect`和`Damageable`两个`trait`的泛型。因为如果我们想在一个`Reflect`对象上调用`Damageable`的方法，这就是满足约束的条件。

> [!CAUTION]
>
> 这里的T必须是一个真实存在的结构体，不能是一个动态类型（如我们之前所说）

```rust
impl<T> CreateTypeData<T> for ReflectDamageable
    where
        T: Reflect + Damageable<Health: Reflect>,
{
    //...
}
```

一旦我们完成了这些，我们就可以写出下面这些代码来。注意这里的约束是如何传递的。

**impl ReflectDamageable中damage →struct ReflectDamageable中的damage**

其中后者是bevy_reflect自动调用我们编写的`create_type_data`生成的。这时我们通过泛型紧箍咒强行把`Damageable`添加到了其中，这时rust编译器将从这里把这个约束传播并求解，于是编译器将会反推出 `Zombie: Reflect + Damageable` 必须成立。

```rust
// 把这些信息注册到一个TypeRegistry里
let mut registry = TypeRegistry::default();
registry.register::<Zombie>();
registry.register_type_data::<Zombie, ReflectDamageable>();
// 通过反射对象身上的typeid拿到注册表中的type data
let type_id = value.reflect_type_info().type_id();
let reflect_damageable = registry
    .get_type_data::<ReflectDamageable>(type_id)
    .unwrap();

// 直接通过转发操作，我们就能调用到真正的trait方法
reflect_damageable.damage(value.as_reflect_mut(), Box::new(25i32));
assert_eq!(value.take::<Zombie>().unwrap(), Zombie { health: 75 });
```

---

每次都要编写上面那些胶水代码来转发这些实现实在是太麻烦了，而且还极其容易出错，因此bevy_reflect为我们提供了一个 **\#[reflect_trait]**指令，可以为我们自动实现这些功能，我们可以这样写。

```rust
#[derive(Reflect)]
struct Skeleton {
    health: i32,
}

// 这一条指令自动为我们生成了ReflectHealth这个struct和上面的转发方法
#[reflect_trait]
trait Health {
    fn health(&self) -> i32;
}
impl Health for Skeleton {
    fn health(&self) -> i32 {
        self.health
    }
}

// 然后我们可以直接使用这个ReflectHealth对象
// 注意⚠️我们这里看不到ReflectHealth的定义，他是隐含在#[reflect_trait]中的
// 一般而言对于任何一个trait XXX，这个指令生成的TypeData是形如ReflectXXX的TypeData
registry.register_type_data::<Skeleton, ReflectHealth>();
let type_id = value.reflect_type_info().type_id();
let reflect_health = registry.get_type_data::<ReflectHealth>(type_id).unwrap();
let value: &dyn Health = reflect_health.get(value.as_reflect()).unwrap();
assert_eq!(value.health(), 50);
```

解决！

### 11.3.4 序列化

//TODO

## 11.4 补充

//TODO