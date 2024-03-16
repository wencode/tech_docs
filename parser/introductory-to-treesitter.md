# TreeSitter简介

*原文：[Introductory To Treesitter](https://teknologiumum.com/posts/introductory-to-treesitter)*

本文简要介绍了树形解析器，这是一个适用于文本编辑器的通用解析工具，它的用途远不止于文本编辑，有许多其他酷炫的应用。  
本文发布于2022-01-27。

在当下的世界里，我们的编码工作越来越依赖于文本编辑器或集成开发环境（IDE）所提供的智能功能。对于大部分人而言，IDE和文本编辑器不过是编写代码的辅助工具，因此他们可能不太关注这些工具的内部工作机制（毕竟，它只不过是个文本编辑器，对吧）。但也有人，像我一样，对文本编辑器背后的技术细节充满好奇。

我发现，了解这些工具如何在底层运作非常有趣。例如，[语言服务器协议（LSP）](https://microsoft.github.io/language-server-protocol/)统一了诸如跳转到定义、根据上下文自动补全代码、预览引用等智能语言特性的实现。还有一个类似于LSP但相对较不为人知且使用较少的是调试适配器协议（DAP）。而我在这篇文章中想要深入讨论的工具是树形解析器（Treesitter）。

## 树形解析器是什么
引述自官方网站：

    树形解析器是一款解析器生成工具和增量解析库。它能够为源代码文件构建出一个具体的语法树，并且能够在源代码编辑过程中高效地更新这个语法树。

简而言之，树形解析器是Github开发的一个极快的增量解析器，其速度之快，理论上能够实现在你每次按键时即时增量解析你的文件。它还具备错误恢复功能，这意味着即使文件中存在错误，也不会影响到文件其余部分的抽象语法树（AST）的结构。

我之所以说“理论上”，是因为其实际表现还取决于具体的解析器实现。有些语言需要复杂的手工编写的解析规则，这可能会降低解析速度。当然，手工编写的解析器的效能好坏也是一个影响因素。

## 为什么我要使用树形解析器？

在讨论树形解析器时，我们通常指的是在文本编辑器中的应用。树形解析器能够提供一个与你的代码实时同步的抽象语法树（AST）。你可能会问，在文本编辑器中为何需要AST？原因在于，与基于正则表达式的语法高亮相比，AST能够实现更为准确的语法高亮。此外，它还能提供更加智能的代码折叠、代码导航和结构化编辑等功能。

从一种编程语言生成AST并非易事，这需要使用到解析器。而构建一个既快速以至于能够实现每次键盘敲击时的增量解析、又能进行错误恢复的解析器更是一项挑战。通过使用树形解析器，我们可以“免费”获得这些高级功能！（实际上，并不完全免费，因为你还需要自行构建语法规则，不过这个话题我们留到后面再讨论。）

## 树形解析器的应用场景
我主要使用Neovim，所以接下来我会着重讲解在Neovim中树形解析器的应用场景。

### 更精准的语法高亮
Neovim继承自Vim的传统语法高亮系统基于正则表达式，这是大多数编辑器的共通之处。这种方法导致了复杂难懂且运行缓慢的正则表达式规则。相比之下，树形解析器提供的语法高亮不仅更快、更精确，其查询文件——决定代码高亮方式的文件——也更为清晰易读。举个例子：

```javascript
" Operators;
" match single-char operators:          - + % < > ! & | ^ * =
" and corresponding two-char operators: -= += %= <= >= != &= |= ^= *= ==
syn match goOperator /[-+%<>!&|^*=]=\?/
" match / and /=
syn match goOperator /\/\%(=\|\ze[^/*]\)/
" match two-char operators:               << >> &^
" and corresponding three-char operators: <<= >>= &^=
syn match goOperator /\%(<<\|>>\|&^\)=\?/
" match remaining two-char operators: := && || <- ++ --
syn match goOperator /:=\|||\|<-\|++\|--/
```

再对比一下这个：

```javascript
; Operators
[
  "--" "-" "-=" ":=" "!" "!=" "..." "*" "*" "*=" "/" "/="
  "&" "&&" "&=" "%" "%=" "^" "^=" "+" "++" "+=" "<-" "<"
  "<<" "<<=" "<=" "=" "==" ">" ">=" ">>" ">>=" "|" "|=" "||" "~"
] @operator
```

因为复杂的解析工作交给了解析器，所以看起来简化了很多。查询过程仅需将特定节点“映射”到对应的高亮组，这正是@operator部分的作用所在。

不过，Neovim目前的实现还不尽完美。在某些特殊情况下，它可能无法正确地进行语法高亮，这大多是由于语法规则设计得不够好，而非Neovim本身的问题。通过对比这两幅图像，你可以明显看到语法高亮的质量有了多大的提升。

![](https://i.ibb.co/BypQxTB/cmp.png)
*comparison*

实际上，它的颜色可以更丰富，但我选择的配色方案并不那么绚烂，因为过多的颜色会让我感到分心 :p

我推荐观看Max Brunsfeld的[演讲](https://youtu.be/Jes3bD6P0To?t=372)，相比在这里展示的单张图片，它提供了更为深入的比较。

你可能认为，来自语言服务器的语义语法高亮功能更加强大，确实是这样，但这种强大也有其局限。那么，为什么还要使用树形解析器呢？正如任何其他技术一样，两者都有其优点和缺点。

* 可移植性

树形解析器几乎可以在任何地方使用，而语言服务器的语义高亮功能仅在能够运行语言服务器的环境中可用。比如，你可以在浏览器中使用树形解析器，但不能使用语言服务器。

* 语义能力

由于树形解析器仅在单一文件中工作，其上下文理解能力相比于语义高亮大大受限。以下是一个例子：

![](https://i.ibb.co/VYZC0Rw/Shot-2022-01-27-17-06-33.png)
*useState*

树形解析器（第一行）无法识别setState是否为一个函数，因为它是从另一个文件中调用的，因此它将该节点视为普通变量。而语义高亮（第二行）能够识别setDebouncedValue是一个函数，并据此进行高亮。

* 性能优势

在大多数情况下，树形解析器的性能远超出预期。因为它仅限于分析单个文件，减少了处理量，加之解析器本身采用C语言编写，使得其运行速度非常快。相比之下，语义高亮的性能并不总是那么理想，部分原因是某些语言服务器使用的编程语言较慢。然而，如果手写的解析规则不够优化，树形解析器的性能也可能受到影响。

### 结构化代码编辑
在传统的文本编辑中，我们处理的内容通常是无结构的，仅仅由行和列构成。传统文本编辑器无法识别出哪些部分属于语法树的节点。然而，借助树形解析器，我们能够获取这些详细信息，进而实现结构化的文本编辑。

如果你有Lisp编程背景，那你可能已经对结构化编辑有所了解。这种编辑方式不是简单地修改文本，而是对其语法树进行编辑。比如，不是选择从第12行到第30行，或者从第3列到第40列（实际上，我们大多数人都不会这样考虑，而是会直接将光标移动到想要选择的地方），你可以直接命令“选择这个函数”或“选择这个类”或“移动这个函数”，系统会自动为你完成。

Neovim中包含了一个增量选择功能，允许用户按照语法树的结构逐渐扩大或缩小选择范围。不再是“选择这个部分到那个部分”，而是“从这个节点选择到其第四层父节点”。以下是一个快速演示：

![](https://i.ibb.co/cNTxp7n/ezgif-com-gif-maker.gif)

EmacsConf2021上还有一场[演讲](https://www.youtube.com/watch?v=FwDsuz0waIY)，它以更详细的方式展示了结构化编辑的实际操作。

### 更加智能的代码折叠
当我们需要浏览代码但又不希望被其中的某些细节所干扰时，代码折叠功能就显得非常有用。

一般而言，代码折叠是基于缩进进行的。例如，有如下代码时：

```c++
void some_function(std::string foo, std::string bar, int baz, int qux) {
    // some long function implementation
    // doing something really important
}
```

它的折叠效果会符合我们的预期：

```c++
void some_function(std::string foo, std::string bar, int baz, int qux) {...
}
```

但是，当你遇到一个像下面这样的函数时，情况就会变得复杂，这种情况在函数有很多参数并且你想要对齐这些参数时非常常见。

```c++
void some_function(std::string foo,
                   std::string bar,
                   int baz,
                   int qux) {
    // some long function implementation
    // doing something really important
}
```

其折叠效果会是这样的：

```c++
void some_function(std::string foo,...
}
```

确实，效果不太理想，几乎所有的参数都被折叠了。但是如果使用树形解析器，其效果会是这样：

```c++
void some_function(std::string foo,
                   std::string bar,
                   int baz,
                   int qux) {...
}
```

因为我们可以利用抽象语法树（AST），它能够准确识别出哪些节点需要被折叠。无论你的代码缩进方式如何，只要生成的AST是准确的，它就能够正确地执行代码折叠。

Max Brunsfeld在他的演讲中也专门讲述了[树形解析器如何使代码折叠变得更加智能](https://youtu.be/Jes3bD6P0To?t=749)。

## 目前已采用树形解析器的编辑器
到目前为止，已经有几款编辑器开始使用树形解析器，虽然这样的编辑器还不多。

Atom是一个显而易见的例子，因为它是由Github开发的。它利用树形解析器实现了代码高亮、代码折叠、增量选择等功能。完整的公告可以在[这里](https://github.blog/2018-10-31-atoms-new-parsing-system/)阅读。

Visual Studio Code推出了一个名为[vscode-anycode](https://github.com/microsoft/vscode-anycode)的扩展，它基本上是一个基于树形解析器的轻量级、简化版语言服务器的替代品。它适用于那些不能运行真正语言服务器的环境，比如[github.dev](https://github.dev/)和[vscode.dev](https://vscode.dev/)。然而，其准确性可能不及真正的语言服务器。

从v0.5版本开始，Neovim实现了树形解析器，但目前仍处于初步的测试阶段。有些情况下，它的表现可能不如人们所期望的那样优秀，但到了v0.7版本（本文撰写时该版本在主分支上，还未正式发布），情况有了明显改善。

Emacs也实施了树形解析器的使用，虽然我对此不太了解。他们有一个非常[详细的网站](https://emacs-tree-sitter.github.io/)，覆盖了其功能等信息。

..以及一些其他不太常用的编辑器，例如[Helix](https://helix-editor.com/)、[Lapce](https://helix-editor.com/)等。

## 树形解析器的非文本编辑应用

树形解析器的应用并不仅限于文本编辑，它还能用于许多与文本编辑无关的领域。例如，[tjdevries/tree-sitter-lua](https://github.com/tjdevries/tree-sitter-lua)项目提供了一个将EmmyLua文档转换成Vimdoc格式的文档生成工具。[sunjon/telescope-arecibo.nvim](https://github.com/sunjon/telescope-arecibo.nvim)项目则利用树形解析器作为一种替代浏览器DOM选择器API的方法（如查询节点、获取节点的内容等）。还有[mjlbach/babelfish.nvim](https://github.com/mjlbach/babelfish.nvim)项目，它实质上是一个Markdown到Vimdoc的转换器。基本上，任何需要抽象语法树(AST)的场景，都可以通过树形解析器来实现。

## 树形解析器的解析机制
和其他解析器一样，树形解析器需要一种“语法”规则来指导其如何解析文档。树形解析器的语法规则是通过一个叫作grammar.js的文件用一种叫做DSL（特定领域语言）的方式来编写的。其内容如下所示：

```javascript
module.exports = grammar({
    // the grammar's name
    name: "javascript",
 
    // these are the nodes for the hand-written parsing rule to consume
    externals: ($) => [$._automatic_semicolon, $._template_chars, $._ternary_qmark],
 
    // the parsing rules
    rules: {
        program: ($) => seq(optional($.hash_bang_line), repeat($.statement)),
 
        hash_bang_line: ($) => /#!.*/,
 
        // the rest of the rules goes here
    },
});
```

由于这个过程相当复杂，我推荐你阅读树形解析器的[官方文档](https://tree-sitter.github.io/tree-sitter/creating-parsers)，该文档全面覆盖了创建自己的解析器所需了解的各个方面。

## 树形解析器查询
树形解析器的查询功能，如其名称所示，允许我们像使用CSS选择器查询HTML文档一样查询AST。这一功能通过一种非常简单的Scheme语言来实现。其形式如下：

```
(some_node (node) @target_name)
```

通过@target_name，可以将特定的节点绑定到target_name。此外，它提供了一些特殊的操作符来帮助捕获更复杂的模式，例如通配符、否定和量化等，这些与正则表达式的使用类似。

有关更多细节，官方文档中有一个完整的专门介绍这方面的[章节](https://tree-sitter.github.io/tree-sitter/using-parsers#pattern-matching-with-queries)。

结语
树形解析器是一种相对较新的技术，目前还未得到广泛应用。如果你想开始探索树形解析器，我推荐从[官方文档](https://tree-sitter.github.io/)着手，那里有非常详细的说明。
