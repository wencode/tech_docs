#一个下午编写树形解析器语法的方法

*原文：[How to write a tree-sitter grammar in an afternoon](https://siraben.dev/2022/03/01/tree-sitter.html)*  
2022-3-1

为何选择树形解析器？
我们终于有了一个树形解析器能够编译的语法了。那我们怎么实际上运行它呢？
语法高亮
美化打印
代码检查
在编辑器中实现IDE-like的特性（自动补全、结构导航、跳转到定义）
测试语法

这篇文章曾在[Hacker News](https://news.ycombinator.com/item?id=30661127)上引发讨论。

随着每一个十年的过去，实现新编程语言的任务似乎变得越来越简单。解析器生成器解决了解析的难题，还能提供信息丰富的错误提示。宿主语言中的表达式类型系统使得我们能够轻松地对递归语法树进行模式匹配，并在我们遗漏某些情况时提醒我们。属性测试和模糊测试让我们能够比以往任何时候都更快、更全面地测试边缘情况。编译到LLVM等中间语言可以让即使是最简单的语言也能拥有合理的性能。

假如你刚刚创造了一种新语言，并利用了编程语言领域的最新最伟大技术，如果你希望人们真正采用并使用它，你接下来应该关注什么？我认为，应该是编写一个树形解析器语法。在我详细解释树形解析器是什么之前，这里有一些你能够更容易实现的功能：

- 语法高亮
- 美化打印
- 代码检查
- 在编辑器中实现IDE-like的特性（自动补全、结构导航、跳转到定义）

最棒的是，这一切你可以在一个下午完成！在这篇文章中，我们将为一个简单的命令式语言[Imp](https://softwarefoundations.cis.upenn.edu/lf-current/Imp.html)编写一个语法，你可以在[这里](https://github.com/siraben/tree-sitter-imp)获取源代码。

这篇文章的灵感来源于我在提升[FORMULA](https://github.com/siraben/tree-sitter-formula)和[Spin](https://github.com/siraben/tree-sitter-promela)的开发者体验方面的研究。

## 为何选择树形解析器？
树形解析器是一个解析器生成工具，但与其他解析器生成工具不同的是，它在进行增量解析上特别优秀，能够在输入存在语法错误的情况下也创建出有用的解析树。最重要的是，它速度极快、无依赖性，能让你在每次敲击键盘的瞬间以毫秒为单位解析整个文件。生成的解析器采用C语言编写，同时提供了多种编程语言的[绑定](https://tree-sitter.github.io/tree-sitter/#language-bindings)，这意味着你也可以通过编程方式来遍历这棵树。

## 为Imp设计的树形解析器语法
[Imp](https://softwarefoundations.cis.upenn.edu/lf-current/Imp.html)是一种常用于编程语言理论教学的简单命令式语言。它包含了算术表达式、布尔表达式以及包括顺序执行、条件判断和while循环在内的各种语句。

以下是一个Imp程序示例，它计算变量x的阶乘并将结果存储在变量y中。

```lua
// Compute factorial
z := x;
y := 1;
while ~(z = 0) do
  y := y * z;
  z := z - 1;
end
```

## 项目设置指南
请参阅树形解析器的[官方开发指南](https://tree-sitter.github.io/tree-sitter/creating-parsers#getting-started)。

如果你使用的是Nix系统，可以通过运行nix shell nixpkgs#tree-sitter nixpkgs#nodejs-16-x命令来启动一个包含所需依赖的shell环境。

需要注意的是，为了继续阅读这篇文章，你不必实际上完成这些设置，因为我会在文章的关键部分提供终端的输出内容。

## 编写语法规则
###表达式

首先，我们根据教材中提供的表达式语法进行编写。以下是参考语法：

```javascript
a := nat
   | id
   | a + a
   | a - a
   | a * a
   | (a)

b := true
   | false
   | a = a
   | a <= a
   | ~b
   | b && b
```

其中，a代表算术表达式，b代表布尔表达式。

最简单的部分是处理数字和变量，我们可以添加如下规则：

```javascript
id: $ => /[a-z]+/,
nat: $ => /[0-9]+/,
```

算术表达式的语法可以轻松地进行转换：

```javascript
program: $ => $.aexp,
aexp: $ => choice(
    /[0-9]+/,
    /[a-z]+/,
    seq($.aexp,'+',$.aexp),
    seq($.aexp,'-',$.aexp),
    seq($.aexp,'*',$.aexp),
    seq('(',$.aexp,')'),
),
```

### 定义优先级和结合律
尝试编译一下看看！树形解析器给出的反馈如下：

```bash
Unresolved conflict for symbol sequence:

  aexp  '+'  aexp  •  '+'  …

Possible interpretations:

  1:  (aexp  aexp  '+'  aexp)  •  '+'  …
  2:  aexp  '+'  (aexp  aexp  •  '+'  aexp)

Possible resolutions:

  1:  Specify a left or right associativity in `aexp`
  2:  Add a conflict for these rules: `aexp`
```

树形解析器立即指出我们的规则存在歧义，即同一序列的标记可以构成不同的解析树。我们在编写代码时肯定不希望存在这种歧义！因此，让我们将所有操作定义为左结合性：

```javascript
program: $ => $.aexp,
aexp: $ => choice(
    /[0-9]+/,
    /[a-z]+/,
    prec.left(1,seq($.aexp,'+',$.aexp)),
    prec.left(1,seq($.aexp,'-',$.aexp)),
    prec.left(1,seq($.aexp,'*',$.aexp)),
    seq('(',$.aexp,')'),
),
```

但是，当我们尝试解析1*2-3*4时，结果似乎并不完全正确：
![](https://siraben.dev/assets/parse.svg)

它被解析成了((1*2)-3)*4，这显然是一种不同的解释方式！我们可以通过为乘法运算符*指定prec.left(2,...)来解决这个问题。这样，我们得到的解析树正是我们所期望的。

值得注意的是，在许多真实的编程语言规范中，都会明确给出二元运算符的优先级，所以确定运算符的结合性和优先级通常相当直接。

### 布尔表达式和语句

布尔表达式和语句的语法结构类似，在随附的代码[仓库](https://github.com/siraben/tree-sitter-imp/blob/master/grammar.js)中可以找到详细信息。

## 测试语法

我们终于有了一个树形解析器能够编译的语法了。那我们怎么实际上运行它呢？树形解析器CLI提供了两个子命令来帮助我们：tree-sitter parse和tree-sitter test。parse子命令需要一个文件路径作为输入，并使用当前的语法解析该文件，然后将解析树输出到标准输出。test子命令运行一组在简单语法中定义的测试：

```bash
===
skip statement
===
skip
---

(program
  (stmt
    (skip)))
```

测试的名称由等号行标识，紧接着是待解析的程序代码，然后是一行短横线，后面是预期的解析树。

当我们执行tree-sitter test命令时，如果测试通过，会显示一个勾选标记；如果测试失败，则显示一个叉号，同时会展示预期解析树与实际解析树之间的差异对比（为了示例错误，我将示例代码替换成了skip; skip）：

```bash
tests:
    ✗ skip
    ✓ assignment
    ✓ prec
    ✓ prog

1 failure:

expected / actual

  1. skip:

    (program
      (stmt
        (seq
          (stmt
            (skip))
          (stmt
            (skip)))))
        (skip)))
```

## 语法高亮的魅力！
不管你信不信，编写一个树形解析器语法基本上就是这么简单！我们可以马上利用它来实现语法高亮。传统的编辑器中的语法高亮技术依靠正则表达式和一些特定的启发式方法来对令牌进行颜色标记，但由于树形解析器能够访问整个解析树，它不仅能够为标识符、数字和关键字等进行着色，还能够根据上下文进行着色——比如，一致地高亮本地变量和用户自定义类型。

tree-sitter highlight命令允许你对源代码进行[语法高亮](https://tree-sitter.github.io/tree-sitter/syntax-highlighting)，并能在终端中渲染或输出为HTML格式。

树形解析器的语法高亮是基于查询的。关键的一步是，我们需要为树中不同的节点指定高亮名称。对这种简单语言来说，我们只需要以下5行代码。方括号用来表示选择项，意味着如果树中的任何节点匹配列表中的某一项，则将相应的捕获名称（以@前缀）指定给它。

```javascript
[ "while" "end" "if" "then" "else" "do" ] @keyword
[ "*" "+" "-" "=" ":=" "~" ] @operator
(comment) @comment
(num) @number
(id) @variable.builtin
```

这里是在阶乘程序上使用tree-sitter highlight --html命令得到的结果

```lua
// Compute factorial
z := x;
y := 1;
while ~(z = 0) do
 y := y * z;
 z := z - 1;
end
```

效果非常好！运算符、关键词、数字和标识符都被清楚地高亮显示，而注释则以灰色和斜体显示，使代码更加易于阅读。

## 接下来可以做的

创建一个树形解析器语法只是一个开始。既然你已经拥有了一种即使面对语法错误也能快速可靠地生成语法树的方法，你可以以此为基础来开发更多工具。我将在下面简要介绍一些话题，但它们实际上值得在未来的博客文章中进行深入探讨。

### 进阶的语法高亮
有了树形解析器，我们可以使语法高亮在语义上变得更加丰富。也就是说，我们可以让语法高亮器用一种颜色来标记局部变量名，用另一种颜色来标记全局变量，区分字段访问和方法访问等。使用基于正则表达式的高亮器来实现这种细致的高亮几乎是不可能的，这就像用[正则表达式来解析HTML](https://stackoverflow.com/questions/1732348/regex-match-open-tags-except-xhtml-self-contained-tags)一样无效。

### 编辑器集成

![](https://siraben.dev/assets/emacs-querying.png)

树形解析器语法被编译为动态库，能够加载到包括Emacs、Atom和VS Code在内的各种编辑器上，支持包括WebAssembly在内的所有平台。借助各编辑器的扩展机制，你可以开发基于这些编辑器的包，利用语法树进行多种操作，如结构化代码导航、查询特定节点的语法树（参见截图），当然还包括语法高亮。以下是一份利用树形解析器提升编辑体验的项目不完全清单：

- [在Emacs中实现树形感知编辑](https://github.com/ethan-leba/tree-edit)
- [在VS Code中提供不完整的IDE功能](https://github.com/microsoft/vscode-anycode)
- [为Neovim提供的自动完成框架](https://github.com/nvim-treesitter/completion-treesitter)

### 代码检查器
树形解析器在多种语言中提供了[绑定](https://tree-sitter.github.io/tree-sitter/#language-bindings)。利用这些信息和树形解析器的查询语言，你可以遍历语法树，寻找编程语言中的特定模式（或不良模式）。要查看这在Imp中的应用，请参考我用[JavaScript绑定进行Imp代码检查](https://github.com/siraben/ts-lint-example)的最小示例。更多细节将在未来的文章中分享！

## 最终思考
从计算机科学几乎一个世纪前诞生以来，解析技术已经走过了漫长的道路（参见这个[出色的解析技术时间线](https://jeffreykegler.github.io/personal/timeline_v3)）。我们从无法处理递归表达式和运算符优先级，发展到了LALR解析器生成器，现在又有了GLR和树形解析器的快速增量解析技术。有充分的理由相信，成百上千的开发者每天使用的查看代码的工具应该利用这些技术发展。我们可以做得比基于行的编辑或依赖于粗糙的正则表达式来变换和高亮代码更好。未来是结构化的，树形解析器或许将在这一未来中扮演重要的角色！

