# 在Rust中利用类型导向的元编程

我研究如何利用Rust编译器的内部机制，通过类型检查器的信息来进行Rust代码的元编程，例如自动在Rust代码中加入垃圾回收机制，并讨论这种做法的优缺点。

本笔记中提到的所有代码均可在[rustc-type-metaprogramming](https://github.com/willcrichton/rustc-type-metaprogramming)仓库中找到。

引言
在静态类型语言中，元编程——即编写能生成代码的代码——具有广泛的用途。它用于提供那些基础语法或类型系统难以表达的抽象概念。举例来说，Rust通过[宏](https://doc.rust-lang.org/book/first-edition/macros.html)来实现简单的模式匹配代码替代（这是比C预处理器更强大、更干净的版本），比如用于实现类似[println!](https://doc.rust-lang.org/1.11.0/std/macro.println!.html)的可变参数和类似[try!](https://doc.rust-lang.org/1.11.0/std/macro.try!.html)的提前返回功能。

```rust
fn main() {
    println!("{} {} {}", "This has", "many", "arguments");
}
```

然而，基于模式匹配的元编程工具的能力仅限于简单的语法替换。很多常见需求需要深入分析语法结构并根据分析结果生成代码，其中一个典型例子便是自定义派生（custom derive）。在此例中，元编程会分析一个结构体，并根据其字段生成代码，比如自动生成[序列化方法](https://github.com/serde-rs/serde)或[SQL查询](http://docs.diesel.rs/diesel/deserialize/trait.Queryable.html)语句。

```rust
#[derive(Serialize)]
struct Point { x: f32, y: f32 }

fn main() {
    let origin: Point = Point { x: 0, y: 0 };
    println!("{}", origin.to_json()); // {"x": 0, "y": 0}
}
```

这类自定义派生实际上是类型导向的元编程的应用实例，因为它们依据结构体字段的类型来确定生成哪种代码。然而，这种方法存在两个局限性：

1. 这种方法仅适用于结构体，因为Rust要求程序员必须显式声明每个字段的类型。在Rust中，有很多类型并非直接声明，而是由编译器推断出来的。

2. 这种处理方式仅限于语法层面，而非语义层面。例如，如果程序员进行以下操作：

```rust
 type MyFloat = f32;

 #[derive(Serialize)]
 struct Point { x: MyFloat, y: f32 }
 ```
*派生器无法识别MyFloat和f32实际上是同一类型。*

更广泛地讲，大多数编译器都不愿对外公开它们的类型系统或其他内部机制。直至今日，编译器大体上还是被当作黑盒处理，它们接受文本文件作为输入，并输出一个可执行二进制文件或错误信息。在最好的情况下，像Rust这样的语言通过过程宏系统对外暴露其语法结构，但却从未提供有关类型、生命周期或属性/中间表示(IR)的API接口。

然而，同时，编译器通过静态分析对其程序的理解比历史上任何时候都要深入。这其中存在一个权衡——减少程序员需要明确提供的程序信息量（同时保持类型与内存安全）可以提高程序员的效率。但是，这种做法在纯文本层面上使得其他人阅读同一段程序代码变得更加困难，因为理解类型和生命周期对于理解代码的功能至关重要。正因如此，IDE（集成开发环境）实际上在努力破解编译器的“黑盒”。微软的团队开发了语言服务器协议（[Language Server Protocol](https://langserver.org/)），旨在为程序导航和编辑提供一个标准化的共用接口，并且他们还致力于开发[Roslyn](https://github.com/dotnet/roslyn/wiki/Roslyn%20Overview)，这是一个旨在开放C#编译器的新API。

从编译器中提取知识的益处不仅仅体现在IDE上。随着越来越多的编译器API的推出，静态类型语言正逐步走向动态类型语言所拥有的灵活性和可扩展性，且不带来额外的负担。这使得使用当前的内省工具（如调试复杂的数据结构、自动生成序列化工具）变得更加便捷，也为未来的语言扩展（如类型导向的宏解析、内嵌的高性能领域特定语言）奠定了基础。因此，让我们探索一下，利用我们目前的编译器已经能够实现多少功能！

## using the rustc API

在我所了解的适度流行的静态类型语言中，Rust的编译器rustc以其文档全面和易于集成而著称。由于rustc是用Rust编写的，因此在Rust代码中调用Rust编译器的函数变得十分简单。接下来，本文将展示如何利用Rust编译器进行Rust代码的类型指导元编程。

先发出一个提示：Rust编译器的API非常不稳定，经常发生变化。这篇文章中的特定代码可能在几周或几个月后就过时了。运行这些代码需要使用夜间版Rust。如果你像我一样，对Rust元编程和编译器修改有着浓厚的兴趣，那么这些细节将帮助你了解如何实际应用编译器的API。否则，你可以把这看作是一个未来展望，假设这些API变得稳定，那么类型指导的元编程会是一番怎样的景象。所有的代码都可以在我的`;[rustc-type-metaprogramming](https://github.com/willcrichton/rustc-type-metaprogramming)`仓库中找到。现在，让我们开始吧！

首先，我们的目标是调用Rust编译器，提取一些Rust代码片段的类型信息。可以在GitHub上找到Rust编译器API的高级文档。我们首先需要创建一个新的crate，并切换到夜间版：


```rust
$ cargo new --bin rustc-type-metaprogramming
$ cd rustc-type-metaprogramming
$ rustup override set nightly-2018-03-19
```

接下来，我们来填写src/main.rs文件：

```rust
#![feature(rustc_private, quote)]
fn main() {}
```

我们利用Rust的特性门机制（feature gates）来明确表明我们意图使用rustc的私有API以及libsyntax中的引用API（我们将在后文进一步探讨这一点）。在这一步，我们可以参考rustc驱动（librustc_driver）来理解Rust编译器是如何在顶层调用其自身函数的，即用户通过命令行调用rustc时的内部操作。特别是，[run_compiler](https://github.com/rust-lang/rust/blob/8aa27ee30972f16320ae4a8887c8f54616fff819/src/librustc_driver/lib.rs#L445]和[compile_input](https://github.com/rust-lang/rust/blob/8aa27ee30972f16320ae4a8887c8f54616fff819/src/librustc_driver/driver.rs#L67)函数展示了从高层视角审视编译器阶段的方法。这方面的简易英文说明也可以在文档中找到。

我们需要执行许多编译器所需但与我们当前任务几乎无关的操作，如提供命令行选项、为一个不存在的源文件创建代码映射、搭建大量的编译器基础架构等。在代码示例中，我用...来省略了这些不那么有趣的样板代码、生命周期等细节，但你可以在我的仓库中找到完整的示例代码。比如，如果我们想对函数`fn main() { let x = 1 + 2; }`进行类型检查，我们的元编程函数将会是这样的：

```rust
fn main() {
    ...

    let krate = {
        ...
        ast::Crate {
            ...
            quote_item!(fn main() { let x = 1 + 2; })
        }
    };

    let hir = driver::phase_2_configure_and_expand(krate, ...);
    ...

    ty::TyCtxt::create_and_enter(hir, ..., |tcx| {
        typeck::check_create(tcx).unwrap();
        println!("Type checked successfully!");
    });
}
```

这个过程包含三个主要步骤：首先，我们需要生成程序的语法表示。一种方式是将程序作为字符串表示，然后运行rustc解析器，例如。

```rust
let prog: &str = "fn main() { let x = 1 + 2; }";
let parser = Parser::new(); // not actually this easy IRL
let func: ast::Item = parser.parse(prog).unwrap();
```

然而，使用引用（quotations）或宏来完成解析任务是一种更加优雅的方法。诸如quote_item!这样的引用可以接收Rust代码作为输入，并将其转换为程序的Rust语法树（AST）表示，正如我们上述所做的那样。接下来，我们需要将该函数封装进一个crate中，因为这是Rust编译器所期待的输入格式。

第二步，我们将AST转化为高级中间表示（HIR），相关介绍可以在[此处](https://rust-lang-nursery.github.io/rustc-guide/hir.html)找到。HIR的语法形式比程序员使用的AST简单，例如，for循环被转换为简单的loop循环。最后，我们通过创建一个类型上下文（tcx）并使用TyCtxt::check_crate来运行类型检查器。如果我们的代码片段像示例中那样通过类型检查，那么这段代码将会执行并打印出成功消息。你可以通过以下方式来验证这一点：

```rust
$ cargo run
Type checked successfully!
```

## 从rustc中提取类型信息

既然我们已经能够运行编译器，下一步我们希望提取它计算出的类型。比方说，我们想打印出示例程序中每个表达式的类型。一种方法是通过递归匹配语句手动遍历语法树，这种方法既繁琐又难以扩展，特别是在AST结构发生变化时。相对地，我们可以采用访问者模式，只针对我们关注的语法树部分定义行为，对其他部分则使用默认实现。

幸运的是，这在Rust编译器中是一个通用模式，因此他们已经实现了我们所需的大部分功能！具体而言，librustc::hir::intravisit提供了一个我们可以实现的[Visitor](https://github.com/rust-lang/rust/blob/8aa27ee30972f16320ae4a8887c8f54616fff819/src/librustc/hir/intravisit.rs#L149)特征，以便遍历HIR树。


```rust
struct TestVisitor {
    tcx: ty::TyCtxt
}

impl Visitor for TestVisitor {
    ...

    fn visit_expr(&mut self, expr: &hir::Expr) {
        let ty = self.tcx.type_of(expr); // not actually this easy IRL
        println!("Node: {:?}, type: {:?}", expr, ty);
        intravisit::walk_expr(self, expr);
    }
}

fn main() {
    ...

    ty::TyCtxt::create_and_enter(hir, ..., |tcx| {
        typeck::check_crate(tcx).unwrap();
        let mut visitor = TestVisitor { tcx: tcx };
        tcx.hir.visit(&mut visitor);
    });
}
```

执行这段代码后，我们得到了如下输出：

```bash
$ cargo run
Node: expr(13: { let x = 1 + 2; }), type: ()
Node: expr(11: 1 + 2), type: i32
Node: expr(9: 1), type: i32
Node: expr(10: 2), type: i32
```

太好了！我们成功访问到了每个表达式及其类型，尽管这些类型从未被明确书写过。代码的运作方式是创建了一个包含类型上下文tcx的访问者TestVisitor，当我们使用type_of询问时，tcx会告诉我们表达式的类型。这个访问者穿行于HIR树中，当发现诸如1+2这样的表达式时，它会调用一个函数来打印出该表达式及其类型。接下来，我们调用walk_expr函数，该函数继续递归访问表达式的各个组成部分，本例中为1和2。需要注意的是，在Rust中，函数体是由块组成的，而块本身也是表达式，因此块{let x = 1 + 2;}作为一个表达式，其返回类型是空元组（即单位类型）。

## Rust的自动GC

利用类型导向的元编程的一个潜在应用是自动化地将垃圾收集技术应用于特定的代码块中。有时，在Rust中处理生命周期可能会比较复杂，因此如果编译器能够默认地自动进行引用计数，那将大大简化开发过程，尽管这在最简单的情况下（同时也是性能最低的情况）可能看起来像下面这样：

```rust
// before
let x: i32 = 1;
let y: i32 = x + 1;

// after
let x: Rc<i32> = Rc::new(1);
let y: Rc<i32> = Rc::new(*x + 1);
```

虽然这种转换的许多步骤可以通过语法来完成，但在转换过程中了解类型可以帮助我们更精细地进行转换（仅转换特定的类型）并生成更好的错误信息（例如，Rc<i32>中的问题实际上是源代码中的i32问题）。为了展示这一概念的最简单实证，我通过一个过程宏实现了这种方法：auto_gc!。比如说，如果我们有一个如下所示的main.rs文件：

```rust
...
fn main() {
    auto_gc! {
        let x = 1;
        let y: i32 = 1 + x;
    };
    println!("{:?}", *y);
}
```

接着运行cargo expand（相当于gcc -E，用于展开宏），这将生成：

```rust
...
fn main() {
    let x: Rc<i32> = Rc::new(1);
    let y: Rc<i32> = Rc::new(*Rc::new(1) + *Rc::new(*x));
    ...
}
```

请注意，尽管原始源码中没有指定，展开后的x变量明确标注了Rc<i32>类型，这是因为我们能够利用Rust类型检查器的信息。

这一实现（源码见此）采用了一个“folder”（而非访问者）来对HIR中每个节点生成输出，基本上保持代码不变，只是在适当的地方插入了解引用和Rc::new的调用。我的代码被封装在Rust的过程宏接口中，这允许在auto_gc!调用中的代码被我函数生成的任意代码所替换。

## 未来的工作
我很高兴能够将这个项目启动。向Rust开发团队致以最高的敬意，感谢他们在编译器文档化工作上的投入。我相信这将为未来带来巨大的好处，不仅对那些希望在编译器上进行创新的人有益，也对于想要将它推向新境界的人，像我一样，非常有帮助。

尽管如此，在实际操作中，这种方法今天面临许多协调性的挑战：

1. HIR并不是设计来像AST那样进行转换的。虽然AST模块提供了Folder特性，但HIR模块却没有，因此我不得不自行实现这一特性。
2. 所有针对代码生成的工具都是基于AST而非HIR。例如，当我在对一个HIR树进行操作时，我需要手动定义HIR与AST之间值的转换，这看起来是大量不必要的工作。而且，没有为HIR设计的引用库。
3. 在编译器内（比如在过程宏定义中）使用编译器（在我自己的代码里）可能会带来风险。我遇到了一个复杂的bug，因为我不小心定义了两个字符串互换上下文，导致关键词被随机变成其他词汇，产生了一些奇怪的结果。

这些问题都不是无法克服的，主要是指出需要更好的库支持来处理和映射HIR构造回AST。我计划进一步探索rustc需要什么以更好地支持类型导向的元编程。
