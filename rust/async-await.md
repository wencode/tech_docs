# Async/Await
2020年3月27日

本文深入探讨了Rust中的协作式多任务处理以及async/await功能。文章详尽地介绍了Rust中async/await的实现机制，涉及Future特性的设计、状态机转换以及Pinning。接着，我们通过创建一个异步键盘任务和一个初级的执行器，向我们的内核引入了async/await的初步支持。

本博客在[GitHub](https://github.com/phil-opp/blog_os)上进行公开开发。如果您遇到任何问题或有疑问，请在GitHub上提交issue。您也可以在文章底部留下评论。关于本文的完整源代码，可以在[post-12](https://github.com/phil-opp/blog_os/tree/post-12)分支中找到。

目录
- [多任务处理](#多任务处理)
  - [抢占式多任务处理](#抢占式多任务处理)
  - [协作多任务处理](#协作多任务处理)
- [Rust中的Async/Await](#rust中的asyncawait)
    - [Futures](#futures)
    - [操作Futures](#操作futures)
    - [Async/Await模式](#asyncawait模式)
    - [执行器与唤醒器](#executors-and-wakers)
    - [合作式多任务处理?](#合作式多任务处理?)
- [实现](#实现)
    - [任务](#异步键盘任务的状态机)
    - [简单的执行器](#简单的执行器)
    - [异步键盘输入](#异步键盘任务的测试)
    - [有唤醒器支持的执行器](#有唤醒器支持的执行器)
- [总结](#总结)
- [下一步](#下一步)
- [评论区](#评论区)


## 多任务处理
大部分操作系统的核心特性之一是多任务处理，也就是同时执行多个任务的能力。举个例子，当你阅读这篇文章时，很可能你的电脑上还开着其他应用程序，比如文本编辑器或终端窗口。即便你只打开了一个浏览器窗口，你的电脑很可能还在后台运行各种任务，如管理桌面窗口、检查更新或对文件进行索引。

虽然表面上看起来所有任务都在同时运行，但实际上在任一时刻，单个CPU核心只能执行一个任务。操作系统通过快速在活动任务之间切换，创造出任务似乎在并行运行的错觉，因为计算机的处理速度非常快，我们大多数时间都察觉不到这些快速切换。

尽管单核CPU一次只能执行一个任务，多核CPU则能够真正并行地执行多个任务。例如，一个8核心的CPU能够同时运行8个任务。我们将在将来的文章中讲解如何配置多核CPU。在本篇文章中，为了简化讨论，我们将主要关注单核CPU。（值得一提的是，所有的多核CPU初始时只有一个核心是活跃的，因此目前我们可以把它们当作单核CPU来对待。）

多任务处理分为两种形式：协作式多任务处理要求任务定期主动释放CPU控制权，以便其他任务能够继续进行。抢占式多任务处理通过操作系统的功能在任意时刻强制暂停某些线程，从而切换到其他线程。下面我们将更进一步详细探讨这两种多任务处理的形式，并讨论它们的优势与局限。

### 抢占式多任务处理
抢占式多任务处理的核心思想在于操作系统控制任务切换的时机。它利用的是操作系统在每次中断发生时都能重新获得对CPU的控制权这一点。因此，每当系统接收到新的输入时，便可实施任务切换。比如，用户移动鼠标或接收到网络数据包时，操作系统就可以进行任务切换。操作系统还可以通过设置硬件定时器，在特定时间后发送中断信号，以此来决定一个任务可以运行多长时间。

下图展示了硬件中断触发下的任务切换流程：

![](https://os.phil-opp.com/async-await/regain-control-on-interrupt.svg)

在图的第一行，CPU正在执行程序A中的任务A1，此时其他所有任务均处于暂停状态。第二行显示，当硬件中断信号到达CPU时，根据硬件中断的描述，CPU立刻停止当前任务A1的执行，跳转至中断描述符表（IDT）中定义的中断处理函数。通过此中断处理函数，操作系统重新获得了CPU的控制权，从而可以切换到任务B1，而非继续执行任务A1。

🔗状态保存  
由于任务可能在任意时刻被中断，它们有可能正处于某些计算的中间过程。为了能够在后续恢复这些任务，操作系统必须备份任务的全部状态，包括[调用栈](https://en.wikipedia.org/wiki/Call_stack)和所有CPU寄存器的值。这一过程被称作[上下文切换](https://en.wikipedia.org/wiki/Context_switch)。

鉴于调用栈可能非常庞大，操作系统通常会为每个任务配置独立的调用栈，而不是在每次任务切换时备份调用栈的内容。这样具有独立堆栈的任务被称为[执行线程](https://en.wikipedia.org/wiki/Thread_(computing))，简称线程。采用为每个任务配置独立栈的做法意味着，在进行上下文切换时，只需保存寄存器内容（包括程序计数器和栈指针）。这种方式大幅降低了上下文切换的性能损耗，这非常关键，因为上下文切换的频率可能高达每秒100次。

🔗讨论  
抢占式多任务处理的一个主要优势在于操作系统能够完全控制任务的执行时间，确保每个任务都能公平地获取CPU时间，无需依赖于任务之间的相互合作。这一点在执行第三方任务或当系统被多个用户共享时显得尤为重要。

抢占式多任务处理的一个缺点是每个任务都需要独立的栈空间。相较于共享栈空间，这会导致每个任务的内存使用量增加，并往往限制了系统能够同时运行的任务数量。另外，每次任务切换时，操作系统都需要保存完整的CPU寄存器状态，这也是一个缺点，因为即便任务只使用了一小部分寄存器，操作系统也需要保存所有寄存器的状态。

抢占式多任务处理和线程是操作系统的核心组成部分，它们使得操作系统能够运行不被信任的用户空间程序。我们将在未来的文章中进一步深入探讨这些概念。但在本篇中，我们将重点讨论协作式多任务处理，它同样为我们的内核提供了重要的功能。

### 协作多任务处理
协作多任务处理的方式与强制在任意时刻暂停任务的方式不同，它让每个任务持续运行，直到任务自主决定释放CPU控制权。这种方式允许任务在最合适的时刻进行暂停，比如在等待某个I/O操作完成时。

协作多任务处理通常在编程语言层面实现，如通过协程或async/await形式。其核心思想是由程序员或编译器在程序中插入yield操作，这些操作会主动释放CPU控制权，从而允许其他任务接替执行。例如，在执行复杂循环的每一轮迭代之后，都可以插入一个yield操作。

协作多任务处理经常与异步操作结合使用。与其等待某个操作完成并在此期间阻碍其他任务运行，不如让异步操作在未完成时返回“未就绪”状态。这样，等待中的任务就可以执行yield操作，让出控制权，以便其他任务可以继续执行。

🔗状态保存  
由于任务自行决定暂停的时机，它们不需要操作系统介入保存状态。任务可以在暂停之前自行保存所需继续执行的确切状态，这通常能带来更优的性能表现。比如，一个刚完成复杂计算的任务可能只需要保存计算的最终结果，因为它不再需要中间的计算结果了。

语言支持下的协作任务实现，常常可以在任务暂停前，备份调用栈中所需的部分。以Rust的async/await实现为例，它会在一个自动生成的结构体中保存所有还需要用到的局部变量。通过在任务暂停前备份调用栈的相关部分，所有任务能共享同一个调用栈，显著减少了每个任务的内存使用量。这种方法使得可以创建几乎无限多的协作任务，而不会导致内存耗尽。

🔗讨论  
协作多任务处理的一个主要弱点是，如果一个任务不遵循协作原则，它可能会无限期地持续执行。这样，一个有缺陷或恶意的任务可能会阻碍其他任务的执行，导致系统运行缓慢甚至完全卡住。因此，只有在确信所有任务都能够相互配合的情况下，才适宜采用协作多任务处理。作为一个负面示例，依赖任意用户级程序的协作来保障操作系统的运行并不明智。

然而，协作多任务处理在性能和内存管理方面提供了显著的优势，这使得它在程序内部，尤其是结合异步操作使用时，成为了一种优秀的选择。考虑到操作系统内核是一个对性能要求极高、需要与异步硬件交互的程序，协作多任务处理成为了实施并发的一个合适选择。

## Rust中的Async/Await
Rust语言通过async/await机制，提供了对协作多任务处理的一流支持。在深入探讨async/await是什么以及其工作原理之前，我们首先需要了解Rust中的futures以及异步编程如何运作。

### Futures
Future是一种表示可能还未就绪的值的数据结构。这个值可以是由另一个任务计算产生的整数，或是从网络上下载的文件。通过使用futures，我们可以在该值真正需要之前，继续执行其他操作，而不是静止等待。

🔗示例  
通过一个简单的示例来说明futures的概念是最直观的：

![](https://os.phil-opp.com/async-await/async-example.svg)

这张序列图展示了主函数如何从文件系统读取文件，然后调用一个名为foo的函数。这一过程执行了两次：一次是通过同步方式调用read_file，另一次是通过异步方式调用async_read_file。

在同步调用中，主函数必须等待文件系统完成文件加载后才能调用foo函数，这意味着它需要再次等待结果。

而在异步的async_read_file调用中，文件系统直接返回一个future，并在后台异步加载文件。这使得主函数能够更早地调用foo函数，并与文件加载过程并行执行。在这个示例中，文件加载甚至在foo函数返回之前就完成了，因此主函数在foo返回后可以直接使用文件，无需额外等待。

### Rust中的Futures
在Rust中，Futures通过Future Trait来实现，其定义如下所示：

```rust
pub trait Future {
    type Output;
    fn poll(self: Pin<&mut Self>, cx: &mut Context) -> Poll<Self::Output>;
}
```

[关联类型](https://doc.rust-lang.org/book/ch19-03-advanced-traits.html#specifying-placeholder-types-in-trait-definitions-with-associated-types)Output定义了异步操作返回值的类型。举个例子，上述图表中的async_read_file函数会返回一个Output设为File类型的Future实例。

[poll](https://doc.rust-lang.org/nightly/core/future/trait.Future.html#tymethod.poll)方法使我们能够检查异步值是否已经准备就绪。它返回一个Poll枚举，其结构如下：

```rust
pub enum Poll<T> {
    Ready(T),
    Pending,
}
```

当值已准备就绪时（比如文件已经完全从磁盘读取完毕），它会以Ready变量形式返回。否则，会返回Pending变量，向调用方表示值还未准备好。

poll方法有两个参数：self: Pin<&mut Self> 和 cx: &mut Context。前者与常规的&mut self引用类似，但不同之处在于Self值被[“固定”](https://doc.rust-lang.org/nightly/core/pin/index.html)在其内存位置。在不先理解async/await的工作原理的情况下，理解Pin及其必要性是比较困难的。因此，我们将在本文后面进行详细说明。

cx: &mut Context参数的作用是向异步任务传递一个[Waker](https://doc.rust-lang.org/nightly/core/task/struct.Waker.html)实例，比如文件系统的加载操作。这个Waker使得异步任务能够通知其已完成（或部分完成），例如文件已从磁盘加载完成。因为主任务知道当Future准备就绪时会接到通知，所以它不需要反复调用poll方法。我们将在本文后续部分，当我们实现我们自己的waker类型时，对这个过程进行更详细的解释。

### 操作Futures
到目前为止，我们已经了解了futures的定义以及poll方法背后的基本概念。但是，我们还不清楚如何有效地操作futures。问题在于futures代表的是异步任务的结果，这些结果可能还没有准备好。在实际应用中，我们经常直接需要这些值来进行后续的计算。那么问题来了：我们需要future的值时，如何高效地获取它呢？

🔗等待Futures  
一种可能的解决方案是等待直到future就绪。这个过程可能是这样的：

```rust
let future = async_read_file("foo.txt");
let file_content = loop {
    match future.poll(…) {
        Poll::Ready(value) => break value,
        Poll::Pending => {}, // do nothing
    }
}
```

这里，我们通过在循环中不断调用poll方法来积极等待future就绪。poll方法的具体参数在这里不重要，因此我们没有展示它们。尽管这种方法可行，但它极其低效，因为它让CPU保持忙碌状态直到所需值变得可用。

一个更高效的方法是阻塞当前线程直到future就绪。当然，这种方法只有在系统支持线程时才可行，因此这种方法至少目前还不适用于我们的内核。即使在支持阻塞的系统上，这通常也不是首选的做法，因为它会将异步任务转换回同步任务，从而失去并行任务带来的潜在性能优势。

🔗Future组合器  
与直接等待相比，另一种选择是使用future组合器。Future组合器是一些方法，如map，它们允许将多个futures链式组合和修改，类似于Iterator Trait的方法。这些组合器不是直接等待future的结果，而是返回一个新的future，这个新的future会在poll被调用时执行映射操作。

例如，一个将Future<Output = String>转换为Future<Output = usize>的简单string_len组合器可能如下所示：

```rust
struct StringLen<F> {
    inner_future: F,
}

impl<F> Future for StringLen<F> where F: Future<Output = String> {
    type Output = usize;

    fn poll(mut self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<T> {
        match self.inner_future.poll(cx) {
            Poll::Ready(s) => Poll::Ready(s.len()),
            Poll::Pending => Poll::Pending,
        }
    }
}

fn string_len(string: impl Future<Output = String>)
    -> impl Future<Output = usize>
{
    StringLen {
        inner_future: string,
    }
}

// Usage
fn file_len() -> impl Future<Output = usize> {
    let file_content_future = async_read_file("foo.txt");
    string_len(file_content_future)
}
```

这段代码未能完全解决固定（[pinning](https://doc.rust-lang.org/stable/core/pin/index.html)）的问题，但作为示例已经足够。其基本思想是，string_len函数将一个给定的Future实例包装进一个新的StringLen结构体，这个结构体同样实现了Future接口。当对包装后的future进行poll操作时，它会继续poll内部的future。如果内部值还未准备好，包装后的future也会返回Poll::Pending。如果内部值已经准备好，字符串将从Poll::Ready变量中提取出来并计算其长度。然后，这个长度被再次封装进Poll::Ready并返回。

通过这个string_len函数，我们可以计算一个异步字符串的长度，而无需等待其就绪。由于该函数又返回了一个Future，调用者不能直接对返回值进行操作，而需要再次利用组合器函数。这样，整个调用过程都变成了异步的，我们可以在某个节点高效地同时等待多个futures，例如，在main函数中。

因为手动编写组合器函数相当复杂，它们通常由库来提供。虽然Rust标准库本身尚未提供任何组合器方法，但半官方的且与no_std兼容的[futures](https://docs.rs/futures/0.3.4/futures/)库却做到了。它的[FutureExt](https://docs.rs/futures/0.3.4/futures/future/trait.FutureExt.html) Trait提供了如[map](https://docs.rs/futures/0.3.4/futures/future/trait.FutureExt.html#method.map)或[then](https://docs.rs/futures/0.3.4/futures/future/trait.FutureExt.html#method.then)这样的高级组合器方法，使得用户可以使用任意闭包来操作结果。

🔗优势  
Future组合器的主要优势在于它们维持了操作的异步性。当与异步I/O接口结合使用时，这种方式能够实现极高的性能。Future组合器以普通结构体及其 Trait实现的形式被实现，使得编译器能够进行深度优化。关于这一点的更多细节，可以参阅介绍在Rust生态中添加futures的[《Rust中的零成本futures》](https://aturon.github.io/blog/2016/08/11/futures/)文章。

🔗劣势  
尽管Future组合器使得能够编写非常高效的代码成为可能，但由于类型系统和基于闭包的接口，它们在某些情况下可能难以使用。比如，考虑以下代码：

```rust
fn example(min_len: usize) -> impl Future<Output = String> {
    async_read_file("foo.txt").then(move |content| {
        if content.len() < min_len {
            Either::Left(async_read_file("bar.txt").map(|s| content + &s))
        } else {
            Either::Right(future::ready(content))
        }
    })
}
```
*[（在playground中尝试）](https://play.rust-lang.org/?version=stable&mode=debug&edition=2018&gist=91fc09024eecb2448a85a7ef6a97b8d8)*

这里，我们读取了foo.txt文件，然后用[then](https://docs.rs/futures/0.3.4/futures/future/trait.FutureExt.html#method.then)组合器串联一个基于文件内容的第二个Future。如果内容的长度小于给定的min_len，我们就读取另一个文件bar.txt，并用[map](https://docs.rs/futures/0.3.4/futures/future/trait.FutureExt.html#method.map)组合器将其内容附加到原始内容上。否则，我们只返回foo.txt的内容。

传递给then的闭包需要使用[move](https://doc.rust-lang.org/std/keyword.move.html)关键字，否则会出现min_len的生命周期错误。if和else必须返回相同类型，这就是需要使用[Either](https://docs.rs/futures/0.3.4/futures/future/enum.Either.html)包装器的原因。因为我们在不同的代码块中返回了不同类型的Future，我们必须使用一个统一类型的包装器来整合它们。[ready](https://docs.rs/futures/0.3.4/futures/future/fn.ready.html)函数用于将一个值包装成一个立即就绪的Future，这个函数在这里是必需的，因为Either期望其包装的值实现Future接口。

可以想见，这在更大的项目中很快就会导致代码非常复杂。特别是当涉及到借用和不同生命周期时，情况会变得尤为复杂。因此，为了使异步代码编写变得简单，Rust投入了大量工作以支持async/await，目标是根本简化异步代码的编写。

## Async/Await模式
async/await的设计理念是允许程序员以编写看似同步的代码的方式来编写代码，但这些代码通过编译器转换成异步代码。这一机制基于两个关键词：async和await。async关键词可以用在函数签名中，把一个同步函数变成一个返回Future的异步函数：

```rust
async fn foo() -> u32 {
    0
}

// the above is roughly translated by the compiler to:
fn foo() -> impl Future<Output = u32> {
    future::ready(0)
}
```

仅有async关键词可能并不那么实用。然而，在async函数内部，可以使用await关键词来获取Future的异步值：

```rust
async fn example(min_len: usize) -> String {
    let content = async_read_file("foo.txt").await;
    if content.len() < min_len {
        content + &async_read_file("bar.txt").await
    } else {
        content
    }
}
```
*[（在playground中试试看）](https://play.rust-lang.org/?version=stable&mode=debug&edition=2018&gist=d93c28509a1c67661f31ff820281d434)*

这个函数直接转换自之前使用组合器函数的示例。通过使用.await操作符，我们无需使用任何闭包或Either类型便可获取Future的值。因此，我们能像编写同步代码一样编写我们的代码，区别仅在于这依然是异步代码。

🔗状态机转换  
在内部，编译器将async函数的内容转换成一个状态机，每次.await调用代表了一个不同的状态。对于上面的示例函数，编译器会创建一个包含如下四个状态的状态机：

![](https://os.phil-opp.com/async-await/async-state-machine-states.svg)

每个状态代表函数中的一个不同的暂停点。“开始”和“结束”状态分别代表函数执行的起始和终点。“等待foo.txt”状态表示函数正在等待第一次async_read_file操作的结果。“等待bar.txt”状态则代表函数在等待第二次async_read_file操作结果的暂停点。

状态机通过让每次poll调用可能引发状态转换来实现Future Trait：

![](https://os.phil-opp.com/async-await/async-state-machine-basic.svg)

该图表用箭头来标示状态转换，并用菱形来表示决策分支。例如，如果foo.txt文件尚未准备好，会沿着标有“no”的路径进入“等待foo.txt”状态。相反，则沿“yes”路径前进。无标题的小红色菱形代表示例函数中if content.len() < 100的条件分支。

我们看到，第一次poll调用启动了函数，并使其运行，直到遇到一个尚未就绪的future。如果路径上的所有futures都已就绪，函数可以一直运行到“结束”状态，在那里它以Poll::Ready的形式返回结果。否则，状态机进入等待状态并返回Poll::Pending。在下一次poll调用时，状态机会从上一个等待状态开始，重试上一次的操作。

🔗保存状态  
为了能够从上一个等待状态继续执行，状态机需要内部跟踪当前状态。此外，它还需要保存下一次poll调用时继续执行所需的所有变量。在这里，编译器的作用显得尤为重要：因为它知道哪些变量在何时被使用，因此它可以自动生成仅包含所需变量的结构体。

作为示例，编译器会为上述示例函数生成如下结构体：

```rust
// The `example` function again so that you don't have to scroll up
async fn example(min_len: usize) -> String {
    let content = async_read_file("foo.txt").await;
    if content.len() < min_len {
        content + &async_read_file("bar.txt").await
    } else {
        content
    }
}

// The compiler-generated state structs:

struct StartState {
    min_len: usize,
}

struct WaitingOnFooTxtState {
    min_len: usize,
    foo_txt_future: impl Future<Output = String>,
}

struct WaitingOnBarTxtState {
    content: String,
    bar_txt_future: impl Future<Output = String>,
}

struct EndState {}
```

在“开始”和“等待foo.txt”状态中，需要保存min_len参数，以便后续与content.len()进行比较。“等待foo.txt”状态进一步保存了一个foo_txt_future，代表通过async_read_file调用返回的future。当状态机继续执行时，需要重新poll这个future，因此必须保存它。

“等待bar.txt”的状态则包含了用于稍后当bar.txt准备就绪时进行字符串拼接的content变量。此外，它还保存了一个表示bar.txt加载进程的bar_txt_future。该结构体不包含min_len变量，因为在进行content.len()比较之后，该变量不再被需要。在“结束”状态，由于函数已经执行完毕，因此不存储任何变量。

请记住，这仅是编译器可能生成的代码的一个示例。结构体的命名和字段布局是实现细节，实际上可能会有所不同。

🔗完整的状态机类型  
虽然精确的编译器生成代码是实现细节，但设想一下为示例函数生成的状态机可能的样子，有助于加深理解。我们已经定义了表示不同状态并包含所需变量的结构体。为了基于这些结构体创建一个状态机，我们可以将它们合并成一个[enum](https://doc.rust-lang.org/book/ch06-01-defining-an-enum.html)类型：

```rust
enum ExampleStateMachine {
    Start(StartState),
    WaitingOnFooTxt(WaitingOnFooTxtState),
    WaitingOnBarTxt(WaitingOnBarTxtState),
    End(EndState),
}
```

在“开始”和“等待foo.txt”状态中，需要保存min_len参数，以便后续与content.len()进行比较。“等待foo.txt”状态进一步保存了一个foo_txt_future，代表通过async_read_file调用返回的future。当状态机继续执行时，需要重新poll这个future，因此必须保存它。

“等待bar.txt”的状态则包含了用于稍后当bar.txt准备就绪时进行字符串拼接的content变量。此外，它还保存了一个表示bar.txt加载进程的bar_txt_future。该结构体不包含min_len变量，因为在进行content.len()比较之后，该变量不再被需要。在“结束”状态，由于函数已经执行完毕，因此不存储任何变量。

请记住，这仅是编译器可能生成的代码的一个示例。结构体的命名和字段布局是实现细节，实际上可能会有所不同。

🔗完整的状态机类型  
虽然精确的编译器生成代码是实现细节，但设想一下为示例函数生成的状态机可能的样子，有助于加深理解。我们已经定义了表示不同状态并包含所需变量的结构体。为了基于这些结构体创建一个状态机，我们可以将它们合并成一个枚举类型：

```rust
enum ExampleStateMachine {
    Start(StartState),
    WaitingOnFooTxt(WaitingOnFooTxtState),
    WaitingOnBarTxt(WaitingOnBarTxtState),
    End(EndState),
}
```

在我们的设计中，每个状态都通过一个独立的枚举变量来表示，并且将相应状态的结构体作为该变量的一个字段。为了处理状态之间的转换，编译器根据提供的示例函数自动生成了Future特性的实现代码：

```rust
impl Future for ExampleStateMachine {
    type Output = String; // return type of `example`

    fn poll(self: Pin<&mut Self>, cx: &mut Context) -> Poll<Self::Output> {
        loop {
            match self { // TODO: handle pinning
                ExampleStateMachine::Start(state) => {…}
                ExampleStateMachine::WaitingOnFooTxt(state) => {…}
                ExampleStateMachine::WaitingOnBarTxt(state) => {…}
                ExampleStateMachine::End(state) => {…}
            }
        }
    }
}
```

这个Future的输出类型是String，因为这是我们示例函数的返回类型。在实现poll函数时，我们通过一个循环内的match语句来检查当前的状态。基本的思路是，尽可能地切换到下一个状态，当无法继续时，我们会明确返回Poll::Pending。

为简化展示，这里的代码是简化版，并未涉及到[固定操作](https://doc.rust-lang.org/stable/core/pin/index.html)、所有权管理、生命周期等复杂内容。因此，以下代码仅为示例，不应直接使用。实际上，编译器生成的代码会正确地处理所有这些问题，尽管可能采用不同的方式来实现。

下面，我们将针对每个match分支分别介绍代码。首先从Start状态开始讲起：

```rust
ExampleStateMachine::Start(state) => {
    // from body of `example`
    let foo_txt_future = async_read_file("foo.txt");
    // `.await` operation
    let state = WaitingOnFooTxtState {
        min_len: state.min_len,
        foo_txt_future,
    };
    *self = ExampleStateMachine::WaitingOnFooTxt(state);
}
```

状态机在Start状态下，表示它处于函数执行的最开始位置。此时，我们执行示例函数体中直到第一个.await之前的所有代码。为处理.await操作，我们改变self状态机的状态为WaitingOnFooTxt，并构建一个WaitingOnFooTxtState结构体。

因为match self {…}语句是在一个循环中执行的，所以接下来会跳转到WaitingOnFooTxt分支：

```rust
ExampleStateMachine::WaitingOnFooTxt(state) => {
    match state.foo_txt_future.poll(cx) {
        Poll::Pending => return Poll::Pending,
        Poll::Ready(content) => {
            // from body of `example`
            if content.len() < state.min_len {
                let bar_txt_future = async_read_file("bar.txt");
                // `.await` operation
                let state = WaitingOnBarTxtState {
                    content,
                    bar_txt_future,
                };
                *self = ExampleStateMachine::WaitingOnBarTxt(state);
            } else {
                *self = ExampleStateMachine::End(EndState);
                return Poll::Ready(content);
            }
        }
    }
}
```

在这个分支中，我们首先尝试调用foo_txt_future的poll函数。如果它未准备好，我们就退出循环并返回Poll::Pending。此时，self保持在WaitingOnFooTxt状态，因此下一次对状态机进行poll调用时，会再次进入这个分支并尝试轮询foo_txt_future。

一旦foo_txt_future准备就绪，我们将结果赋给content变量，并继续执行示例函数中的代码。如果content的长度小于状态结构中保存的最小长度min_len，我们将异步读取bar.txt文件。此时，我们通过更改状态为WaitingOnBarTxt来翻译.await操作。由于我们在一个循环中执行match，执行过程会直接跳到对应新状态的分支，继续poll bar_txt_future。

如果执行到else分支，意味着没有更多的.await操作需要处理。此时，我们到达了函数的结束点，并将content以Poll::Ready的形式返回。同时，我们将当前状态设置为End状态。

WaitingOnBarTxt状态的代码如下所示：

```rust
ExampleStateMachine::WaitingOnBarTxt(state) => {
    match state.bar_txt_future.poll(cx) {
        Poll::Pending => return Poll::Pending,
        Poll::Ready(bar_txt) => {
            *self = ExampleStateMachine::End(EndState);
            // from body of `example`
            return Poll::Ready(state.content + &bar_txt);
        }
    }
}
```

类似于处理WaitingOnFooTxt状态的方式，我们首先尝试轮询bar_txt_future。如果结果仍然是挂起的，我们就退出循环，返回Poll::Pending。如果不是，我们将执行示例函数的最后一个操作：将content变量和来自future的结果进行拼接。随后，我们将状态机更新到End状态，并以Poll::Ready形式返回最终结果。

End状态的代码实现如下：

```rust
ExampleStateMachine::End(_) => {
    panic!("poll called after Poll::Ready was returned");
}
```

根据规则，一旦Future返回了Poll::Ready，就不应再次进行轮询。因此，如果在已经处于End状态时调用了poll函数，我们将触发panic。

至此，我们了解了编译器生成的状态机以及其如何实现Future特性的潜在方式。实际上，编译器采用的生成代码方式有所不同。（如果你对此感兴趣，目前的实现基于协程，但这仅仅是一个实现上的细节。）

关于示例函数本身的代码生成，最后一部分是这样的, 回想一下，函数的声明是这样定义的：

```rust
async fn example(min_len: usize) -> String
```

现在，由于整个函数体由状态机实现，函数需要做的就是初始化状态机并返回它。对应的生成代码可能如下所示：

```rust
fn example(min_len: usize) -> ExampleStateMachine {
    ExampleStateMachine::Start(StartState {
        min_len,
    })
}
```

现在函数不再使用async修饰符，因为它显式地返回一个实现了Future特性的ExampleStateMachine类型。按照预期，状态机以Start状态被构造出来，并且用min_len参数初始化对应的状态结构体。

需要注意的是，这个函数并不启动状态机的执行。这是Rust futures设计的一个基本原则：futures在第一次被轮询之前不会执行任何操作。

### Pinning
在这篇讨论中，我们已经不止一次地遇到了"固定"（pinning）的概念。现在，是时候深入了解什么是固定，以及为什么固定如此重要了。

🔗自引用结构体  
正如之前解释的那样，状态机转换把每个暂停点的局部变量都保存在一个结构体中。对于像我们的示例函数这样的简单例子来说，这个过程相对简单，并没有引起任何问题。但是，当变量之间存在相互引用时，问题就变得复杂了。例如，看看下面这个函数：

```rust
async fn pin_example() -> i32 {
    let array = [1, 2, 3];
    let element = &array[2];
    async_write_file("foo.txt", element.to_string()).await;
    *element
}
```

该函数创建了一个包含1、2、3三个元素的小数组。接着，它创建了一个指向数组最后一个元素的引用，并将这个引用存储在变量element中。然后，它异步地将这个数字转换成字符串并写入到foo.txt文件中。最后，它返回由element引用的那个数字。

因为这个函数仅使用了一次await操作，所以生成的状态机包括三个状态：开始（start）、结束（end）和“等待写操作”（waiting on write）。函数不接受任何参数，所以开始状态对应的结构体是空的。如同之前一样，结束状态的结构体也是空的，因为此时函数已经完成。而“等待写操作”状态的结构体则更加值得关注：

```rust
struct WaitingOnWriteState {
    array: [1, 2, 3],
    element: 0x1001c, // address of the last array element
}
```

我们需要同时存储数组和element变量，因为element变量是函数返回值所必需的，而element又是通过数组来引用的。由于element是一个引用，它实际上存储了一个指向被引用元素的指针（即内存地址）。这里，我们使用0x1001c作为示例内存地址。实际上，它应该是数组字段最后一个元素的地址，所以它依赖于结构体在内存中的具体位置。这种包含内部指针的结构体被称为自引用结构体，因为它们通过自己的某个字段来引用自身。

🔗自引用结构体的问题  
我们的自引用结构体的内部指针引出了一个本质问题，当我们观察它的内存布局时，这个问题变得非常明显：

![](https://os.phil-opp.com/async-await/self-referential-struct.svg)

数组字段从地址0x10014开始，而element字段则从地址0x10020开始。它指向地址0x1001c，因为数组的最后一个元素正位于此地址。目前为止，一切都还正常。但是，当我们将这个结构体移到一个不同的内存地址时，就会遇到问题：

![](https://os.phil-opp.com/async-await/self-referential-struct-moved.svg)

我们稍微移动了结构体，使其现在从地址0x10024开始。这可能发生在我们将结构体作为函数参数传递或者将其分配给另一个栈变量时。问题是element字段仍然指向地址0x1001c，尽管最后一个数组元素现在位于地址0x1002c。因此，这个指针变成了悬空指针，导致在下一次调用poll时出现未定义的行为。

🔗可能的解决方法   
解决悬空指针问题有三种基本方法：

* 在移动时更新指针：这个方法的思路是，每当结构体在内存中移动时就更新内部指针，以保证移动后指针仍然有效。不幸的是，这种方法需要对Rust做出广泛的修改，这将导致潜在的巨大性能损失。原因是需要某种运行时系统来追踪所有结构体字段的类型，并且在每次移动操作时检查是否需要进行指针更新。

* 存储偏移量而不是自引用：为了避免需要更新指针，编译器可以尝试将自引用以结构体起始处的偏移量形式存储。例如，上述"WaitingOnWriteState"结构体中的element字段可以用一个值为8的element_offset字段来存储，因为被引用的数组元素距结构体起始位置有8字节远。因为当结构体移动时偏移量保持不变，所以不需要更新任何字段。

这种方法的问题是，它需要编译器能够检测到所有的自引用。这在编译时是不可能做到的，因为引用的值可能依赖于用户输入，因此我们又需要一个运行时系统来分析引用并正确创建状态结构体。这不仅会引入运行时开销，还会阻碍某些编译器优化，因此会再次造成巨大的性能损失。

* 禁止移动结构体：如上所述，只有在内存中移动结构体时，才会出现悬空指针问题。通过完全禁止自引用结构体的移动操作，也可以避免这个问题。这种方法的一个主要优点是，它可以在类型系统层面上实现，不需要额外的运行时开销。缺点是，它将如何处理可能是自引用的结构体的移动操作的责任放在了开发者身上。

由于Rust遵循提供零成本抽象的原则，即抽象不应增加额外的运行时成本，因此选择了第三种解决方案。为了达到这个目的，[RFC 2349](https://github.com/rust-lang/rfcs/blob/master/text/2349-pin.md)提出了[pinning](https://doc.rust-lang.org/stable/core/pin/index.html) API。下面，我们将对这个API进行简要的介绍，并解释它如何与async/await和futures配合使用。

🔗堆上分配的值

首先，我们注意到，[堆上分配](https://os.phil-opp.com/heap-allocation/)的值大部分时间内都有一个固定的内存地址。这些值是通过分配调用创建的，随后通过如Box<T>这样的指针类型来引用。尽管移动指针类型本身是可能的，但指针所指向的堆内存值会保持在相同的内存地址上，直到通过释放调用来释放它为止。

利用堆分配，我们尝试去创建一个自引用结构体：

```rust
fn main() {
    let mut heap_value = Box::new(SelfReferential {
        self_ptr: 0 as *const _,
    });
    let ptr = &*heap_value as *const SelfReferential;
    heap_value.self_ptr = ptr;
    println!("heap value at: {:p}", heap_value);
    println!("internal reference: {:p}", heap_value.self_ptr);
}

struct SelfReferential {
    self_ptr: *const Self,
}
```
*[（可在playground上尝试）](https://play.rust-lang.org/?version=stable&mode=debug&edition=2018&gist=ce1aff3a37fcc1c8188eeaf0f39c97e8)*

我们创建了一个名为SelfReferential的简单结构体，其中包含了一个指针字段。我们首先用一个空指针初始化这个结构体，然后通过Box::new在堆上为其分配空间。接着，我们确定了这个堆分配结构体的内存地址，并将其存储在一个名为ptr的变量中。最后，我们通过将ptr变量赋值给self_ptr字段，使得这个结构体成为了自引用的。

当我们在[playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2018&gist=ce1aff3a37fcc1c8188eeaf0f39c97e8)上运行这段代码时，我们发现堆值的地址和其内部指针是相同的，这意味着self_ptr字段有效地成为了一个自引用。由于heap_value变量仅是一个指针，移动它（比如，通过将它作为函数参数传递）不会改变结构体本身的地址，因此即便指针被移动了，self_ptr依然是有效的。

然而，这个示例仍有破绽所在：我们可以移出Box<T>中的值或替换其内容：

```rust
let stack_value = mem::replace(&mut *heap_value, SelfReferential {
    self_ptr: 0 as *const _,
});
println!("value at: {:p}", &stack_value);
println!("internal reference: {:p}", stack_value.self_ptr);
```
*[（可在playground上尝试）](https://play.rust-lang.org/?version=stable&mode=debug&edition=2018&gist=e160ee8a64cba4cebc1c0473dcecb7c8)*

在这里，我们使用了mem::replace函数来将堆上分配的值替换为一个新的结构体实例。这允许我们把原先的heap_value移动到栈上，而结构体中的self_ptr字段现在则成了一个指向旧堆地址的悬空指针。当你在playground上运行这个示例时，你会发现打印出的“value at:”和“internal reference:”确实展示了不同的指针。因此，仅仅通过堆分配一个值并不足以确保自引用的安全。

导致上述问题的根本原因在于Box<T>允许我们获取堆上分配值的&mut T引用。这种&mut引用使得使用像mem::replace或mem::swap这样的方法来使堆上的值无效变成了可能。为了解决这个问题，我们必须避免创建指向自引用结构体的&mut引用。

🔗Pin<Box<T>>与Unpin  
pinning API通过[Pin](https://doc.rust-lang.org/stable/core/pin/struct.Pin.html)包装类型和[Unpin](https://doc.rust-lang.org/nightly/std/marker/trait.Unpin.html)标记型Trait提供了一个解决&mut T问题的方案。这些类型背后的思路是，把所有能够用来获取到被包装值的&mut引用的Pin方法（比如[get_mut](https://doc.rust-lang.org/nightly/core/pin/struct.Pin.html#method.get_mut)或[deref_mut](https://doc.rust-lang.org/nightly/core/pin/struct.Pin.html#method.deref_mut)）都依赖于Unpin Trait。Unpin Trait是一个[自动实现](https://doc.rust-lang.org/reference/special-types-and-traits.html#auto-traits)的 Trait，它默认为所有类型自动实现，除了那些明确选择不实现的类型。通过让自引用结构体选择不实现Unpin，我们就没有（安全的）方式能从Pin<Box<T>>类型中获取&mut T了。结果，它们内部的自引用就能保证保持有效。

作为示例, 更新我们之前提到的SelfReferential类型为例，来展示如何选择不实现Unpin：

```rust
use core::marker::PhantomPinned;

struct SelfReferential {
    self_ptr: *const Self,
    _pin: PhantomPinned,
}
```

我们通过增加第二个字段_pin，其类型为[PhantomPinned](https://doc.rust-lang.org/nightly/core/marker/struct.PhantomPinned.html)来选择不实现Unpin Trait。这个类型是一个零大小的标记类型，唯一的目的是不实现Unpin Trait。因为[auto trait](https://doc.rust-lang.org/reference/special-types-and-traits.html#auto-traits)的机制，单一的非Unpin字段就足够让整个结构体选择不实现Unpin。

第二步是将示例中的Box\<SelfReferential\>类型更改为Pin\<Box\>类型。这样做最简单的方式是使用Box::pin函数，而不是Box::new来创建堆上的值：

```rust
let mut heap_value = Box::pin(SelfReferential {
    self_ptr: 0 as *const _,
    _pin: PhantomPinned,
});
```

除了将Box::new替换为Box::pin外，我们还需要在结构体的初始化器中添加新的_pin字段。由于PhantomPinned是一个零大小类型，我们初始化它时只需提及其类型名称。

当我们尝试运行我们调整后的示例时，我们发现它不再能够正常运行：

```bash
error[E0594]: cannot assign to data in a dereference of `std::pin::Pin<std::boxed::Box<SelfReferential>>`
  --> src/main.rs:10:5
   |
10 |     heap_value.self_ptr = ptr;
   |     ^^^^^^^^^^^^^^^^^^^^^^^^^ cannot assign
   |
   = help: trait `DerefMut` is required to modify through a dereference, but it is not implemented for `std::pin::Pin<std::boxed::Box<SelfReferential>>`

error[E0596]: cannot borrow data in a dereference of `std::pin::Pin<std::boxed::Box<SelfReferential>>` as mutable
  --> src/main.rs:16:36
   |
16 |     let stack_value = mem::replace(&mut *heap_value, SelfReferential {
   |                                    ^^^^^^^^^^^^^^^^ cannot borrow as mutable
   |
   = help: trait `DerefMut` is required to modify through a dereference, but it is not implemented for `std::pin::Pin<std::boxed::Box<SelfReferential>>`
```

两个错误发生的原因都是因为Pin<Box<SelfReferential>>类型不再实现DerefMut Trait。这正符合我们的预期，因为DerefMut Trait会返回一个&mut引用，这正是我们希望避免的。这种情况仅发生于我们同时选择不实现Unpin并将Box::new替换为Box::pin的情况下。

现在的问题是，编译器不仅仅是阻止在第16行代码中移动类型，还禁止了在第10行初始化self_ptr字段。这是因为编译器不能区分&mut引用的合法与非法使用。为了让初始化操作再次成为可能，我们不得不使用不安全的[get_unchecked_mut](https://doc.rust-lang.org/nightly/core/pin/struct.Pin.html#method.get_unchecked_mut)方法：

```rust
// safe because modifying a field doesn't move the whole struct
unsafe {
    let mut_ref = Pin::as_mut(&mut heap_value);
    Pin::get_unchecked_mut(mut_ref).self_ptr = ptr;
}
```
*[（可在playground上尝试）](https://play.rust-lang.org/?version=stable&mode=debug&edition=2018&gist=b9ebbb11429d9d79b3f9fffe819e2018)

[get_unchecked_mut](https://doc.rust-lang.org/nightly/core/pin/struct.Pin.html#method.get_unchecked_mut)函数应用于Pin<&mut T>而不是Pin<Box<T>>，因此我们需要使用Pin::as_mut来转换这个值。接着，我们可以利用get_unchecked_mut返回的&mut引用来设置self_ptr字段。

此时，唯一剩余的错误是对mem::replace的预期错误。要注意的是，这个操作试图将堆上分配的值移动到栈上，这会打破存储在self_ptr字段中的自引用。通过选择不实现Unpin并利用Pin<Box<T>>，我们可以在编译时阻止这种操作，从而能够安全地处理自引用结构体。正如我们所见，编译器尚未能证明自引用的创建过程是安全的（至少现在是这样），因此我们需要使用unsafe代码块，并且需要自己验证其正确性。

🔗栈上的Pinning和Pin<&mut T>

在上一部分，我们了解了如何使用Pin<Box<T>>安全地创建堆上分配的自引用值。虽然这种方法有效且相对安全（除了需要用到unsafe代码块的构造外），但所需的堆分配会带来性能成本。由于Rust努力在可能的情况下提供零成本抽象，pinning API也支持创建指向栈上分配值的Pin<&mut T>实例。

与拥有被包装值所有权的Pin<Box<T>>实例不同，Pin<&mut T>实例仅是临时借用被包装的值。这让情况变得更加复杂，因为它要求程序员自行确保额外的约束。最关键的是，Pin<&mut T>必须在引用的T的整个生命周期内保持固定，这对于栈上的变量来说验证起来可能比较困难。为了辅助处理这一问题，存在像[pin-utils](https://docs.rs/pin-utils/0.1.0-alpha.4/pin_utils/)这样的库，但我仍然不推荐在不完全清楚自己在做什么的情况下在栈上使用固定。

想要进一步了解，可以查阅[pin模块](https://doc.rust-lang.org/nightly/core/pin/index.html)的文档以及[Pin::new_unchecked](https://doc.rust-lang.org/nightly/core/pin/struct.Pin.html#method.new_unchecked)方法。

🔗Pinning and Futures

如本文所述，[Future::poll](https://doc.rust-lang.org/nightly/core/future/trait.Future.html#tymethod.poll)方法采用了固定技术，具体是通过一个Pin<&mut Self>参数实现的：

```rust
fn poll(self: Pin<&mut Self>, cx: &mut Context) -> Poll<Self::Output>
```

之所以使用self: Pin<&mut Self>而不是常规的&mut self，是因为我们看到，通过async/await生成的future实例通常含有自引用。通过将自身封装进Pin，并让编译器对由async/await生成的自引用futures选择不实现Unpin，我们确保了在poll调用之间futures不会在内存中移动。这保障了所有内部引用的有效性。

需要指出的是，在第一次进行poll调用之前移动futures是没有问题的。这是因为futures在被首次poll之前是懒加载的，不会执行任何操作。因此，生成的状态机的初始状态因此只包含函数参数，而不包含任何内部引用。为了进行poll调用，调用方必须先将future封装进Pin，这样就确保了future不能再在内存中被移动。由于在栈上使用固定技术更加复杂，我建议总是结合使用[Box::pin](https://doc.rust-lang.org/nightly/alloc/boxed/struct.Box.html#method.pin)和[Pin::as_mut](https://doc.rust-lang.org/nightly/core/pin/struct.Pin.html#method.as_mut)。

如果你对如何安全实现一个利用栈固定的future组合器函数感兴趣，可以参考futures库中map组合器方法的[源代码](https://docs.rs/futures-util/0.3.4/src/futures_util/future/future/map.rs.html)，以及pin文档中关于[投影和结构性固定](https://doc.rust-lang.org/stable/std/pin/index.html#projections-and-structural-pinning)的部分。

### Executors and Wakers
利用async/await，我们可以以一种完全异步的方式灵活地操作futures。然而，如之前所学，futures在被轮询前不会执行任何操作。这意味着我们必须在某一时刻调用poll方法，否则异步代码将永远不会执行。

对于单个future，我们可以通过[如上所述](https://os.phil-opp.com/async-await/#waiting-on-futures)的循环来手动等待每个future完成。然而，这种方法效率低下，对于需要创建大量futures的程序来说并不实用。解决这一问题的最常用方法是定义一个全局的执行器，该执行器负责轮询系统中所有的futures直到它们完成。

🔗执行器  
执行器的目的是允许我们将futures作为独立的任务发起，通常通过某种spawn方法实现。接着，执行器负责轮询所有的futures直到它们完成。将所有futures集中管理的一个巨大优势是，当任一future返回Poll::Pending时，执行器可以切换到其他future。这样，异步操作能够并行执行，充分利用CPU资源。

许多执行器实现还可以充分利用多核CPU系统。它们创建一个线程池，如果有足够的工作量，能够使用所有核心，并通过如工作窃取等技术来在核心之间平衡负载。针对嵌入式系统也有特殊的执行器实现，它们针对低延迟和内存开销进行了优化。

为了减少重复轮询futures所带来的开销，执行器通常会利用Rust futures支持的唤醒器API。

🔗唤醒器  
唤醒器API的核心思想是，在每次调用poll时传递一个特殊的[Waker](https://doc.rust-lang.org/nightly/core/task/struct.Waker.html)类型，它被封装在[Context](https://doc.rust-lang.org/nightly/core/task/struct.Context.html)类型中。这个Waker类型由执行器创建，并可被异步任务用于表示其已经（部分）完成。因此，对于先前返回Poll::Pending的future，执行器不需要重新调用poll，直到它被对应的唤醒器通知。

下面是一个简单的例子，最好地展示了这一点：

```rust
async fn write_file() {
    async_write_file("foo.txt", "Hello").await;
}
```

这个函数异步地写入字符串“Hello”到foo.txt文件中。由于硬盘写入需要一段时间，这个future的首次poll调用很可能返回Poll::Pending。然而，硬盘驱动会内部存储传递给poll调用的Waker，并用它在文件写入硬盘时通知执行器。这意味着，执行器不必在收到waker通知之前浪费时间去尝试再次poll这个future。

当我们在本文的实现部分创建我们自己支持waker的执行器时，我们将详细探讨Waker类型的工作机制。

### 合作式多任务处理?
在本文的开头，我们讨论了抢占式与合作式多任务处理。抢占式多任务处理依赖于操作系统来强制切换正在运行的任务，而合作式多任务处理则要求任务通过定期进行yield操作来自愿放弃CPU控制权。合作式方法的一个主要优势是，任务可以自行保存其状态，这导致上下文切换更加高效，并允许任务之间共享同一个调用堆栈。

虽然这可能不是显而易见的，但futures和async/await实际上是合作式多任务处理模式的一种实现：

* 每一个被添加到执行器中的future本质上是一个合作式任务。
* futures通过返回Poll::Pending（或在完成时返回Poll::Ready）放弃CPU核心的控制权，而不是使用显式的yield操作。
    - 没有任何机制强制futures放弃CPU。如果它们选择，它们可以通过在循环中无限旋转等方式永远不从poll返回。
    - 由于每个future都有可能阻塞执行器中其他futures的执行，因此我们需要信任它们不会执行恶意操作。
* futures内部存储了所有必要的状态，以便在下一次poll调用时继续执行。通过async/await，编译器自动识别所有必要的变量并将它们储存在生成的状态机中。
    - 仅保存继续执行所需的最少状态。
    - 由于poll方法在返回时释放了调用堆栈，因此可以使用同一个堆栈来poll其他futures。
我们看到，futures和async/await完美符合合作式多任务处理模式；它们仅仅使用了不同的术语。因此，在接下来的内容中，我们将“任务”和“future”这两个术语互换使用。

## 实现

现在我们已经理解了Rust中基于futures和async/await的协作式多任务是如何工作的，现在是时候在我们的内核中加入对它的支持了。由于[Future](https://doc.rust-lang.org/nightly/core/future/trait.Future.html) Trait是core库的一部分，而async/await是语言本身的特性，我们在#![no_std]内核中使用它们没有什么特别的要求。唯一的条件是我们需要使用至少为2020-03-25的Rust nightly版本，因为在那之前async/await不兼容no_std环境。

使用足够新的夜间版本后，我们可以开始在main.rs中利用async/await：

```rust
// in src/main.rs

async fn async_number() -> u32 {
    42
}

async fn example_task() {
    let number = async_number().await;
    println!("async number: {}", number);
}
```

async_number函数是一个异步函数，因此编译器将它转换成了一个实现Future Trait的状态机。因为该函数仅返回42，所以得到的future在第一次调用poll时会直接返回Poll::Ready(42)。与async_number相同，example_task函数也是一个异步函数。它等待async_number返回的数值，然后使用println宏将其打印出来。

为了执行example_task返回的future，我们需要持续调用它的poll方法，直到它通过返回Poll::Ready表示已完成。要做到这一点，我们需要创建一个简单的执行器类型。

### Task

在开始实现执行器之前，我们先创建了一个新的task模块，并在其中定义了一个Task类型：

```rust
// in src/lib.rs

pub mod task;
```

```rust
// in src/task/mod.rs

use core::{future::Future, pin::Pin};
use alloc::boxed::Box;

pub struct Task {
    future: Pin<Box<dyn Future<Output = ()>>>,
}
```

Task结构是围绕一个固定在内存中、堆分配的、动态派发的future的新类型封装，其输出为()类型。这个设计有几个关键点需要注意：

* 我们规定与任务相关联的future必须返回()类型。这意味着任务本身不会返回任何结果，它们仅仅是为了执行某些副作用而执行。比如，我们之前定义的example_task函数就没有返回值，但是它会作为副作用在屏幕上打印信息。

* 通过使用dyn关键词，我们说明我们在Box中存储了一个trait对象。这表示future上的方法是通过动态派发执行的，这样就允许Task类型存储不同种类的futures。这一点非常重要，因为每个async函数都有其独特的类型，而我们希望能够创建多个不同的任务。

* 正如我们在讨论固定技术的部分所学到的，通过使用Pin<Box>类型，我们可以确保一个值不会在内存中被移动。这是通过将其放置在堆上并阻止创建指向它的&mut引用来实现的。这一点非常关键，因为通过async/await生成的futures可能包含指向自己的指针，这些指针在future被移动时会变得无效。

为了允许从任意future创建新的Task结构体，我们实现了一个new函数：

```rust
// in src/task/mod.rs

impl Task {
    pub fn new(future: impl Future<Output = ()> + 'static) -> Task {
        Task {
            future: Box::pin(future),
        }
    }
}
```

这个函数接受一个输出类型为()的任意future，并通过[Box::pin](https://doc.rust-lang.org/nightly/alloc/boxed/struct.Box.html#method.pin)函数在内存中对它进行固定。然后，它将这个装箱的future封装在Task结构体内并返回。这里需要'static生命周期标记，因为返回的Task可能会存活很长时间，因此future也需要在整个期间内有效。

我们还添加了一个poll方法，使执行器能够轮询存储的future：

```rust
// in src/task/mod.rs

use core::task::{Context, Poll};

impl Task {
    fn poll(&mut self, context: &mut Context) -> Poll<()> {
        self.future.as_mut().poll(context)
    }
}
```

因为Future Trait的[poll](https://doc.rust-lang.org/nightly/core/future/trait.Future.html#tymethod.poll)方法期望在Pin<&mut T>类型上调用，我们首先使用[Pin::as_mut](https://doc.rust-lang.org/nightly/core/pin/struct.Pin.html#method.as_mut)方法来转换self.future字段，这个字段的类型是Pin<Box<T>>。然后，我们在转换后的self.future字段上调用poll方法并返回其结果。由于Task::poll方法应仅由我们接下来要创建的执行器调用，我们将这个方法设为task模块的私有方法。

### 简单的执行器

因为执行器的设计可能非常复杂，我们决定首先创建一个非常基础的执行器，稍后再开发具有更多功能的执行器。为了做到这一点，我们首先创建了一个新的task::simple_executor子模块：

```rust
// in src/task/mod.rs

pub mod simple_executor;
```

```rust
// in src/task/simple_executor.rs

use super::Task;
use alloc::collections::VecDeque;

pub struct SimpleExecutor {
    task_queue: VecDeque<Task>,
}

impl SimpleExecutor {
    pub fn new() -> SimpleExecutor {
        SimpleExecutor {
            task_queue: VecDeque::new(),
        }
    }

    pub fn spawn(&mut self, task: Task) {
        self.task_queue.push_back(task)
    }
}
```

这个结构体有一个单一的字段task_queue，其类型为[VecDeque](https://doc.rust-lang.org/stable/alloc/collections/vec_deque/struct.VecDeque.html)，这实质上是一个向量，允许在两端进行推入(push)和弹出(pop)操作。使用这个类型的理念是，我们通过spawn方法在队列末尾添加新任务，然后从队列前端弹出下一个要执行的任务。这样，我们就实现了一个简单的[先入先出](https://en.wikipedia.org/wiki/FIFO_(computing_and_electronics))（FIFO）队列。

🔗虚拟唤醒器  
为了能够调用poll方法，我们需要创建一个[Context](https://doc.rust-lang.org/nightly/core/task/struct.Context.html)类型，它封装了一个[Waker](https://doc.rust-lang.org/stable/core/task/struct.RawWaker.html)类型。为了从简单做起，我们首先创建一个无实际功能的虚拟唤醒器。为此，我们创建一个RawWaker实例，它定义了不同Waker方法的实现，然后使用[Waker::from_raw](https://doc.rust-lang.org/stable/core/task/struct.Waker.html#method.from_raw)函数将其转化为一个Waker：

```rust
// in src/task/simple_executor.rs

use core::task::{Waker, RawWaker};

fn dummy_raw_waker() -> RawWaker {
    todo!();
}

fn dummy_waker() -> Waker {
    unsafe { Waker::from_raw(dummy_raw_waker()) }
}
```

使用from_raw函数是不安全的，因为如果程序员不遵循RawWaker的文档要求，可能会导致未定义的行为。在我们探究dummy_raw_waker函数的实现之前，我们首先尝试理解RawWaker类型是如何工作的。

🔗RawWaker  
[RawWaker](https://doc.rust-lang.org/stable/core/task/struct.RawWaker.html)类型需要程序员显式地定义一个[虚拟方法表](https://en.wikipedia.org/wiki/Virtual_method_table)（vtable），这个表格详细指定了在RawWaker被克隆、唤醒或被释放时应该调用哪些函数。这个vtable的布局是通过[RawWakerVTable](https://doc.rust-lang.org/stable/core/task/struct.RawWakerVTable.html)类型定义的。每个函数都会接收一个\*const ()参数，它是一个抹去类型信息的指针，指向某个具体的值。之所以使用\*const ()指针而不是具体的引用，是因为RawWaker类型设计为非泛型，但仍需支持任意类型。这个指针是通过将其作为[RawWaker::new](https://doc.rust-lang.org/stable/core/task/struct.RawWaker.html#method.new)的data参数来提供的，仅用于初始化一个RawWaker。然后，Waker使用这个RawWaker来调用vtable函数，并将data作为参数传递。

一般情况下，RawWaker是为某个封装在[Box](https://doc.rust-lang.org/stable/alloc/boxed/struct.Box.html)或[Arc](https://doc.rust-lang.org/stable/alloc/sync/struct.Arc.html)类型中的堆分配的结构体创建的。对于这类类型，可以使用[Box::into_raw](https://doc.rust-lang.org/stable/alloc/boxed/struct.Box.html#method.into_raw)等方法，将Box<T>转换成\*const T指针。这个指针随后可以被转换成一个匿名的\*const ()指针，并传递给RawWaker::new。由于每个vtable函数都接收相同的\*const ()作为参数，这些函数可以安全地将指针转回Box<T>或&T进行操作。如你所想，这个过程极其危险，很容易因为错误而导致未定义的行为。因此，除非必要，否则不建议手动创建RawWaker。

🔗虚拟RawWaker  
虽然不推荐手动创建RawWaker，但目前创建一个无操作的虚拟Waker的唯一方法就是这样。幸运的是，我们的目标就是不执行任何操作，这让实现dummy_raw_waker函数变得相对安全一些：

```rust
// in src/task/simple_executor.rs

use core::task::RawWakerVTable;

fn dummy_raw_waker() -> RawWaker {
    fn no_op(_: *const ()) {}
    fn clone(_: *const ()) -> RawWaker {
        dummy_raw_waker()
    }

    let vtable = &RawWakerVTable::new(clone, no_op, no_op, no_op);
    RawWaker::new(0 as *const (), vtable)
}
```

我们首先定义了两个内部函数，分别是no_op和clone。no_op函数接收一个*const ()指针并不进行任何操作。clone函数也接收一个*const ()指针，然后通过重新调用dummy_raw_waker来返回一个新的RawWaker。我们利用这两个函数创建了一个最简单的RawWakerVTable：用clone函数处理克隆操作，而no_op函数用于处理所有其他操作。因为RawWaker不执行任何实际操作，所以使用clone返回一个新的RawWaker而不是真正克隆它并不会造成问题。

创建好vtable后，我们通过[RawWaker::new](https://doc.rust-lang.org/stable/core/task/struct.RawWaker.html#method.new)函数来创建一个RawWaker。传递的*const ()参数并不重要，因为vtable中的函数不会用到它。因此，我们简单地传递了一个空指针。

🔗实现run方法  
既然我们有了创建Waker实例的方式，我们就可以在我们的执行器上实现一个run方法了。最简单的run方法是反复循环轮询队列中的所有任务，直到所有任务都执行完毕。这种方法并不高效，因为它没有利用Waker类型的通知机制，但这是让事情开始运转的一种简单方式：

```rust
// in src/task/simple_executor.rs

use core::task::{Context, Poll};

impl SimpleExecutor {
    pub fn run(&mut self) {
        while let Some(mut task) = self.task_queue.pop_front() {
            let waker = dummy_waker();
            let mut context = Context::from_waker(&waker);
            match task.poll(&mut context) {
                Poll::Ready(()) => {} // task done
                Poll::Pending => self.task_queue.push_back(task),
            }
        }
    }
}
```

该函数利用一个while let循环处理task_queue中的所有任务。对于每个任务，它首先通过我们的dummy_waker函数返回的Waker实例创建一个Context类型。接着，使用这个上下文调用Task::poll方法。如果poll方法返回Poll::Ready，意味着任务已完成，我们便进入下一个任务。如果任务返回Poll::Pending，则将其再次加入队列末尾，以便在后续循环中再次轮询。

🔗尝试运行  
有了我们的SimpleExecutor，我们现在可以尝试在main.rs中运行example_task函数返回的任务：

```rust
// in src/main.rs

use blog_os::task::{Task, simple_executor::SimpleExecutor};

fn kernel_main(boot_info: &'static BootInfo) -> ! {
    // […] initialization routines, including `init_heap`

    let mut executor = SimpleExecutor::new();
    executor.spawn(Task::new(example_task()));
    executor.run();

    // […] test_main, "it did not crash" message, hlt_loop
}


// Below is the example_task function again so that you don't have to scroll up

async fn async_number() -> u32 {
    42
}

async fn example_task() {
    let number = async_number().await;
    println!("async number: {}", number);
}
```

运行时，我们会看到期待的“async number: 42”消息被打印到屏幕上：

![](https://os.phil-opp.com/async-await/qemu-simple-executor.png)

让我们回顾一下这个示例中发生的各个步骤：

* 首先，我们创建了一个新的SimpleExecutor实例，其task_queue为空。

* 接下来，我们调用异步函数example_task，它返回一个future。我们将这个future封装进Task类型，这会将其移到堆上并固定，然后通过spawn方法将这个任务添加到执行器的task_queue中。

* 然后我们调用run方法，开始执行队列中的任务。这个过程包括：
    - 从task_queue中取出任务。
    - 为任务创建一个RawWaker，将其转换为Waker实例，然后基于此创建一个Context实例。
    - 利用刚创建的Context，调用任务future的poll方法。
    - 因为example_task不需要等待任何东西，它可以在第一次调用poll时直接运行至完成。这就是打印“async number: 42”消息的时刻。
    - 由于example_task直接返回Poll::Ready，它不会被再次加入任务队列。

* 当task_queue为空时，run方法结束，我们的kernel_main函数继续执行，并打印出“It did not crash!”消息。

### 异步键盘输入
我们的简单执行器并未利用唤醒器通知，而是简单地循环处理所有任务直至完成。对我们当前的示例而言，这并不构成问题，因为我们的example_task能够在首次调用poll时直接执行完毕。要体验到使用正确实现的唤醒器所带来的性能优势，我们首先需要创建一个真正的异步任务，也就是说，在首次poll调用时很可能返回Poll::Pending的任务。

我们的系统中已经存在某种形式的异步性，我们可以利用这一点：硬件中断。如我们在中断相关文章中所学到的，硬件中断可以在由外部设备决定的任何时间点发生。例如，硬件定时器在预定时间过去后向CPU发送一个中断。当CPU接收到一个中断时，它会立即将控制权转移到在中断描述符表（IDT）中定义的相应处理函数。

接下来，我们将基于键盘中断创建一个异步任务。键盘中断是此类任务的一个很好的候选者，因为它既是非确定性的，也是对延迟非常敏感的。非确定性意味着下一次按键发生的时间无法预测，因为这完全取决于用户。对延迟敏感意味着我们希望及时处理键盘输入，否则用户可能会感觉到输入延迟。为了高效支持这样的任务，确保执行器能够正确地支持唤醒器通知非常关键。

🔗扫描码队列  
目前，我们直接在中断处理器中处理键盘输入。长期这样做并不理想，因为中断处理器应该尽可能保持简短，它们可能会打断重要工作。中断处理器应仅执行最必要的工作量（比如，读取键盘扫描码），并将剩余工作（比如，解释扫描码）留给后台任务。

将工作委派给后台任务的一个常见模式是创建某种队列。中断处理器将工作单元推入队列，而后台任务则处理队列中的工作。将这一模式应用到我们的键盘中断上，意味着中断处理器仅负责从键盘读取扫描码，将其推入队列，然后返回。键盘任务位于队列的另一端，负责解释并处理推送到队列中的每一个扫描码：

![](https://os.phil-opp.com/async-await/scancode-queue.svg)

这个队列可以简单地通过一个受互斥锁保护的[VecDeque](https://doc.rust-lang.org/stable/alloc/collections/vec_deque/struct.VecDeque.html)来实现。但是，在中断处理程序中使用互斥锁不是一个好主意，因为这很容易导致死锁。比如说，当用户在键盘任务已经锁定队列的情况下按键时，中断处理程序会尝试再次获取锁，从而无限期地等待。此方法的另一个问题是，当VecDeque满了时，它会通过新的堆分配来自动扩容，这又可能导致死锁，因为我们的分配器内部也使用了互斥锁。此外，堆分配可能失败或在堆碎片化的情况下耗时较长。

为了避免这些问题，我们需要一种不需互斥锁或分配即可进行推送操作的队列实现。可以通过使用无锁的[原子操作](https://doc.rust-lang.org/core/sync/atomic/index.html)来推入和弹出元素来实现这样的队列。这样，创建的推入和弹出操作只需要&self引用，因此可以无需互斥锁即可使用。为了避免推送时的分配，队列可以背后支持一个预先分配的固定大小的缓冲区。虽然这会使队列有界（即，有最大长度），但实际上通常能定义出队列长度的合理上限，因此这不成问题。

🔗crossbeam Crate  
以正确且高效的方式实现这样一个队列非常具有挑战性，因此我建议使用现有的、经过良好测试的实现。[crossbeam](https://github.com/crossbeam-rs/crossbeam)是一个流行的Rust项目，它为并发编程实现了多种无需互斥锁的类型。它提供的ArrayQueue类型正是我们需要的。幸运的是，这个类型完全兼容带有分配支持的no_std crates。

要使用这个类型，我们需要在项目中添加对crossbeam-queue crate的依赖：

```toml
# in Cargo.toml

[dependencies.crossbeam-queue]
version = "0.2.1"
default-features = false
features = ["alloc"]
```

默认情况下，这个crate依赖于标准库。为了使其与no_std兼容，我们需要禁用其默认功能，并启用alloc功能。*（值得注意的是，我们也可以依赖于主crossbeam crate，它重新导出了crossbeam-queue crate，但这会导致依赖数量增加和编译时间延长。）*

🔗队列实现  

借助ArrayQueue类型，我们现在可以在一个新的task::keyboard模块中创建全局的扫描码队列：

```rust
// in src/task/mod.rs

pub mod keyboard;
```

```rust
// in src/task/keyboard.rs

use conquer_once::spin::OnceCell;
use crossbeam_queue::ArrayQueue;

static SCANCODE_QUEUE: OnceCell<ArrayQueue<u8>> = OnceCell::uninit();
```

因为[ArrayQueue::new](https://docs.rs/crossbeam/0.7.3/crossbeam/queue/struct.ArrayQueue.html#method.new)进行堆分配，而堆分配在编译时（目前）无法实现，所以我们无法直接初始化静态变量。作为替代，我们使用conquer_once crate的[OnceCell](https://docs.rs/conquer-once/0.2.0/conquer_once/raw/struct.OnceCell.html)类型，它允许我们安全地一次性初始化静态值。要引入这个crate，我们需要在Cargo.toml中添加它作为依赖项：

```toml
# in Cargo.toml

[dependencies.conquer-once]
version = "0.2.0"
default-features = false
```

尽管我们这里可以使用[lazy_static](https://docs.rs/lazy_static/1.4.0/lazy_static/index.html)宏，但OnceCell原语有个优势，即我们可以确保初始化不会在中断处理函数中进行，避免了中断处理函数执行堆分配。

🔗填充队列  
为了填充扫描码队列，我们创建了一个名为add_scancode的新函数，计划从中断处理程序中调用它：

```rust
// in src/task/keyboard.rs

use crate::println;

/// Called by the keyboard interrupt handler
///
/// Must not block or allocate.
pub(crate) fn add_scancode(scancode: u8) {
    if let Ok(queue) = SCANCODE_QUEUE.try_get() {
        if let Err(_) = queue.push(scancode) {
            println!("WARNING: scancode queue full; dropping keyboard input");
        }
    } else {
        println!("WARNING: scancode queue uninitialized");
    }
}
```

我们使用[OnceCell::try_get](https://docs.rs/conquer-once/0.2.0/conquer_once/raw/struct.OnceCell.html#method.try_get)来获取已初始化队列的引用。如果队列尚未初始化，我们就忽略键盘扫描码并打印警告。我们不在此函数中尝试初始化队列是因为它将被中断处理程序调用，而中断处理程序不应执行堆分配。由于这个函数不应从我们的main.rs中调用，我们使用pub(crate)可见性，使其仅在我们的lib.rs中可用。

[ArrayQueue::push](https://docs.rs/crossbeam/0.7.3/crossbeam/queue/struct.ArrayQueue.html#method.push)方法只需要一个&self引用，这让在静态队列上调用此方法变得非常简单。ArrayQueue类型自行处理所有必要的同步操作，因此我们这里不需要互斥锁包装。如果队列已满，我们也会打印警告。

为了在键盘中断时调用add_scancode函数，我们在interrupts模块中更新了keyboard_interrupt_handler函数：

```rust
// in src/interrupts.rs

extern "x86-interrupt" fn keyboard_interrupt_handler(
    _stack_frame: InterruptStackFrame
) {
    use x86_64::instructions::port::Port;

    let mut port = Port::new(0x60);
    let scancode: u8 = unsafe { port.read() };
    crate::task::keyboard::add_scancode(scancode); // new

    unsafe {
        PICS.lock()
            .notify_end_of_interrupt(InterruptIndex::Keyboard.as_u8());
    }
}
```

我们移除了该函数中所有键盘处理代码，仅添加了对add_scancode函数的调用。该函数的其余部分保持不变。

如预期，当我们现在使用cargo run运行我们的项目时，按键事件不再直接打印到屏幕上。相反，每次按键我们都会看到关于扫描码队列未初始化的警告。

🔗扫描码流  
为了初始化SCANCODE_QUEUE并以异步方式从队列中读取扫描码，我们创建了一个新的ScancodeStream类型：

_private字段的目的是为了防止从模块外部构造此结构体。这让new函数成为构建这个类型的唯一方法。在该函数中，我们首先尝试初始化SCANCODE_QUEUE静态变量。如果它已经被初始化，我们将触发panic，以确保只创建一个ScancodeStream实例。

为了将扫描码提供给异步任务，下一步是实现一个类似poll的方法，该方法尝试从队列中取出下一个扫描码。尽管这听起来像我们应该为我们的类型实现Future Trait，但这里并不完全适合。问题是，Future Trait仅抽象了一个单一的异步值，并期望在其返回Poll::Ready后不再调用poll方法。然而，我们的扫描码队列包含多个异步值，因此继续对其进行轮询是合理的。

🔗Stream Trait   
鉴于产生多个异步值的类型很常见，[futures](https://docs.rs/futures/0.3.4/futures/)库为这类类型提供了一个有用的抽象：Stream Trait。该 Trait的定义如下：

```rust
pub trait Stream {
    type Item;

    fn poll_next(self: Pin<&mut Self>, cx: &mut Context)
        -> Poll<Option<Self::Item>>;
}
```

这个定义与Future Trait非常相似，但有以下不同：

    - 关联类型被命名为Item，而不是Output。
    - 与返回Poll<Self::Item>的poll方法不同，Stream Trait定义了一个返回Poll<Option<Self::Item>>的poll_next方法（注意附加的Option）。

还有一个语义上的差异：poll_next可以被反复调用，直到它返回Poll::Ready(None)以表示流结束。从这个角度来看，该方法类似于Iterator::next方法，后者在遍历完所有值后也会返回None。

🔗实现 Stream  
我们来为我们的ScancodeStream实现Stream Trait，以便以异步方式提供SCANCODE_QUEUE中的值。首先，我们需要添加对futures-util crate的依赖，这个crate包含了Stream类型：

```toml
# in Cargo.toml

[dependencies.futures-util]
version = "0.3.4"
default-features = false
features = ["alloc"]
```

为了使crate与no_std兼容，我们禁用了默认特性，并启用了alloc特性以访问基于分配的类型（我们稍后会需要到这个）。*（需要注意的是，我们也可以依赖主futures crate，它会重新导出futures-util crate，但这会导致依赖数量增加以及编译时间延长。）*

现在，我们可以导入并实现Stream Trait：

```rust
// in src/task/keyboard.rs

use core::{pin::Pin, task::{Poll, Context}};
use futures_util::stream::Stream;

impl Stream for ScancodeStream {
    type Item = u8;

    fn poll_next(self: Pin<&mut Self>, cx: &mut Context) -> Poll<Option<u8>> {
        let queue = SCANCODE_QUEUE.try_get().expect("not initialized");
        match queue.pop() {
            Ok(scancode) => Poll::Ready(Some(scancode)),
            Err(crossbeam_queue::PopError) => Poll::Pending,
        }
    }
}
```

我们首先使用[OnceCell::try_get](https://docs.rs/conquer-once/0.2.0/conquer_once/raw/struct.OnceCell.html#method.try_get)方法来获取对已初始化的扫描码队列的引用。这个操作应该永远不会失败，因为我们在new函数中初始化了队列，因此我们可以安全地使用expect方法在未初始化的情况下触发panic。接着，我们使用ArrayQueue::pop方法尝试从队列中获取下一个元素。如果成功，我们返回包含扫描码的Poll::Ready(Some(…))。如果失败，这意味着队列为空。这种情况下，我们返回Poll::Pending。

🔗唤醒器支持  
像Futures::poll方法一样，Stream::poll_next方法要求在返回Poll::Pending之后变得准备就绪时通知执行器。这样，执行器不需要在被通知之前再次轮询同一个任务，这大大减少了等待任务的性能开销。

为了发送这种通知，任务应该从传递的Context引用中提取[Waker](https://doc.rust-lang.org/nightly/core/task/struct.Waker.html)并将其存储在某处。当任务准备就绪时，它应该调用存储的Waker的[wake](https://doc.rust-lang.org/nightly/core/task/struct.Waker.html)方法，以通知执行器应该再次轮询该任务。

🔗AtomicWaker
为了为我们的ScancodeStream实现Waker通知，我们需要一个地方来在poll调用之间存储Waker。我们不能将它作为ScancodeStream本身的一个字段，因为它需要能够从add_scancode函数访问。解决方案是使用futures-util crate提供的AtomicWaker类型的静态变量。像ArrayQueue类型一样，这种类型基于原子指令，并且可以安全地存储在静态变量中并并发修改。

让我们使用AtomicWaker类型来定义一个静态的WAKER变量：

```rust
// in src/task/keyboard.rs

use futures_util::task::AtomicWaker;

static WAKER: AtomicWaker = AtomicWaker::new();
```

该设计思想是，poll_next实现会将当前的唤醒器存储在这个静态变量中，并且当队列中添加了新的扫描码时，add_scancode函数会在其上调用wake函数。

🔗存储唤醒器  
根据poll/poll_next定义的规范，当任务返回Poll::Pending时，需要为传递的唤醒器注册一个唤醒操作。我们将调整我们的poll_next实现以满足此要求：

```rust
// in src/task/keyboard.rs

impl Stream for ScancodeStream {
    type Item = u8;

    fn poll_next(self: Pin<&mut Self>, cx: &mut Context) -> Poll<Option<u8>> {
        let queue = SCANCODE_QUEUE
            .try_get()
            .expect("scancode queue not initialized");

        // fast path
        if let Ok(scancode) = queue.pop() {
            return Poll::Ready(Some(scancode));
        }

        WAKER.register(&cx.waker());
        match queue.pop() {
            Ok(scancode) => {
                WAKER.take();
                Poll::Ready(Some(scancode))
            }
            Err(crossbeam_queue::PopError) => Poll::Pending,
        }
    }
}
```

如先前所做，我们首先使用[OnceCell::try_get](https://docs.rs/conquer-once/0.2.0/conquer_once/raw/struct.OnceCell.html#method.try_get)函数来获取对初始化的扫描码队列的引用。然后，我们乐观地尝试从队列中弹出元素，并在成功时返回Poll::Ready。这样，我们可以避免在队列非空时注册唤醒器的性能损耗。

如果queue.pop()的首次调用未成功，表明队列可能为空。但仅是可能，因为在检查之后中断处理器可能已异步地填充了队列。由于下一次检查可能再次发生此类竞争条件，我们需要在第二次检查前在WAKER静态变量中注册唤醒器。这样，即使我们在返回Poll::Pending之前可能收到唤醒，但对于检查后推送的任何扫描码，我们都保证会收到唤醒。

在通过AtomicWaker::register函数注册传入Context中包含的唤醒器后，我们再次尝试从队列中弹出元素。如果此时成功，我们返回Poll::Ready。我们也会使用AtomicWaker::take方法再次移除注册的唤醒器，因为不再需要唤醒通知了。如果queue.pop()第二次未成功，我们像之前一样返回Poll::Pending，但这次我们已经注册了唤醒操作。

值得注意的是，对于尚未返回Poll::Pending的任务，有两种可能唤醒的情况。一种是当唤醒操作在返回Poll::Pending之前立即发生的上述竞争情况。另一种是在注册唤醒器后队列不再为空，因此返回Poll::Ready。由于这些意外唤醒无法避免，执行器需要能够正确处理它们。

🔗唤醒存储的唤醒器  
为了唤醒存储的唤醒器，我们在add_scancode函数中增加了对WAKER.wake()的调用：

```rust
// in src/task/keyboard.rs

pub(crate) fn add_scancode(scancode: u8) {
    if let Ok(queue) = SCANCODE_QUEUE.try_get() {
        if let Err(_) = queue.push(scancode) {
            println!("WARNING: scancode queue full; dropping keyboard input");
        } else {
            WAKER.wake(); // new
        }
    } else {
        println!("WARNING: scancode queue uninitialized");
    }
}
```

我们所做的唯一更改是在成功推送到扫描码队列后调用WAKER.wake()。如果WAKER静态变量中注册了唤醒器，此方法将在其上调用同名的[wake](https://doc.rust-lang.org/stable/core/task/struct.Waker.html#method.wake)方法，以通知执行器。否则，此操作无效，即没有任何事情发生。

重要的是，我们只有在推送到队列之后才调用wake，因为否则可能会过早地唤醒任务，而此时队列仍为空。例如，当使用多线程执行器在不同CPU核心上并行启动被唤醒的任务时，就可能发生这种情况。虽然我们目前还不支持线程，但我们不久将会添加，并且我们不希望届时出现问题。

🔗键盘任务  
现在我们为ScancodeStream实现了Stream特征之后，我们可以用它来创建一个异步键盘任务：

```rust
// in src/task/keyboard.rs

use futures_util::stream::StreamExt;
use pc_keyboard::{layouts, DecodedKey, HandleControl, Keyboard, ScancodeSet1};
use crate::print;

pub async fn print_keypresses() {
    let mut scancodes = ScancodeStream::new();
    let mut keyboard = Keyboard::new(layouts::Us104Key, ScancodeSet1,
        HandleControl::Ignore);

    while let Some(scancode) = scancodes.next().await {
        if let Ok(Some(key_event)) = keyboard.add_byte(scancode) {
            if let Some(key) = keyboard.process_keyevent(key_event) {
                match key {
                    DecodedKey::Unicode(character) => print!("{}", character),
                    DecodedKey::RawKey(key) => print!("{:?}", key),
                }
            }
        }
    }
}
```

这段代码和我们之前在[键盘中断处理函数](https://os.phil-opp.com/hardware-interrupts/#interpreting-the-scancodes)中用到的非常相似，不同的是，现在我们不再直接从I/O端口读取扫描码，而是从ScancodeStream中取得它们。首先，我们创建一个新的ScancodeStream，然后不断利用[StreamExt](https://docs.rs/futures-util/0.3.4/futures_util/stream/trait.StreamExt.html) Trait提供的[next](https://docs.rs/futures-util/0.3.4/futures_util/stream/trait.StreamExt.html#method.next)方法来获取一个解析为流中下一个元素的Future。通过在该Future上使用await操作符，我们可以异步地等待其结果。

我们使用while let循环，直到流返回None表示结束。由于我们的poll_next方法永远不会返回None，实际上这形成了一个无限循环，因此print_keypresses任务永远不会完成。

让我们把print_keypresses任务添加到main.rs中的执行器里，以重新使键盘输入工作：

```rust
// in src/main.rs

use blog_os::task::keyboard; // new

fn kernel_main(boot_info: &'static BootInfo) -> ! {

    // […] initialization routines, including init_heap, test_main

    let mut executor = SimpleExecutor::new();
    executor.spawn(Task::new(example_task()));
    executor.spawn(Task::new(keyboard::print_keypresses())); // new
    executor.run();

    // […] "it did not crash" message, hlt_loop
}
```

再次执行cargo run, 将看到键盘输入重新可工作了:

![](https://os.phil-opp.com/async-await/qemu-keyboard-output.gif)

如果你注意观察你电脑的CPU利用率，你会发现QEMU进程现在持续使CPU保持忙碌状态。这是因为我们的SimpleExecutor在循环中不断地轮询任务。因此，即使我们没有按下任何键盘键，执行器也会重复地对print_keypresses任务调用poll方法，尽管该任务无法取得任何进展，并且每次都会返回Poll::Pending。

### 有唤醒器支持的执行器
为了解决性能问题，我们需要构建一个能够正确利用唤醒器通知的执行器。这样，当下一个键盘中断发生时，执行器就会得到通知，因此它就无需反复轮询print_keypresses任务了。

🔗任务ID  
要构建一个具有适当唤醒器通知支持的执行器的第一步是为每个任务分配一个唯一的ID。这是必须的，因为我们需要一种方法来指明应该唤醒哪个任务。我们首先创建一个新的TaskId封装类型：

```rust
// in src/task/mod.rs

#[derive(Debug, Clone, Copy, PartialEq, Eq, PartialOrd, Ord)]
struct TaskId(u64);
```

TaskId结构体是一个简单的围绕u64的封装类型。我们为它派生了几个特性以使其可打印、可复制、可比较及可排序。排序特别重要，因为我们接下来想把TaskId作为[BTreeMap](https://doc.rust-lang.org/alloc/collections/btree_map/struct.BTreeMap.html)的键类型使用。

为了生成一个新的唯一ID，我们创建了一个TaskId::new函数：

```rust
use core::sync::atomic::{AtomicU64, Ordering};

impl TaskId {
    fn new() -> Self {
        static NEXT_ID: AtomicU64 = AtomicU64::new(0);
        TaskId(NEXT_ID.fetch_add(1, Ordering::Relaxed))
    }
}
```

该函数利用类型为[AtomicU64](https://doc.rust-lang.org/core/sync/atomic/struct.AtomicU64.html)的静态NEXT_ID变量来确保每个ID只被分配一次。[fetch_add](https://doc.rust-lang.org/core/sync/atomic/struct.AtomicU64.html#method.fetch_add)方法原子地增加值并在一个原子操作中返回之前的值。这意味着，即使TaskId::new方法并行调用，每个ID也都会精确地只被返回一次。[Ordering](https://doc.rust-lang.org/core/sync/atomic/enum.Ordering.html)参数指定了编译器是否允许在指令流中重排fetch_add操作。由于我们的需求仅仅是ID的唯一性，因此在本例中最弱的Relaxed Ordering就已足够。

现在，我们可以通过添加一个额外的id字段来扩展我们的Task类型：

```rust
// in src/task/mod.rs

pub struct Task {
    id: TaskId, // new
    future: Pin<Box<dyn Future<Output = ()>>>,
}

impl Task {
    pub fn new(future: impl Future<Output = ()> + 'static) -> Task {
        Task {
            id: TaskId::new(), // new
            future: Box::pin(future),
        }
    }
}
```

新增的id字段使得可以唯一地标识一个任务，这是为了特定任务的唤醒所必需的。

🔗运行器类型  
我们在task::executor模块中创建了一个新的Executor类型：

```rust
// in src/task/mod.rs

pub mod executor;
```

```rust
// in src/task/executor.rs

use super::{Task, TaskId};
use alloc::{collections::BTreeMap, sync::Arc};
use core::task::Waker;
use crossbeam_queue::ArrayQueue;

pub struct Executor {
    tasks: BTreeMap<TaskId, Task>,
    task_queue: Arc<ArrayQueue<TaskId>>,
    waker_cache: BTreeMap<TaskId, Waker>,
}

impl Executor {
    pub fn new() -> Self {
        Executor {
            tasks: BTreeMap::new(),
            task_queue: Arc::new(ArrayQueue::new(100)),
            waker_cache: BTreeMap::new(),
        }
    }
}
```

与我们对SimpleExecutor所做的使用[VecDeque](https://doc.rust-lang.org/stable/alloc/collections/vec_deque/struct.VecDeque.html)来存储任务不同，我们使用一个task IDs的task_queue和一个名为tasks的[BTreeMap](https://doc.rust-lang.org/alloc/collections/btree_map/struct.BTreeMap.html)，后者包含了实际的Task实例。这个映射通过TaskId来索引，以便有效地继续特定任务的执行。

task_queue字段是一个任务ID的[ArrayQueue](https://docs.rs/crossbeam/0.7.3/crossbeam/queue/struct.ArrayQueue.html)，这个队列被封装在实现了引用计数的[Arc](https://doc.rust-lang.org/stable/alloc/sync/struct.Arc.html)类型中。引用计数允许值在多个所有者之间共享所有权。它通过在堆上分配值并计数活跃的引用来工作。当活跃引用数降至零时，该值不再需要并将被释放。

我们选择Arc<ArrayQueue>类型作为task_queue，因为它将在执行器和唤醒器之间共享。概念是唤醒器将被唤醒任务的ID推送到队列。执行器位于队列的接收端，从tasks映射中通过它们的ID检索被唤醒的任务，然后执行它们。选择使用固定大小的队列而非无界队列（例如[SegQueue](https://docs.rs/crossbeam-queue/0.2.1/crossbeam_queue/struct.SegQueue.html)）的原因是中断处理程序向此队列推送时不应进行内存分配。

除了task_queue和tasks映射外，Executor类型还有一个waker_cache字段，这同样是一个映射。这个映射在任务的Waker被创建后进行缓存。这样做有两个原因：首先，它通过重用同一个任务的多次唤醒的同一个唤醒器而不是每次创建新的唤醒器来提高性能。其次，它确保引用计数的唤醒器不会在中断处理函数中被释放，因为这可能导致死锁（稍后会有更多细节说明）。

为了创建Executor，我们提供了一个简单的new函数。我们为task_queue选择了100的容量，这对于可预见的未来应该足够。如果我们的系统在某个时刻有超过100个并发任务，我们可以轻松增加这个容量。

🔗分配任务  
就像对SimpleExecutor那样，我们为Executor类型提供了spawn方法，这个方法将指定的任务添加到tasks映射中，并通过将其ID推送到task_queue来立即唤醒它：

```rust
// in src/task/executor.rs

impl Executor {
    pub fn spawn(&mut self, task: Task) {
        let task_id = task.id;
        if self.tasks.insert(task.id, task).is_some() {
            panic!("task with same ID already in tasks");
        }
        self.task_queue.push(task_id).expect("queue full");
    }
}
```

如果映射中已存在相同ID的任务，[BTreeMap::insert]方法会返回它。这本不应发生，因为每个任务都有一个唯一的ID，因此如果发生这种情况，我们会触发panic，表示代码中存在错误。同样，如果task_queue满了，我们也会触发panic，因为如果我们选择了足够大的队列大小，这种情况本不应发生。

🔗执行任务  
为了执行task_queue中的所有任务，我们创建了一个名为run_ready_tasks的私有方法：

```rust
// in src/task/executor.rs

use core::task::{Context, Poll};

impl Executor {
    fn run_ready_tasks(&mut self) {
        // destructure `self` to avoid borrow checker errors
        let Self {
            tasks,
            task_queue,
            waker_cache,
        } = self;

        while let Ok(task_id) = task_queue.pop() {
            let task = match tasks.get_mut(&task_id) {
                Some(task) => task,
                None => continue, // task no longer exists
            };
            let waker = waker_cache
                .entry(task_id)
                .or_insert_with(|| TaskWaker::new(task_id, task_queue.clone()));
            let mut context = Context::from_waker(waker);
            match task.poll(&mut context) {
                Poll::Ready(()) => {
                    // task done -> remove it and its cached waker
                    tasks.remove(&task_id);
                    waker_cache.remove(&task_id);
                }
                Poll::Pending => {}
            }
        }
    }
}
```

这个函数的基本理念与我们的SimpleExecutor相似：遍历task_queue中的所有任务，为每个任务创建唤醒器，然后对它们进行轮询。但与将挂起的任务再次添加到task_queue末尾不同，我们让TaskWaker实现负责把被唤醒的任务重新加入队列。这个唤醒器类型的实现很快将会展示。

让我们深入了解run_ready_tasks方法的一些实现细节：

* 我们通过[解构](https://doc.rust-lang.org/book/ch18-03-pattern-syntax.html#destructuring-to-break-apart-values)将self拆分为其三个字段，以避免一些借用检查器的错误。具体来说，我们的实现需要在闭包中访问self.task_queue，但当前这样做会尝试完全借用self。这是一个基本的借用检查器问题，在[RFC 2229](https://github.com/rust-lang/rfcs/pull/2229)实施后将得到[解决](https://github.com/rust-lang/rust/issues/53488)。

* 对于每一个从队列中弹出的任务ID，我们从tasks映射中检索对应任务的可变引用。因为我们的ScancodeStream实现在检查是否需要将任务置为挂起状态之前就注册了唤醒器，可能会发生对一个已不存在的任务进行唤醒的情况。这种情况下，我们简单地忽略该唤醒并继续处理队列中的下一个ID。

* 为了避免每次轮询时创建唤醒器的性能损耗，我们利用waker_cache映射在唤醒器创建后为每个任务存储其唤醒器。为此，我们结合使用BTreeMap::entry方法和Entry::or_insert_with来创建一个新的唤醒器（如果尚不存在），然后获取对它的可变引用。为了创建新的唤醒器，我们克隆task_queue并将其连同任务ID一起传给TaskWaker::new函数（实现将在下文展示）。由于task_queue被封装在Arc中，克隆仅增加了对值的引用计数，但仍指向同一个堆分配的队列。注意，这样重用唤醒器的方式并不适用于所有唤醒器实现，但我们的TaskWaker类型允许这样做。

当任务返回Poll::Ready时，意味着任务已完成。这种情况下，我们使用[BTreeMap::remove](https://doc.rust-lang.org/alloc/collections/btree_map/struct.BTreeMap.html#method.remove)方法将其从tasks映射中移除。我们同样移除了它的缓存唤醒器（如果存在）。

🔗唤醒器设计  
唤醒器的职责是将被唤醒任务的ID推送给执行器的task_queue。我们通过创建一个新的TaskWaker结构体来实现这一功能，该结构体储存了任务ID和对task_queue的引用：

```rust
// in src/task/executor.rs

struct TaskWaker {
    task_id: TaskId,
    task_queue: Arc<ArrayQueue<TaskId>>,
}
```

由于task_queue的所有权在执行器和唤醒器之间是共享的，我们采用了[Arc](https://doc.rust-lang.org/stable/alloc/sync/struct.Arc.html)封装类型来实施共享的引用计数所有权。

唤醒操作的实现非常简单：

```rust
// in src/task/executor.rs

impl TaskWaker {
    fn wake_task(&self) {
        self.task_queue.push(self.task_id).expect("task_queue full");
    }
}
```

我们将task_id推送到被引用的task_queue中。因为修改ArrayQueue类型仅需要共享引用，我们可以在&self而不是&mut self上实现此方法。

🔗Wake Trait  
为了能够在轮询futures时使用我们的TaskWaker类型，我们首先需要把它转换成一个[Waker](https://doc.rust-lang.org/nightly/core/task/struct.Waker.html)实例。这是必要的，因为[Future::poll](https://doc.rust-lang.org/nightly/core/future/trait.Future.html#tymethod.poll)方法作为参数接受一个Context实例，而这只能从Waker类型构造出来。虽然我们通过提供一个[RawWaker](https://doc.rust-lang.org/stable/core/task/struct.RawWaker.html)类型的实现来达成这一点是可能的，但通过实现基于Arc的Wake Trait，然后利用标准库提供的From实现来构造Waker，是一种既更简单也更安全的方法。

该 Trait的实现如下所示：

```rust
// in src/task/executor.rs

use alloc::task::Wake;

impl Wake for TaskWaker {
    fn wake(self: Arc<Self>) {
        self.wake_task();
    }

    fn wake_by_ref(self: &Arc<Self>) {
        self.wake_task();
    }
}
```

由于唤醒器通常在执行器和异步任务之间共享，所以这些 Trait方法要求Self实例被封装在[Arc](https://doc.rust-lang.org/stable/alloc/sync/struct.Arc.html)类型中，后者实现了引用计数所有权。这意味着我们需要将我们的TaskWaker移到一个Arc中以便调用这些方法。

wake与wake_by_ref方法之间的区别在于后者仅需要对Arc的引用，而前者则接收Arc的所有权，因而经常需要增加引用计数。不是所有类型都支持通过引用来唤醒，因此实现wake_by_ref方法是可选的。然而，这可以带来更好的性能，因为它避免了不必要的引用计数变动。在我们的场景中，我们可以简单地将这两种方法都委托给我们的wake_task函数，该函数仅需一个共享的&self引用。

🔗创建唤醒器  
由于Waker类型为实现了Wake Trait的所有Arc包装的值提供了[From](https://doc.rust-lang.org/nightly/core/convert/trait.From.html)转换支持，我们现在可以实现TaskWaker::new函数，这正是我们的Executor::run_ready_tasks方法所需的。

我们使用传入的task_id和task_queue创建了TaskWaker。然后，我们将TaskWaker包裹在一个Arc中，并利用Waker::from方法将其转换为[Waker](https://doc.rust-lang.org/stable/core/task/struct.RawWakerVTable.html)。这个from方法会处理为我们的TaskWaker类型构建[RawWakerVTable](https://doc.rust-lang.org/stable/core/task/struct.RawWakerVTable.html)和[RawWaker](https://doc.rust-lang.org/stable/core/task/struct.RawWaker.html)实例的工作。如果你对它如何详细工作感兴趣，可以查看[alloc crate](https://github.com/rust-lang/rust/blob/cdb50c6f2507319f29104a25765bfb79ad53395c/src/liballoc/task.rs#L58-L87)中的实现。

🔗run方法  
有了我们的唤醒器实现，我们终于可以为我们的执行器构建一个run方法了：

```rust
// in src/task/executor.rs

impl Executor {
    pub fn run(&mut self) -> ! {
        loop {
            self.run_ready_tasks();
        }
    }
}
```

这个方法仅仅是在循环中调用run_ready_tasks函数。虽然理论上当tasks映射变空时我们可以从函数返回，但这实际上永远不会发生，因为我们的keyboard_task永远不会完成，因此一个简单的循环就足够了。由于该函数永远不会返回，我们使用!返回类型来向编译器标记这个函数为[发散](https://doc.rust-lang.org/stable/rust-by-example/fn/diverging.html)函数。

我们现在可以在kernel_main中改用新的Executor替代SimpleExecutor：

```rust
// in src/main.rs

use blog_os::task::executor::Executor; // new

fn kernel_main(boot_info: &'static BootInfo) -> ! {
    // […] initialization routines, including init_heap, test_main

    let mut executor = Executor::new(); // new
    executor.spawn(Task::new(example_task()));
    executor.spawn(Task::new(keyboard::print_keypresses()));
    executor.run();
}
```

我们仅需更改导入和类型名称。由于我们的run函数被标记为发散，编译器知道它永远不会返回，因此我们不再需要在kernel_main函数末尾调用hlt_loop了。

当我们现在使用cargo run运行我们的内核时，我们看到键盘输入依然可以工作：

![](https://os.phil-opp.com/async-await/qemu-keyboard-output-again.gif)

尽管如此，QEMU的CPU使用率并未有所改善。这是因为我们仍然让CPU全时忙碌。我们不再不停地轮询任务直到它们被唤醒，但我们依然在一个忙碌循环中检查task_queue。要解决这个问题，如果没有更多的工作，我们需要让CPU休眠。

🔗闲置时休眠   
基本想法是当task_queue为空时执行hlt指令。这个指令会使CPU休眠，直到下一个中断到来。CPU能够在中断发生时立即被激活，确保我们可以在中断处理函数将任务推送到task_queue时立即做出响应。

为了实现这一点，我们在执行器中创建了一个名为sleep_if_idle的新方法，并在run方法中调用它：

```rust
// in src/task/executor.rs

impl Executor {
    pub fn run(&mut self) -> ! {
        loop {
            self.run_ready_tasks();
            self.sleep_if_idle();   // new
        }
    }

    fn sleep_if_idle(&self) {
        if self.task_queue.is_empty() {
            x86_64::instructions::hlt();
        }
    }
}
```

由于我们在run_ready_tasks之后直接调用sleep_if_idle——而run_ready_tasks会循环直到task_queue为空，所以再次检查队列可能看起来没必要。然而，硬件中断可能在run_ready_tasks返回后立刻发生，所以在调用sleep_if_idle函数时，队列中可能已经有了新任务。只有在队列依旧为空的情况下，我们才通过[x86_64](https://docs.rs/x86_64/0.14.2/x86_64/index.html) crate提供的[instructions::hlt](https://docs.rs/x86_64/0.14.2/x86_64/instructions/fn.hlt.html)包装函数执行hlt指令，让CPU休眠。

遗憾的是，这个实现中仍然有一个微妙的竞争条件。由于中断是异步的，随时可能发生，中断可能恰好在is_empty检查与调用hlt之间发生：

```rust
if self.task_queue.is_empty() {
    /// <--- interrupt can happen here
    x86_64::instructions::hlt();
}
```

如果此中断向task_queue推送了任务，我们会在有现成的任务可执行时让CPU休眠。在最糟糕的情况下，这可能导致键盘中断的处理延迟到下一次按键或下一个定时器中断。那我们该如何防止这种情况发生呢？

答案是在检查之前禁用CPU上的中断，并与执行hlt指令一同原子性地重新启用它们。这样，期间发生的所有中断都将被延后到hlt指令之后，从而不会错过任何唤醒。为了实现这种方法，我们可以使用[x86_64](https://docs.rs/x86_64/0.14.2/x86_64/index.html) crate提供的[interrupts::enable_and_hlt](https://docs.rs/x86_64/0.14.2/x86_64/instructions/interrupts/fn.enable_and_hlt.html)函数。

我们对sleep_if_idle函数的更新实现如下所示：

```rust
// in src/task/executor.rs

impl Executor {
    fn sleep_if_idle(&self) {
        use x86_64::instructions::interrupts::{self, enable_and_hlt};

        interrupts::disable();
        if self.task_queue.is_empty() {
            enable_and_hlt();
        } else {
            interrupts::enable();
        }
    }
}
```

为了避免竞争条件，我们在检查task_queue是否为空之前禁用了中断。如果队列为空，我们就使用[enable_and_hlt](https://docs.rs/x86_64/0.14.2/x86_64/instructions/interrupts/fn.enable_and_hlt.html)函数将启用中断和CPU休眠作为一个单一的原子操作。如果队列不再为空，意味着在run_ready_tasks返回后有中断唤醒了一个任务。在这种情况下，我们再次启用中断并直接继续执行，而不执行hlt。

现在，当没有工作要做时，我们的执行器能够正确地使CPU休眠。当我们使用cargo run再次运行内核时，我们可以看到QEMU进程的CPU使用率大大降低。

### 可能的扩展

我们的执行器现在能够以一种高效的方式运行任务了。它通过利用唤醒器通知来避免对等待中的任务进行轮询，并在当前没有工作要做时使CPU休眠。尽管如此，我们的执行器仍然相对基础，还有许多可能的方式来扩展其功能：

    - **调度**：目前，我们的task_queue使用[VecDeque](https://doc.rust-lang.org/stable/alloc/collections/vec_deque/struct.VecDeque.html)类型来实现一个先入先出（FIFO）策略，这通常也被称作轮转调度。这种策略可能并不是所有工作负载的最优选择。比如，对延迟敏感的任务或是进行大量I/O操作的任务可能需要被优先考虑。更多信息请参考[《操作系统：三个简单的部分》](http://pages.cs.wisc.edu/~remzi/OSTEP/)一书的[调度章节](http://pages.cs.wisc.edu/~remzi/OSTEP/cpu-sched.pdf)或Wikipedia上的[调度文章](https://en.wikipedia.org/wiki/Scheduling_(computing))。

    - **任务生成**：我们的Executor::spawn方法当前需要一个&mut self引用，因此在调用run方法之后就无法使用了。为了解决这一问题，我们可以创建一个额外的Spawner类型，它与执行器共享某种队列，并允许任务内部生成新任务。这个队列可以直接是task_queue，也可以是执行器在其运行循环中检查的另一个独立队列。

    - **利用线程**：我们目前还不支持线程，但我们将在下一篇文章中添加这一功能。这将允许我们在不同线程中启动执行器的多个实例。这种方法的好处是它能够减少因长时间运行的任务而带来的延迟，因为其他任务可以并行运行。这种方法也使得利用多个CPU核心成为可能。

    - **负载均衡**：在添加线程支持时，知道如何在执行器之间分配任务以确保所有CPU核心得到充分利用变得尤为重要。一种常用的技术是[工作窃取](https://en.wikipedia.org/wiki/Work_stealing)。

## 结论

我们开篇介绍了多任务处理的概念，并区分了抢占式多任务处理（定期强制中断运行中的任务）和协作式多任务处理（任务运行直至它们自愿让出CPU控制权）。

接着，我们探讨了Rust对async/await的支持如何在语言层面上实现了协作式多任务处理。Rust的实现基于轮询机制的Future Trait之上，该 Trait抽象了异步任务。通过使用async/await，几乎可以像处理普通同步代码那样处理future。区别在于异步函数再次返回一个Future，而这个Future需要在某个点被加入到执行器中以便运行。

在内部，编译器将async/await代码转换为状态机，每个.await操作对应一个潜在的暂停点。借助对程序的了解，编译器能够只保存每个暂停点所需的最小状态，从而实现每个任务非常小的内存消耗。一个挑战是，生成的状态机可能包含自引用结构，例如当异步函数的局部变量互相引用时。为了防止指针失效，Rust使用Pin类型确保在首次轮询之后futures不能再在内存中移动。

在我们的实现中，我们首先创建了一个非常基础的执行器，它在一个忙碌循环中轮询所有启动的任务，完全不利用Waker类型。然后，我们通过实现一个异步键盘任务展示了唤醒器通知的优势。该任务使用crossbeam crate提供的无锁ArrayQueue类型定义了一个静态的SCANCODE_QUEUE。键盘中断处理程序不再直接处理按键事件，而是将所有接收到的扫描码放入队列中，然后唤醒注册的Waker来通知有新输入可用。在接收端，我们创建了ScancodeStream类型，提供一个解析为队列中下一个扫描码的Future。这使得创建一个使用async/await解析和打印队列中扫描码的异步print_keypresses任务成为可能。

为了利用键盘任务的唤醒器通知，我们创建了一个新的Executor类型，它使用Arc共享的task_queue存放准备好的任务。我们实现了TaskWaker类型，直接将唤醒任务的ID推送至该task_queue，随后这些任务由执行器再次轮询。为了在没有任务可运行时节约电源，我们加入了使用hlt指令使CPU休眠的支持。最后，我们讨论了一些可能的扩展，比如提供多核支持，来增强我们的执行器。

## 下一步
通过使用async/await，我们的内核现在支持了基础的协作式多任务处理。尽管协作式多任务处理效率很高，但如果单个任务运行时间过长，就会导致延迟问题，阻止其他任务的运行。因此，给我们的内核增加抢占式多任务处理的支持变得很有必要。

在接下来的文章中，我们将介绍线程——抢占式多任务处理的最常见形式。除了解决长时间运行的任务问题外，线程还将为我们利用多CPU核心以及将来运行不受信任的用户程序奠定基础。

## 支持我
创建和维护这个博客及相关库是一项巨大的工作，但我确实非常享受这一过程。通过支持我，你可以使我有更多时间投入到创造新内容、开发新功能和持续维护中。支持我的最好方式是在GitHub上成为我的[赞助](https://github.com/sponsors/phil-opp)者。感谢你的支持！

## 评论区

如果你遇到了问题，想要提供反馈，或者有更多想法想要讨论，欢迎在这里留言！请使用英语并遵守Rust的[行为守则](https://www.rust-lang.org/policies/code-of-conduct)。这个评论区直接链接到GitHub上的一个[讨论](https://github.com/phil-opp/blog_os/discussions/categories/post-comments?discussions_q=%22Async/Await%22%20in%3Atitle)，如果你愿意，也可以在那里进行评论。

