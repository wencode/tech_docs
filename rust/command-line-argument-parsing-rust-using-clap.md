# 在Rust中使用Clap解析命令行参数

在本文中，我们将探讨如何手动解析传递给Rust应用程序的命令行参数，为什么对于较大型应用来说手动解析可能并非佳选，以及Clap库是如何帮助解决这些问题的，内容包括：  
- [建立一个示例Rust应用程序](#建立一个示例Rust应用程序)
- [Clap是什么?](#Clap是什么?)
- [将Clap加入到项目中](#将Clap加入到项目中)
- [更新帮助信息](#更新帮助信息)
- [在Clap中加入标志](#在Clap中加入标志)
- [将参数设置为可选](#将参数设置为可选)
- [修复空字符串错误](#修复空字符串错误)
- [使用Clap进行日志记录](#使用Clap进行日志记录)
- [计数和查找项目](#计数和查找项目)


需要注意的是，你应该熟悉阅读和编写基本的Rust代码，比如变量声明、if-else语句块、循环以及结构体。

## 建立一个示例Rust应用程序

假设我们有一个包含很多基于Node.js的项目的文件夹，我们想要了解，“我们使用了哪些包——包括依赖包——以及使用了多少次？”

毕竟，那1GB的node_modules不可能全是唯一的依赖，对吗 😰 …？

如果我们编写一个小程序来计算我们在项目中使用某个包的次数会怎样？

为了做到这一点，我们可以在Rust中使用**cargo new package-hunter**命令来创建一个项目。现在**src/main.rs**文件中已经有了默认的main函数：

```rust
fn main() {
    println!("Hello, world!");
}
```

下一步似乎相当简单：获取传递给应用程序的参数。因此，稍后编写一个单独的函数来提取其他参数：

```rust
fn get_arguments() {
    let args: Vec<_> = std::env::args().collect(); // get all arguements passed to app
    println!("{:?}", args);
}
fn main() {
    get_arguments();
}
```

当我们运行它的时候，我们可以看到一个很好的输出结果，没有出现任何错误或程序崩溃：

```bash
# anything after '--' is passed to your app, not to cargo
> cargo run -- svelte 
    Finished dev [unoptimized + debuginfo] target(s) in 0.01s
     Running `target/debug/package-hunter svelte`
["target/debug/package-hunter", "svelte"]
```

当然，第一个参数是启动应用程序的命令，而第二个参数则是传递给它的参数。这看起来相当直接了当。

### 编写计数函数
我们现在可以愉快地继续编写计数函数了，这个函数将接收一个名称，并在子目录中计算具有该名称的目录数量：

```rust
use std::collections::VecDeque;
use std::fs;
use std::path::PathBuf;
/// Not the dracula
fn count(name: &str) -> std::io::Result<usize> {
    let mut count = 0;
    // queue to store next dirs to explore
    let mut queue = VecDeque::new();
    // start with current dir
    queue.push_back(PathBuf::from("."));
    loop {
        if queue.is_empty() {
            break;
        }
        let path = queue.pop_back().unwrap();
        for dir in fs::read_dir(path)? {
            // the for loop var 'dir' is actually a result, so we convert it
            // to the actual dir struct using ? here
            let dir = dir?;
            // consider it only if it is a directory
            if dir.file_type()?.is_dir() {
                if dir.file_name() == name {
                    // we have a match, so stop exploring further
                    count += 1;
                } else {
                    // not a match so check its sub-dirs
                    queue.push_back(dir.path());
                }
            }
        }
    }
    return Ok(count);
}
```

我们更新了**get_arguments**函数，使其返回命令之后的第一个参数，并在**main**函数中，我们使用这个参数调用count函数。

当我们在某个项目文件夹内运行这个程序时，它意外地完美运行，并返回计数为1，因为一个项目通常只包含一次特定的依赖。

### 增加深度限制
现在，当我们上升到上一级目录并尝试运行程序时，我们发现了一个问题：因为需要遍历更多的目录，所以需要更长的时间。

理想情况下，我们想从我们的项目根目录运行它，这样我们就可以找到所有包含该依赖的项目，但这样做将花费更多时间。

因此，我们决定妥协，只探索到特定深度的目录。如果某个目录的深度超过了设定的深度，它将被忽略。我们可以向函数添加另一个参数，并更新它以考虑深度：

```rust
/// Not the dracula
fn count(name: &str, max_depth: usize) -> std::io::Result<usize> {
...
queue.push_back((PathBuf::from("."), 0));
...
let (path, crr_depth) = queue.pop_back().unwrap();
if crr_depth > max_depth {
    continue;
}
...
// not a match so check its sub-dirs
queue.push_back((dir.path(), crr_depth + 1));
...   
}
```

现在，应用程序接收两个参数：首先是包名称，然后是要探索的最大深度。

但是，我们希望深度是一个可选参数，这样如果没有提供深度参数，它将探索所有子目录；如果提供了深度参数，则在给定的深度处停止。

为了实现这一点，我们可以更新get_arguments函数，使第二个参数变为可选：


```rust
fn get_arguments() {
    let args: Vec<_> = std::env::args().collect();
    let mdepth = if args.len() > 2 {
        args[2].parse().unwrap()
    } else {
        usize::MAX
    };
    println!("{:?}", count(&args[1], mdepth));
}
```

这样一来，我们可以用两种方式运行程序，而且都能正常工作：

```bash
> cargo run -- svelte
> cargo run -- svelte 5
```

但不幸的是，这种方式并不够灵活。当我们颠倒参数顺序，像这样输入 cargo run 5 package-name 时，应用程序会崩溃，因为它尝试将package-name解析为数字。

### 添加标志

现在，我们可能想要让参数具有自己的标志，比如 -f 和 -d，这样我们就可以以任何顺序输入它们。（这样也符合Unix的习惯，赚取额外的Unix积分！）

我们再次更新get_arguments函数，并且这次为参数添加一个恰当的结构体，这样就可以更容易地返回解析后的参数了：


```rust
#[derive(Default)]
struct Arguments {
    package_name: String,
    max_depth: usize,
}
fn get_arguments() -> Arguments {
    let args: Vec<_> = std::env::args().collect();
    // atleast 3 args should be there : the command name, the -f flag, 
    // and the actual file name
    if args.len() < 3 {
        eprintln!("filename is a required argument");
        std::process::exit(1);
    }
    let mut ret = Arguments::default();
    ret.max_depth = usize::MAX;
    if args[1] == "-f" {
        // it is file
        ret.package_name = args[2].clone();
    } else {
        // it is max depth
        ret.max_depth = args[2].parse().unwrap();
    }
    // now that one argument is parsed, time for seconds
    if args.len() > 4 {
        if args[3] == "-f" {
            ret.package_name = args[4].clone();
        } else {
            ret.max_depth = args[4].parse().unwrap();
        }
    }
    return ret;
}

fn count(name: &str, max_depth: usize) -> std::io::Result<usize> {
...
}

fn main() {
    let args = get_arguments();
    match count(&args.package_name, args.max_depth) {
        Ok(c) => println!("{} uses found", c),
        Err(e) => eprintln!("error in processing : {}", e),
    }
}
```

现在，我们可以使用带有 - 标志的命令来运行它，例如 cargo run -- -f svelte 或 cargo run -- -d 5 -f svelte。

### 参数和标志的问题
但是，这里存在一些严重的问题：我们可以输入两次相同的参数，从而完全跳过文件参数，比如 cargo run -- -d 5 -d 7，或者我们可以输入无效的标志，而程序却在没有任何错误信息的情况下运行 😭。

我们可以在上面第27行检查file_name是否为空来解决这个问题，并且可能在给出错误值时打印所期望的内容。但是，当我们向 -d 传递非数字值时，程序也会崩溃，因为我们直接对解析结果调用了unwrap。

此外，这个应用程序对新用户来说可能有点不友好，因为它不提供任何帮助信息。用户可能不知道要传递哪些参数以及它们的顺序，而且应用程序没有像传统Unix程序那样的 -h 标志来显示这些信息。

虽然这些对于这个特定应用来说只是一些小麻烦，但随着选项的数量和复杂性的增加，手动维护所有这些变得越来越困难。

这就是Clap库派上用场的地方。

## Clap是什么?

Clap是一个库，用于生成参数解析逻辑，为应用程序提供整洁的命令行界面，包括参数说明和 -h 帮助命令。

使用Clap相当简单，仅需对我们当前的设置做一些小改动。

Clap在许多Rust项目中有两个常用版本：V2和V3。V2主要提供基于构建器的实现，用于创建命令行参数解析器。

V3是较新的版本（在撰写本文时），它新增了派生过程宏和构建器实现，这样我们就可以注解我们的结构体，宏会为我们派生出所需的功能。

这两个版本各有优势，如果想了解更详细的差异和特性列表，我们可以查阅它们的[文档和帮助](https://github.com/clap-rs/clap)页面，这些页面提供了示例，并指导在哪些情况下派生和构建器更适用。

在这篇文章中，我们将学习如何使用带有过程宏的Clap V3。

## 将Clap加入到项目中
要在我们的项目中加入Clap库，需要在Cargo.toml文件中添加以下内容：

```toml
[dependencies]
clap = { version = "3.1.6", features = ["derive"] }
```

现在，让我们从main函数中移除get_arguments函数及其调用：
```rust
use std::collections::VecDeque;
use std::fs;
use std::path::PathBuf;
#[derive(Default)]
struct Arguments {
    package_name: String,
    max_depth: usize,
}
/// Not the dracula
fn count(name: &str, max_depth: usize) -> std::io::Result<usize> {
    let mut count = 0;
    // queue to store next dirs to explore
    let mut queue = VecDeque::new();
    // start with current dir
    queue.push_back((PathBuf::from("."), 0));
    loop {
        if queue.is_empty() {
            break;
        }
        let (path, crr_depth) = queue.pop_back().unwrap();
        if crr_depth > max_depth {
            continue;
        }
        for dir in fs::read_dir(path)? {
            let dir = dir?;
            // we are concerned only if it is a directory
            if dir.file_type()?.is_dir() {
                if dir.file_name() == name {
                    // we have a match, so stop exploring further
                    count += 1;
                } else {
                    // not a match so check its sub-dirs
                    queue.push_back((dir.path(), crr_depth + 1));
                }
            }
        }
    }
    return Ok(count);
}
fn main() {}
```

接下来，在Arguments结构体的派生部分，添加Parser和Debug：


```rust
use clap::Parser;
#[derive(Parser,Default,Debug)]
struct Arguments {...}
```

最后，在main函数中，调用parse方法：

```rust
let args = Arguments::parse();
println!("{:?}", args);
```

如果我们仅用cargo run命令运行应用程序，不加任何参数，我们会看到一个错误信息：

```bash
error: The following required arguments were not provided:
    <PACKAGE_NAME>
    <MAX_DEPTH>

USAGE:
    package-hunter <PACKAGE_NAME> <MAX_DEPTH>

For more information try --help
```

这比我们之前手动版本的错误报告要好得多！

另外，它还自动提供了一个 -h 标志来获取帮助，这个标志可以显示参数及其顺序：

```bash
package-hunter 

USAGE:
    package-hunter <PACKAGE_NAME> <MAX_DEPTH>

ARGS:
    <PACKAGE_NAME>    
    <MAX_DEPTH>       

OPTIONS:
    -h, --help    Print help information
```

现在，如果我们为MAX_DEPTH提供了非数字的内容，我们会看到一个错误提示，指出提供的字符串不是数字：

```bash
> cargo run -- 5 test
error: Invalid value "test" for '<MAX_DEPTH>': invalid digit found in string

For more information try --help
```

如果我们按照正确的顺序输入它们，我们会看到println的输出结果：

```bash
> cargo run -- test 5
Arguments { package_name: "test", max_depth: 5 }
```

所有这些改进仅仅通过增加两行代码实现，而且不需要编写任何解析代码或错误处理代码！🎉

## 更新帮助信息
目前，我们的帮助信息有些简单，因为它只显示了参数的名称和顺序。如果用户能看到每个参数的具体用途，或者甚至是应用程序的版本（以便于报告错误），那将更加有用。

Clap也提供了这方面的选项：

```rust
#[derive(...)]
#[clap(author="Author Name", version, about="A Very simple Package Hunter")]
struct Arguments{...}
```

现在，-h 输出显示了所有详细信息，并且还提供了一个 -V 标志来打印出版本号：

```bash
package-hunter 0.1.0
Author Name
A Very simple Package Hunter

USAGE:
    package-hunter <PACKAGE_NAME> <MAX_DEPTH>

ARGS:
    <PACKAGE_NAME>    
    <MAX_DEPTH>       

OPTIONS:
    -h, --help       Print help information
    -V, --version    Print version information
```

在宏中编写多行程序的关于信息可能有些繁琐，因此我们可以用 /// 来为结构体添加文档注释，宏会使用它作为关于信息（如果宏和文档注释都存在，宏中的内容将优先于文档注释）：

```rust
#[clap(author = "Author Name", version, about)]
/// A Very simple Package Hunter
struct Arguments {...}
```

这样做提供的帮助信息与之前相同。

为了增加关于参数的信息，我们可以为参数本身也添加类似的注释：

```bash
package-hunter 0.1.0
Author Name
A Very simple Package Hunter

USAGE:
    package-hunter <PACKAGE_NAME> <MAX_DEPTH>

ARGS:
    <PACKAGE_NAME>    Name of the package to search
    <MAX_DEPTH>       maximum depth to which sub-directories should be explored

OPTIONS:
    -h, --help       Print help information
    -V, --version    Print version information
```

这样之后要实用得多！

现在，让我们重新加入之前的一些功能，比如参数标志（-f 和 -d），以及将深度参数设置为可选项。

## 在Clap中加入标志

Clap让加入标志参数变得非常简单：我们只需为结构体成员添加一个Clap宏注解#[clap(short, long)]。

这里的short是指标志的简写形式，例如 -f，而long是指完整形式，如 --file。我们可以选择其中一个或同时使用两者。有了这个新增功能，我们现在有了以下内容：

```bash
package-hunter 0.1.0
Author Name
A Very simple Package Hunter

USAGE:
    package-hunter --package-name <PACKAGE_NAME> --max-depth <MAX_DEPTH>

OPTIONS:
    -h, --help                           Print help information
    -m, --max-depth <MAX_DEPTH>          maximum depth to which sub-directories should be explored
    -p, --package-name <PACKAGE_NAME>    Name of the package to search
    -V, --version                        Print version information
```

由于两个参数都设有标志，现在没有位置参数了；这意味着我们无法运行 cargo run -- test 5，因为Clap会查找标志，并会提示错误，表示参数未提供。

相反，我们可以运行 cargo run -- -p test -m 5 或 cargo run -- -m 5 -p test，它会正确地解析这两个参数，给出以下输出：

```bash
Arguments { package_name: "test", max_depth: 5 }
```

由于我们总是需要包名，我们可以把它设置为位置参数，这样我们就不需要每次都输入 -p 标志。

要做到这一点，从它那里移除 #[clap(short,long)]；现在，不带任何标志的第一个参数将被视作包名：

```bash
> cargo run -- test -m 5
Arguments { package_name: "test", max_depth: 5 }
> cargo run -- -m 5 test
Arguments { package_name: "test", max_depth: 5 }
```

在使用简写参数时要注意的一点是，如果两个参数都以相同的字母开头——例如package-name和path——并且它们都启用了短标志，那么应用程序在调试构建时会在运行时崩溃，并且在发布构建中会出现一些令人困惑的错误信息。

因此，请确保：

* 所有参数都以不同的字母开头
* 或者，只有一个以相同起始字母的参数启用了短标志

接下来的步骤是将max_depth设置为可选。

## 将参数设置为可选

要使任何参数成为可选项，只需将该参数的类型设置为Option<T>，其中T是原来的类型参数。因此，在我们的例子中，我们有如下设置：

```rust
#[clap(short, long)]
/// maximum depth to which sub-directories should be explored
max_depth: Option<usize>,
```

这应该可以奏效。这种改变也会体现在帮助信息中，其中不会把最大深度列为必需参数：

```bash
package-hunter 0.1.0
Author Name
A Very simple Package Hunter

USAGE:
    package-hunter [OPTIONS] <PACKAGE_NAME>

ARGS:
    <PACKAGE_NAME>    Name of the package to search

OPTIONS:
    -h, --help                     Print help information
    -m, --max-depth <MAX_DEPTH>    maximum depth to which sub-directories should be explored
    -V, --version                  Print version information
```
而且，我们可以在不使用 -m 标志的情况下运行它：

```bash
> cargo run -- test
Arguments { package_name: "test", max_depth: None }
```

但这样仍然有点不便；现在我们必须对max_depth使用match，如果它是None，我们需要像之前一样将其设置为usize::MAX。

不过，Clap还有另一种方案！我们可以为参数设置默认值，而不是将其类型设为Option<T>。

所以，进行如下修改之后：

```rust
#[clap(default_value_t=usize::MAX,short, long)]
/// maximum depth to which sub-directories should be explored
max_depth: usize,
```

我们可以在提供或不提供max_depth的值时运行应用程序（usize的最大值取决于您的系统配置）：

```bash
> cargo run -- test
Arguments { package_name: "test", max_depth: 18446744073709551615 }
> cargo run -- test -m 5
Arguments { package_name: "test", max_depth: 5 }
```

现在，让我们像之前一样在main函数中将其与count函数连接起来：

```rust
fn main() {
    let args = Arguments::parse();
    match count(&args.package_name, args.max_depth) {
        Ok(c) => println!("{} uses found", c),
        Err(e) => eprintln!("error in processing : {}", e),
    }
}
```

通过这样做，我们不仅恢复了原始的功能，而且代码更加简洁，并增加了一些额外的特性！

## 修复空字符串错误
package-hunter基本上按照预期运行，但遗憾的是，从手动解析阶段开始就存在一个微妙的错误，且一直保留在基于Clap的版本中。你能猜出是什么错误吗？

尽管对于我们这个小应用来说，这个错误并不非常严重，但它可能会成为其他应用程序的致命弱点。在我们的例子中，它会在应该报错的情况下给出错误的结果。

尝试运行以下命令：

```bash
> cargo run -- ""
0 uses found
```

在这里，当空包名不应被允许时，package_name却作为空字符串传递进来。这是因为我们运行命令的shell将参数传递给应用程序的方式导致的。

通常，shell使用空格分割传递给程序的参数列表，所以 abc def hij 会被视为三个独立的参数：abc、def和hij。

如果我们想要在一个参数中包含空格，我们必须将其放在引号中，如 "abc efg hij"。这样shell就知道这是一个单独的参数，并将其作为整体传递。

另一方面，这也意味着我们可以向应用程序传递空字符串或只包含空格的字符串。再次，Clap提供了解决方案！它提供了一种方法来拒绝接受参数的空值：


```rust
#[clap(forbid_empty_values = true)]
/// Name of the package to search
package_name: String,
```
有了这个设置，如果我们尝试使用空字符串作为参数，我们会得到一个错误提示：

```rust
> cargo run -- ""
error: The argument '<PACKAGE_NAME>' requires a value but none was supplied
```

但是，这仍然允许使用只包含空格的字符串作为包名，意味着""是一个有效参数。为了解决这个问题，我们需要提供一个自定义验证器，该验证器将检查名称是否包含前导或尾随空格，并在存在这种情况时拒绝它。

我们的验证函数定义如下：

```rust
fn validate_package_name(name: &str) -> Result<(), String> {
    if name.trim().len() != name.len() {
        Err(String::from(
            "package name cannot have leading and trailing space",
        ))
    } else {
        Ok(())
    }
}
```

然后，按照以下方式为package_name设置验证器：

```rust
#[clap(forbid_empty_values = true, validator = validate_package_name)]
/// Name of the package to search
package_name: String,
```

现在，如果我们尝试传递一个空字符串或只包含空格的字符串，它将提示错误，正如我们预期的：

```bash
> cargo run -- "" 
error: The argument '<PACKAGE_NAME>' requires a value but none was supplied
> cargo run -- " "
error: Invalid value " " for '<PACKAGE_NAME>': package name cannot have leading and trailing space
```

通过这种方式，我们可以利用自定义逻辑对参数进行验证，而无需编写解析它们的全部代码。

## 使用Clap进行日志记录

现在应用程序运行得很好，但我们无法知道在它未能正常运行时发生了什么。为此，我们应该记录下应用程序的操作日志，以便在出现故障时查看发生了什么。

像其他命令行应用程序一样，我们应该允许用户轻松设置日志级别。默认情况下，它应该只记录主要细节和错误，以防日志过于杂乱，但在应用程序崩溃时，应该有一种模式可以记录尽可能多的信息。

像其他应用程序一样，让我们的应用使用 -v 标志来设置冗长级别；不使用标志是最低级别的日志记录，-v 是中等级别的日志记录，-vv 是最高级别的日志记录。

为此，Clap提供了一种方法，可以将参数的值设置为它出现的次数，这正是我们在这里所需要的！我们可以添加另一个参数，并按以下方式设置：


```rust
#[clap(short, long, parse(from_occurrences))]
verbosity: usize,
```

现在，如果我们运行程序时没有使用 -v 标志，它的值将为零；如果使用了 -v 标志，则会计算 -v 标志出现的次数：

```bash
> cargo run -- test
Arguments { package_name: "test", max_depth: 18446744073709551615, verbosity: 0 }
> cargo run -- test -v
Arguments { package_name: "test", max_depth: 18446744073709551615, verbosity: 1 }
> cargo run -- test -vv
Arguments { package_name: "test", max_depth: 18446744073709551615, verbosity: 2 }
> cargo run -- -vv test -v
Arguments { package_name: "test", max_depth: 18446744073709551615, verbosity: 3 }
```

利用这个值，我们可以轻松地初始化日志记录器，并使其记录适当数量的详细信息。

我在这篇文章中没有添加虚拟日志代码，因为本文的重点是参数解析，但你可以在文章结尾的代码仓库中找到相关代码。

## 计数和查找项目

现在我们的应用程序运行得很好，我们想要添加另一个功能：列出我们拥有的项目。这样，当我们需要快速获取一个项目列表时，就可以轻松实现了。

Clap具有一个强大的子命令功能，可以为应用程序提供多个子命令。要使用它，需要定义一个带有自己参数的结构体作为子命令。主参数结构体包含所有子命令共有的参数，然后是子命令。

我们将按照以下方式构建我们的CLI：

* 日志详细程度和最大深度参数将位于主结构体中
* count命令将接收文件名以进行查找并输出计数
* projects命令接受一个可选的起始路径开始搜索
* projects命令接受一个可选的排除路径列表，用于跳过指定的目录

因此，我们按如下方式添加count和project枚举：

```rust
use clap::{Parser, Subcommand};
...
#[derive(Subcommand, Debug)]
enum SubCommand {
    /// Count how many times the package is used
    Count {
        #[clap(forbid_empty_values = true, validator = validate_package_name)]
        /// Name of the package to search
        package_name: String,
    },
    /// list all the projects
    Projects {
        #[clap(short, long, default_value_t = String::from("."),forbid_empty_values = true, validator = validate_package_name)]
        /// directory to start exploring from
        start_path: String,
        #[clap(short, long, multiple_values = true)]
        /// paths to exclude when searching
        exclude: Vec<String>,
    },
}
```

在这里，我们把package_name移动到Count变体中，并在Projects变体中添加start_path和exclude选项。

现在，如果我们查看帮助信息，它会列出这两个子命令，并且每个子命令都有其专属的帮助信息。

接下来，我们可以更新main函数以适配这些子命令：

```rust
let args = Arguments::parse();
match args.cmd {
    SubCommand::Count { package_name } => match count(&package_name, args.max_depth, &logger) {
        Ok(c) => println!("{} uses found", c),
        Err(e) => eprintln!("error in processing : {}", e),
    },
    SubCommand::Projects {
        start_path,
        exclude,
    } => {/* TODO */}
}
```

我们还可以像以前一样使用count命令来计算使用次数：

```bash
> cargo run -- -m 5 count test
```

由于max_depth是在主Arguments结构体中定义的，所以它必须在子命令之前指定。

我们可以根据需要为projects命令的exclude（排除）目录选项提供多个值：

```bash
> cargo run -- projects -e ./dir1 ./dir2
["./dir1", "./dir2"] # value of exclude vector
```

我们还可以设置一个自定义分隔符，以防我们不想使用空格分隔值，而是使用自定义字符分隔：

```rust
#[clap(short, long, multiple_values = true, value_delimiter = ':')]
/// paths to exclude when searching
exclude: Vec<String>,
```

现在我们可以使用 : 来分隔值：

```bash
> cargo run -- projects -e ./dir1:./dir2
["./dir1", "./dir2"]
```

这样就完成了应用程序的CLI设计。这里没有展示项目列表功能的实现，但你可以尝试自己编写，或者在GitHub仓库中查看其代码。

##结论

现在你已经了解了Clap，你可以为你的项目创建清晰、优雅的命令行界面。Clap还拥有许多其他功能，如果你的项目需要特定的命令行功能，很可能Clap已经提供了这些功能。

你可以查阅Clap的[文档](https://docs.rs/clap/latest/clap/)和Clap的[GitHub页面](https://github.com/clap-rs/clap)，以获取关于Clap库提供的更多选项的信息。

你还可以在[这里](https://github.com/YJDoc2/LogRocket-Blog-Code/tree/main/arg-parsing-with-clap)获取这个项目的代码。感谢你的阅读！
