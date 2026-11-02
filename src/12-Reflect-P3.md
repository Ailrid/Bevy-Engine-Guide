## 12.1 过程宏

在前面我们说到，bevy依赖rust中的宏来实现反射，现在就让我们来真正看看到底一个宏是如何实现反射的吧。

首先让我们来简明扼要的介绍一下过程宏，这个过程我不会过多的去讲解，因为这里已经有一本足够优秀的书更详细的讨论了关于宏的问题，**如果读者不够熟悉宏，强烈建议阅读本章前先学习：**[The Little Book of Rust Macros](https://lukaswirth.dev/tlborm/introduction.html)。

首先来让我们回忆一下《编译原理》课程（如果你没有学过，那我强烈建议你应该学习一下）中所学的知识。

rust编译器可以被当作一个llvm的巨大前端，他包括了一个编译器前端应该有的几个部分：词法分析器、语法分析器、中间代码生成器等。

在词法分析器中，我们写的代会被词法分析器解析为token流，我们写下的代码变成了一个个标识符、字面值、关键字、符号等不同的token，这是编译器的最小解析单元。

在语法分析器中，这些token会被一个转换为一棵抽象语法树(常被简写为AST)，这是一个树形结构，其树形结构的组织表明了我们缩写的代码的嵌套层级，这时每个节点都有一个明确的语法结构。

在词法分析和语法分析中间，往往还存在着另一个中间数据结构，这个数据结构被称之为标记树（Token Tree），与AST的区别是，标记树只负责描述token的组织结构，而不含有真正的语义和约束信息。**rust中的宏，主要是处理标记树的。**

为什么需要这样设计呢？因为rust编译器极其复杂，AST这样的数据结构经常需要根据rust的版本演进而发生变化，通过把这样一个中间的树形结构暴露出来，并保持稳定，同时这将宏可以处理的语法和AST彻底分开——这意味着你甚至可以用宏写一个编译器，把宏处理的字符串直接翻译成rust代码，在rust里写别的语言，slint就是这么干的！

---

粗略的来分，rust中的宏可以分为声明宏和过程宏两种，而后者又可以细分为三种，因此如果把所有宏的子类型都算上，rust中一共有四种宏。而我们在这里关心的则是后面细分的三种过程宏：**函数宏、属性宏、derive宏（也被叫做派生宏，不过我喜欢直接叫做derive宏）**。

每个宏的基本结构是相同的，一个标记了一个属性的公有函数，这个属性定义了它的过程性宏类型，然后函数返回一个 `TokenStream`。

```rust
// 函数宏
#[proc_macro]
pub fn name(input: TokenStream) -> TokenStream {
    TokenStream::new()
}
// 像这样使用函数宏
fn foo() {
    name!(hello; airlid); // 这里可以是任何可以被转化为token流的表达式
}


// 属性宏
#[proc_macro_attribute]
pub fn name(attr: TokenStream, input: TokenStream) -> TokenStream {
    TokenStream::new()
}

// derive宏
#[proc_macro_derive(Name)]
pub fn my_derive(input: TokenStream) -> TokenStream {
    TokenStream::new()
}

// thiserror中的 #[error] 和 #[from] 就是一个属性宏
// derive(Error)则是一个derive宏, 它下面只能是 enum、struct 或者 union 类型，因为这些是 derive 属性仅可以添加上去的条目。
// 属性宏与 derive 宏的区别在于，属性宏生成的标记是完全替换性质，而 derive 宏生成的标记是追加性质。这也是为什么叫做“派生宏”的原因
#[derive(Debug, Error)]
enum AppError{
    #[error("Config error: {0}")]
    Config(#[from] config::ConfigError),
}
```

## 12.2 syn与quote

// TODO

















