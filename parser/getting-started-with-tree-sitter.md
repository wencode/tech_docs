树形解析器入门

*原文：[Getting started with tree-sitter](https://dcreager.net/2021/06/getting-started-with-tree-sitter/)*
2021-06-07

这是关于使用树形解析器解析框架的一系列文章的开篇。这些文章的初期目标受众主要是那些希望利用现有的语法规则编写跨多种编程语言的程序分析工具的人士。目前，我还未涉及如何为新编程语言创建新的语法规则。

我们的起点会保持简单。本文将指导你如何安装树形解析器的命令行工具和Python语法规则，并使用这些工具来解析和高亮显示一些Python代码。

## 安装树形解析器
首先，你需要安装树形解析器。你可以选择多种安装方法：

### 使用本地包管理器
在一些平台上，可以通过本地包管理器安装树形解析器。例如，在Arch Linux上，你可以通过‘pacman’命令安装树形解析器：

```bash
$ sudo pacman -S tree-sitter
$ tree-sitter --version
tree-sitter 0.19.5
```

同样地，如果你使用的是Mac，可以通过Homebrew安装：

```bash
$ brew install tree-sitter
$ tree-sitter --version
tree-sitter 0.19.5
```

### 预编译的二进制文件
如果你的操作系统没有提供树形解析器的打包（或者提供了，但版本不是最新的），你可以从树形解析器在GitHub上的发布页下载预编译的二进制文件。

树形解析器的发布版本 [github.com](https://github.com/tree-sitter/tree-sitter/releases/latest)

‘tree-sitter’命令行工具是一个无依赖的静态二进制文件，因此你只需下载、解压并将其添加到你的$PATH环境变量即可：

```bash
$ curl -OL https://github.com/tree-sitter/tree-sitter/releases/download/v0.19.5/tree-sitter-linux-x64.gz
$ mkdir -p $HOME/bin
$ gunzip tree-sitter-linux-x64.gz > $HOME/bin/tree-sitter
$ chmod u+x $HOME/bin/tree-sitter
$ export PATH=$HOME/bin:$PATH
$ tree-sitter --version
tree-sitter 0.19.5 (8d8690538ef0029885c7ef1f163b0e32f256a5aa)
```

### NPM包安装方式
树形解析器的命令行工具也可以通过NPM的‘tree-sitter-cli’包安装：

```bash
$ npm install tree-sitter-cli
```

安装后，命令行工具将被放置在你的`node_modules`目录里，你可以使用‘npx’命令来运行它：

```bash
$ npx tree-sitter --version
tree-sitter 0.19.4 (6dd41e2e45f8b4a00fda21f28bc0ebc6b172ffed)
```
（当你需要编辑语法规则时，这种安装方式特别有用，因为它是将树形解析器作为持续集成（CI）构建过程的一部分安装到你的语法规则仓库中最便捷的方法。）

## 安装语言语法
现在，你应该已经安装好了‘tree-sitter’程序。但是，如果我们试图解析一些Python代码，会发现它无法正常工作。

```bash
$ tree-sitter --version
tree-sitter 0.19.5

$ cat example.py
import utils

def add_four(x):
    return x + 4

print(add_four(5))

$ tree-sitter parse example.py
No language found
```

这是因为tree-sitter默认并不包括任何语言的语法——毕竟，系统无法预知用户想要解析和分析哪种特定的编程语言！

因此，如果我们希望解析Python代码，我们需要安装tree-sitter的Python语法。‘tree-sitter’程序提供了一个便利的功能，它可以自动为你生成和编译语言解析器；你只需将相应语法的git仓库克隆到一个指定的位置即可。

首先，我们需要为命令行程序创建一个配置文件。该配置文件将指导‘tree-sitter’去哪里寻找你希望使用的语言语法。执行以下命令后：

```bash
$ tree-sitter init-config
```

‘tree-sitter’将在‘$HOME/.tree-sitter/config.json’为你创建一个新的配置文件。用你喜欢的编辑器打开这个文件，你会在文件顶部看到一个`parser_directories`节：

```bash
$ head -n 6 ~/.tree-sitter/config.json
{
  "parser-directories": [
    "/home/dcreager/github",
    "/home/dcreager/src",
    "/home/dcreager/source"
  ],
```

你可以自由选择用于存放语法定义的目录。‘tree-sitter’程序会认为这些目录下名称符合‘tree-sitter-[language]’模式的任何子目录都包含了一个语法定义，并将在每次启动时自动为这些语法生成和编译解析器。

    要让这一切工作，你还需要安装Node.js和C编译器（因为语法定义是用基于JavaScript的DSL编写的，生成的解析器则是用C语言实现的）。

考虑到这些，你需要将Python语法克隆到配置文件中指定的某个目录中。（如果你决定更改配置文件以使用不同的目录，请确保相应更改下面的命令。）

```bash
$ mkdir -p ~/src
$ cd ~/src
$ git clone https://github.com/tree-sitter/tree-sitter-python
```

## 解析代码
完成以上步骤后，使用‘tree-sitter parse’命令，现在应能够为我们的示例文件生成并显示出一个解析树：

```bash
$ tree-sitter parse example.py
(module [0, 0] - [6, 0]
  (import_statement [0, 0] - [0, 12]
    name: (dotted_name [0, 7] - [0, 12]
      (identifier [0, 7] - [0, 12])))
  (function_definition [2, 0] - [3, 16]
    name: (identifier [2, 4] - [2, 12])
    parameters: (parameters [2, 12] - [2, 15]
      (identifier [2, 13] - [2, 14]))
    body: (block [3, 4] - [3, 16]
      (return_statement [3, 4] - [3, 16]
        (binary_operator [3, 11] - [3, 16]
          left: (identifier [3, 11] - [3, 12])
          right: (integer [3, 15] - [3, 16])))))
  (expression_statement [5, 0] - [5, 18]
    (call [5, 0] - [5, 18]
      function: (identifier [5, 0] - [5, 5])
      arguments: (argument_list [5, 5] - [5, 18]
        (call [5, 6] - [5, 17]
          function: (identifier [5, 6] - [5, 14])
          arguments: (argument_list [5, 14] - [5, 17]
            (integer [5, 15] - [5, 16])))))))
```

你还可以尝试解析其他编程语言的示例文件，以进一步探索。首先，将需要的语言语法克隆到‘$HOME/src’目录下，然后运行‘tree-sitter parse’命令进行解析。
