# 简洁而强大的 Pratt 解析

*原文：[Simple but Powerful Pratt Parsing](https://matklad.github.io/2020/04/13/simple-but-powerful-pratt-parsing.html)*

欢迎阅读我关于 Pratt 解析的文章——这是语法分析领域的一堂高级教程。关于 Pratt 解析的文章汗牛充栋，甚至有专门的[综述文章](https://www.oilshell.org/blog/2017/03/31.html)存在 :)

本文旨在：

* 指出所谓左递归问题被过分强调了。
* 阐述BNF在表达中缀表达式时的不足。
* 介绍并实现一个忠于核心原理且不依赖DSL（领域特定语言）抽象的 Pratt 解析算法。
* 希望通过自我解释，这将是我最后一次尝试彻底理解这个算法。我之前曾[实现](https://github.com/rust-analyzer/rust-analyzer/blob/c388130f5ffbcbe7d3131213a24d12d02f769b87/crates/ra_parser/src/grammar/expressions.rs#L280-L281)过一个生产级别的 Pratt 解析器，但现在我已不再能立即理解那些代码。
本文预设读者对解析技术有一定的了解，不会去解释什么是上下文无关文法等基础概念。

## 简介
解析是编译器将一系列标记转换成树状结构表示的过程：

```plan/text
                            Add
                 Parser     / \
 "1 + 2 * 3"    ------->   1  Mul
                              / \
                             2   3
```

完成这一任务有许多不同的方法，大体上可以归纳为两个主要类别：

* 利用DSL（领域特定语言）来定义语言的抽象语法；
* 手写自定义解析器代码。

Pratt 解析是编写自定义解析器时经常采用的一种技术。

## BNF

在语法分析理论中的一个高峰是利用上下文无关文法（通常采用 BNF，即巴科斯范式的具体语法）来实现线性结构到树状结构的转换：

```BNF
Item ::=
    StructItem
  | EnumItem
  | ...
StructItem ::=
    'struct' Name '{' FieldList '}'
...
```

我记得我最初对这个理念非常着迷，特别是它与自然语言句结构之间的相似性。然而，当我们开始尝试描述表达式时，我的乐观情绪迅速消失。的确，自然的表达式语法允许我们明白什么构成了一个表达式。

```BNF
Expr ::=
    Expr '+' Expr
  | Expr '*' Expr
  | '(' Expr ')'
  | 'number'
```

尽管这套语法看起来非常优雅，实际上它却是含糊且不精确的，需要重新编写以适应自动化解析器的生成。特别是，我们需要明确运算符的优先级和结合律。经过修改的语法如下所示：

```BNF
Expr ::=
    Factor
  | Expr '+' Factor
Factor ::=
    Atom
  | Factor '*' Atom
Atom ::=
    'number'
  | '(' Expr ')'
```

在我看来，这种新的表述方式完全丢失了表达式的“形态”。更重要的是，在我经历了三到四门有关形式语言的课程之后，我才能够自信地构建出这样的语法。

这就是我为何钟爱 Pratt 解析的原因——它是递归下降解析算法的改进版，采用了直观的优先级和结合性概念来解析表达式，而非依赖于使语法变得模糊不清的技巧。

## 递归下降与左递归

手动编写解析器的一种基本技巧是递归下降，这种方法通过一组相互递归的函数来模拟语法结构。比如，前述的语法片段可以表现为如下形式：

```rust
fn item(p: &mut Parser) {
    match p.peek() {
        STRUCT_KEYWORD => struct_item(p),
        ENUM_KEYWORD   => enum_item(p),
        ...
    }
}

fn struct_item(p: &mut Parser) {
    p.expect(STRUCT_KEYWORD);
    name(p);
    p.expect(L_CURLY);
    field_list(p);
    p.expect(R_CURLY);
}
...
```

传统上，教科书常常将左递归语法视为这种方法的弱点，并利用这一缺陷来推广更高级的LR解析技术。一个引起问题的语法示例可能如下所示：

```BNF
Sum ::=
    Sum '+' Int
  | Int
```

实际上，如果我们直接编写求和函数，其效果并不理想：

```rust
fn sum(p: &mut Parser) {
    // Try first alternative
    sum(p); 
    p.expect(PLUS);
    int(p);
    // If that fails, try the second one
    ...
}
```
*此时，我们会立刻进入循环并导致栈溢出。*

理论上，解决这个问题需要重写语法以消除左递归。但在实践中，对于手写的解析器，有一个更简单的解决办法——摒弃纯递归的模式，转而使用循环：

```rust
fn sum(p: &mut Parser) {
    int(p);
    while p.eat(PLUS) {
        int(p);
    }
}
```

## Pratt解析的总体架构
仅仅依靠循环是不足以解析中缀表达式的。Pratt 解析结合了循环和递归的使用：

```rust
fn parse_expr() {
    ...
    loop {
        ...
        parse_expr()
        ...
    }
}
```

这不仅会让你感觉自己像在莫比乌斯带上奔跑的仓鼠，还能有效处理运算符的结合性和优先级！

## 从优先级到绑定力
我要坦白一件事：我对“高优先级”和“低优先级”的概念总是感到困惑。在表达式 a + b * c 中，加法虽然优先级较低，却位于解析树的顶端...

因此，我发现从绑定力这个角度思考更为直观。

```plan/text
expr:   A       +       B       *       C
power:      3       3       5       5
```

星号（*）更强大，它有更大的力量将 B 和 C 紧紧结合在一起，所以表达式被解析为 A + (B * C)。

那结合性如何处理呢？在 A + B + C 中，所有的运算符似乎都具有相同的力量，不明确首先应该合并哪个加号。但是，通过稍微调整力量的不对称性，这也能够被模型化：

```plan/text
expr:      A       +       B       +       C
power:  0      3      3.1      3      3.1     0
```

这里，我们略微提高了加号的右侧力量，使其更紧地绑定右侧操作数。我们还在两端加上零，因为没有运算符从旁边绑定。在这种情况下，第一个（也是唯一的第一个）加号比周围的更紧密地绑定了它的两个参数，因此我们可以先进行这一步的合并：

```plan/text
expr:     (A + B)     +     C
power:  0          3    3.1    0
```

现在我们可以合并第二个加号，得到 (A + B) + C。换句话说，就语法树而言，第二个加号更偏爱其右侧操作数，因此它急于与 C 结合。在这个过程中，第一个加号将 A 和 B 紧密捕获，因为它们没有争议。

Pratt 解析的独到之处在于，它通过从左到右处理字符串，寻找那些比周围运算符更强大的关键运算符。我们即将开始编写代码，但在此之前，让我们先看一个其他的示例。我们将使用点（.）作为具有高绑定力的右结合运算符来进行函数组合。也就是说，f . g . h 被解析为 f . (g . h)，或者说，从绑定力角度看

```plan/text
  f     .    g     .    h
0   8.5    8   8.5    8   0
```

## 最简化的 Pratt 解析器
我们即将解析的表达式中，基本元素为单个字符的数字和变量，并且使用标点作为运算符。首先定义一个简单的词法分析器：

```rust
#[derive(Debug, Clone, Copy, PartialEq, Eq)]
enum Token {
    Atom(char),
    Op(char),
    Eof,
}
struct Lexer {
    tokens: Vec<Token>,
}
impl Lexer {
    fn new(input: &str) -> Lexer {
        let mut tokens = input
            .chars()
            .filter(|it| !it.is_ascii_whitespace())
            .map(|c| match c {
                '0'..='9' |
                'a'..='z' | 'A'..='Z' => Token::Atom(c),
                _ => Token::Op(c),
            })
            .collect::<Vec<_>>();
        tokens.reverse();
        Lexer { tokens }
    }
    fn next(&mut self) -> Token {
        self.tokens.pop().unwrap_or(Token::Eof)
    }
    fn peek(&mut self) -> Token {
        self.tokens.last().copied().unwrap_or(Token::Eof)
    }
}
```

为确保我们正确理解优先级的绑定力，我们将中缀表达式转换成黄金标准（出于某种原因，在波兰不太受欢迎）的无歧义表示法——S表达式：  

```plan/text
1 + 2 * 3 == (+ 1 (* 2 3)).
```

```rust
use std::fmt;

enum S {
    Atom(char),
    Cons(char, Vec<S>),
}

impl fmt::Display for S {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        match self {
            S::Atom(i) => write!(f, "{}", i),
            S::Cons(head, rest) => {
                write!(f, "({}", head)?;
                for s in rest {
                    write!(f, " {}", s)?
                }
                write!(f, ")")
            }
        }
    }
}
```

我们从解析带有原子表达式和两个中缀二元运算符 + 和 * 的表达式开始：

```rust
fn expr(input: &str) -> S {
    let mut lexer = Lexer::new(input);
    expr_bp(&mut lexer)
}

fn expr_bp(lexer: &mut Lexer) -> S {
    todo!()
}

#[test]
fn tests() {
    let s = expr("1 + 2 * 3");
    assert_eq!(s.to_string(), "(+ 1 (* 2 3))")
}
```

这里与我们处理左递归的基本方法大致相同——从解析第一个数字开始，然后进入循环，消耗运算符并执行……某些操作？

```rust
fn expr_bp(lexer: &mut Lexer) -> S {
    let lhs = match lexer.next() {
        Token::Atom(it) => S::Atom(it),
        t => panic!("bad token: {:?}", t),
    };
    loop {
        let op = match lexer.next() {
            Token::Eof => break,
            Token::Op(op) => op,
            t => panic!("bad token: {:?}", t),
        };
        todo!()
    }
    lhs
}

#[test]
fn tests() {
    let s = expr("1"); // 1️⃣
    assert_eq!(s.to_string(), "1");
}
```
*1️⃣注意，我们已经能够解析这个简单的测试案例了！*

我们想要利用这种“力量”的理念，因此，让我们计算运算符的左右“力量”。我们将使用 u8 数据类型来表示“力量”，因此，对于结合性，我们会加1。我们将为输入结束保留“0 力量”，因此，运算符可以拥有的最低“力量”是1。

```rust
fn expr_bp(lexer: &mut Lexer) -> S {
    let lhs = match lexer.next() {
        Token::Atom(it) => S::Atom(it),
        t => panic!("bad token: {:?}", t),
    };
    loop {
        let op = match lexer.peek() {
            Token::Eof => break,
            Token::Op(op) => op,
            t => panic!("bad token: {:?}", t),
        };
        let (l_bp, r_bp) = infix_binding_power(op);
        todo!()
    }
    lhs
}
fn infix_binding_power(op: char) -> (u8, u8) {
    match op {
        '+' | '-' => (1, 2),
        '*' | '/' => (3, 4),
        _ => panic!("bad op: {:?}")
    }
}
```

接下来是引入递归的复杂部分。让我们思考下面这个例子（力量值如下所示）：

```plan/text
a   +   b   *   c   *   d   +   e
  1   2   3   4   3   4   1   2
```

当光标位于第一个加号（+）时，我们知道左侧的绑定力（bp）为1，右侧为2。左侧存有 a。加号（+）之后的下一个运算符是乘号（*），因此我们不应该简单地将 b 加到 a 上。问题在于，我们尚未遇到下一个运算符，我们仅仅是刚刚越过加号。我们能加入前瞻机制吗？似乎不行——我们必须越过所有的 b、c 和 d 才能找到下一个具有较低绑定力的运算符，这似乎没有界限。但我们找到了一线希望！我们当前的右侧优先级为2，为了能够合并表达式，我们需要找到下一个优先级更低的运算符。因此，让我们递归调用 expr_bp 函数，从 b 开始，但同时告诉它一旦绑定力下降到2以下就停止。这就需要在主函数中添加 min_bp 参数。

看，我们得到了一个全功能的最简化 Pratt 解析器：

```rust
fn expr(input: &str) -> S {
    let mut lexer = Lexer::new(input);
    expr_bp(&mut lexer, 0) // 5️⃣
}

fn expr_bp(lexer: &mut Lexer, min_bp: u8) -> S { // 1️⃣
    let mut lhs = match lexer.next() {
        Token::Atom(it) => S::Atom(it),
        t => panic!("bad token: {:?}", t),
    };
    loop {
        let op = match lexer.peek() {
            Token::Eof => break,
            Token::Op(op) => op,
            t => panic!("bad token: {:?}", t),
        };
        let (l_bp, r_bp) = infix_binding_power(op);
        if l_bp < min_bp { // 2️⃣
            break;
        }
        lexer.next(); // 3️⃣ 
        let rhs = expr_bp(lexer, r_bp);
        lhs = S::Cons(op, vec![lhs, rhs]); // 4️⃣
    }
    lhs
}

fn infix_binding_power(op: char) -> (u8, u8) {
    match op {
        '+' | '-' => (1, 2),
        '*' | '/' => (3, 4),
        _ => panic!("bad op: {:?}"),
    }
}

#[test]
fn tests() {
    let s = expr("1");
    assert_eq!(s.to_string(), "1");
    let s = expr("1 + 2 * 3");
    assert_eq!(s.to_string(), "(+ 1 (* 2 3))");
    let s = expr("a + b * c * d + e");
    assert_eq!(s.to_string(), "(+ (+ a (* (* b c) d)) e)");
}
```
*1️⃣min_bp 参数是一个关键的补充。expr_bp 现在能够解析具有相对较高绑定力的表达式。一旦遇到比 min_bp 弱的元素，它就会停止。
2️⃣这就是“停止”的点。
3️⃣在这里，我们跨过运算符本身并进行递归调用。注意我们是如何用 l_bp 对 min_bp 进行检查，并将 r_bp 作为新的 min_bp 用于递归调用。因此，可以把 min_bp 看作是当前表达式左侧运算符的绑定力。
4️⃣最后，解析了正确的右侧表达式后，我们构建了新的当前表达式。
5️⃣启动递归时，我们使用绑定力为零。记住，在开始时，左侧运算符的绑定力是最低的，即零，因为那里实际上没有运算符。*

因此，这40行代码构成了 Pratt 解析算法。它们可能有些复杂，但如果你理解了它们，剩下的工作就是直接的补充。

## 附加功能
现在，我们将添加各种复杂的表达式来展示这个算法的强大和灵活性。首先，我们加入一个高优先级的右结合函数组合运算符：.：

```rust
fn infix_binding_power(op: char) -> (u8, u8) { 
    match op {
        '+' | '-' => (1, 2),
        '*' | '/' => (3, 4),
        '.' => (6, 5),
        _ => panic!("bad op: {:?}"),
    }
}
```

没错，只需要一行代码！注意运算符左侧的绑定力更强，这就实现了我们想要的右结合性：

```rust
let s = expr("f . g . h");
assert_eq!(s.to_string(), "(. f (. g h))");
let s = expr(" 1 + 2 + f . g . h * 3 * 4");
assert_eq!(s.to_string(), "(+ (+ 1 2) (* (* (. f (. g h)) 3) 4))");
```

接下来，我们引入一元运算符 -，其绑定力比二元算术运算符要强，但比组合运算符要弱。这就需要我们改变循环的起始方式，因为我们不再能假设第一个标记是一个原子，还需要处理减号。但是，让我们依据类型来进行。首先，我们定义绑定力。由于这是一个一元运算符，它实际上只有右侧绑定力，那么，我们就直接编码实现它：

```rust
fn prefix_binding_power(op: char) -> ((), u8) { // 1️⃣
    match op {
        '+' | '-' => ((), 5),
        _ => panic!("bad op: {:?}", op),
    }
}
fn infix_binding_power(op: char) -> (u8, u8) {
    match op {
        '+' | '-' => (1, 2),
        '*' | '/' => (3, 4),
        '.' => (8, 7), // 2️⃣
        _ => panic!("bad op: {:?}"),
    }
}
```
*1️⃣这里，我们返回一个虚拟的 () 来明确表示这是一个前缀运算符而不是后缀运算符，因此它只能绑定到右侧的元素。
2️⃣需要注意的是，因为我们想在点（.）和星号（*）之间加入一元减号（-），我们需要将点（.）的优先级上调两位。一般规则是，我们使用奇数优先级作为基准，并且如果运算符是二元的，就因结合性而增加一位。对于一元减号而言，使用 5 或 6 都可以，但保持使用奇数更为一致。*

把这个逻辑加入到 expr_bp 函数中，我们得到了：

```rust
fn expr_bp(lexer: &mut Lexer, min_bp: u8) -> S {
    let mut lhs = match lexer.next() {
        Token::Atom(it) => S::Atom(it),
        Token::Op(op) => {
            let ((), r_bp) = prefix_binding_power(op);
            todo!()
        }
        t => panic!("bad token: {:?}", t),
    };
    ...
}
```

现在，我们只有右侧绑定力（r_bp）而没有左侧绑定力（l_bp），那我们是不是应该直接复制粘贴主循环的一部分代码？记得，我们在递归调用中使用 r_bp。

```rust
fn expr_bp(lexer: &mut Lexer, min_bp: u8) -> S {
    let mut lhs = match lexer.next() {
        Token::Atom(it) => S::Atom(it),
        Token::Op(op) => {
            let ((), r_bp) = prefix_binding_power(op);
            let rhs = expr_bp(lexer, r_bp);
            S::Cons(op, vec![rhs])
        }
        t => panic!("bad token: {:?}", t),
    };
    loop {
        let op = match lexer.peek() {
            Token::Eof => break,
            Token::Op(op) => op,
            t => panic!("bad token: {:?}", t),
        };
        let (l_bp, r_bp) = infix_binding_power(op);
        if l_bp < min_bp {
            break;
        }
        lexer.next();
        let rhs = expr_bp(lexer, r_bp);
        lhs = S::Cons(op, vec![lhs, rhs]);
    }
    lhs
}

#[test]
fn tests() {
    ...
    let s = expr("--1 * 2");
    assert_eq!(s.to_string(), "(* (- (- 1)) 2)");
    let s = expr("--f . g");
    assert_eq!(s.to_string(), "(- (- (. f g)))");
}
```

有趣的是，这种机械的、由类型驱动的转换实际上是有效的。你当然也可以推理出它为何有效。同样的论据适用：在我们处理完一个前缀运算符之后，其操作数由绑定力更强的运算符组成，而我们正好有一个函数能够解析比指定绑定力更强的表达式。

这开始变得有些不合逻辑了。如果使用 ((), u8) 对前缀运算符“刚好有效”，那么 (u8, ()) 能否处理后缀运算符呢？我们加入了 ! 来表示阶乘，它应该比减号（-）绑定力更强，因为 -(92!) 显然比 (-92)! 更有意义。所以，按照惯例——我们需要添加一个新的优先级函数，并且需要调整点（.）的优先级（这在 Pratt 解析器中确实有点麻烦），然后复制粘贴代码……

```rust
let (l_bp, ()) = postfix_binding_power(op);
if l_bp < min_bp {
    break;
}
let (l_bp, r_bp) = infix_binding_power(op);
if l_bp < min_bp {
    break;
}
```

等一下，这里出现了问题。在我们解析完前缀表达式后，我们可能遇到一个后缀运算符或一个中缀运算符。但如果我们在遇到无法识别的运算符时就停下来，这个方法就不会奏效……因此，让我们修改 postfix_binding_power 函数，使其在运算符不是后缀时返回一个选项：

```rust
fn expr_bp(lexer: &mut Lexer, min_bp: u8) -> S {
    let mut lhs = match lexer.next() {
        Token::Atom(it) => S::Atom(it),
        Token::Op(op) => {
            let ((), r_bp) = prefix_binding_power(op);
            let rhs = expr_bp(lexer, r_bp);
            S::Cons(op, vec![rhs])
        }
        t => panic!("bad token: {:?}", t),
    };
    loop {
        let op = match lexer.peek() {
            Token::Eof => break,
            Token::Op(op) => op,
            t => panic!("bad token: {:?}", t),
        };
        if let Some((l_bp, ())) = postfix_binding_power(op) {
            if l_bp < min_bp {
                break;
            }
            lexer.next();
            lhs = S::Cons(op, vec![lhs]);
            continue;
        }
        let (l_bp, r_bp) = infix_binding_power(op);
        if l_bp < min_bp {
            break;
        }
        lexer.next();
        let rhs = expr_bp(lexer, r_bp);
        lhs = S::Cons(op, vec![lhs, rhs]);
    }
    lhs
}

fn prefix_binding_power(op: char) -> ((), u8) {
    match op {
        '+' | '-' => ((), 5),
        _ => panic!("bad op: {:?}", op),
    }
}

fn postfix_binding_power(op: char) -> Option<(u8, ())> {
    let res = match op {
        '!' => (7, ()),
        _ => return None,
    };
    Some(res)
}

fn infix_binding_power(op: char) -> (u8, u8) {
    match op {
        '+' | '-' => (1, 2),
        '*' | '/' => (3, 4),
        '.' => (10, 9),
        _ => panic!("bad op: {:?}"),
    }
}

#[test]
fn tests() {
    let s = expr("-9!");
    assert_eq!(s.to_string(), "(- (! 9))");
    let s = expr("f . g !");
    assert_eq!(s.to_string(), "(! (. f g))");
}
```

有趣的是，新的和旧的测试都顺利通过了。

现在，我们准备引入一种新的表达式类型：带括号的表达式。实际上这并不复杂，我们本来可以从一开始就处理它，但在这里介绍它更合适，你马上就会明白原因。括号实质上是一种基本表达式，处理方式与原子相似：

```rust
let mut lhs = match lexer.next() {
    Token::Atom(it) => S::Atom(it),
    Token::Op('(') => {
        let lhs = expr_bp(lexer, 0);
        assert_eq!(lexer.next(), Token::Op(')'));
        lhs
    }
    Token::Op(op) => {
        let ((), r_bp) = prefix_binding_power(op);
        let rhs = expr_bp(lexer, r_bp);
        S::Cons(op, vec![rhs])
    }
    t => panic!("bad token: {:?}", t),
};
```

遗憾的是，下面的测试没有通过：

```rust
let s = expr("(((0)))");
assert_eq!(s.to_string(), "0");
```

问题出现在下面的循环中——我们唯一的终止条件是到达文件结束符，而`)`显然不是文件结束符。修正这个问题最简单的方法是让`infix_binding_power`在遇到无法识别的操作数时返回`None`。这样，它又一次变得类似于`postfix_binding_power`了！

```rust
fn expr_bp(lexer: &mut Lexer, min_bp: u8) -> S {
    let mut lhs = match lexer.next() {
        Token::Atom(it) => S::Atom(it),
        Token::Op('(') => {
            let lhs = expr_bp(lexer, 0);
            assert_eq!(lexer.next(), Token::Op(')'));
            lhs
        }
        Token::Op(op) => {
            let ((), r_bp) = prefix_binding_power(op);
            let rhs = expr_bp(lexer, r_bp);
            S::Cons(op, vec![rhs])
        }
        t => panic!("bad token: {:?}", t),
    };
    loop {
        let op = match lexer.peek() {
            Token::Eof => break,
            Token::Op(op) => op,
            t => panic!("bad token: {:?}", t),
        };
        if let Some((l_bp, ())) = postfix_binding_power(op) {
            if l_bp < min_bp {
                break;
            }
            lexer.next();
            lhs = S::Cons(op, vec![lhs]);
            continue;
        }
        if let Some((l_bp, r_bp)) = infix_binding_power(op) {
            if l_bp < min_bp {
                break;
            }
            lexer.next();
            let rhs = expr_bp(lexer, r_bp);
            lhs = S::Cons(op, vec![lhs, rhs]);
            continue;
        }
        break;
    }
    lhs
}

fn prefix_binding_power(op: char) -> ((), u8) {
    match op {
        '+' | '-' => ((), 5),
        _ => panic!("bad op: {:?}", op),
    }
}

fn postfix_binding_power(op: char) -> Option<(u8, ())> {
    let res = match op {
        '!' => (7, ()),
        _ => return None,
    };
    Some(res)
}

fn infix_binding_power(op: char) -> Option<(u8, u8)> {
    let res = match op {
        '+' | '-' => (1, 2),
        '*' | '/' => (3, 4),
        '.' => (10, 9),
        _ => return None,
    };
    Some(res)
}
```

现在，让我们加入数组索引操作符：`a[i]`。它属于哪种“-fix”？环绕“-fix”吗？如果只有`a[]`，它显然是后缀操作。如果只有`[i]`，它就完全像括号一样工作。关键在于：`i`部分实际上并不参与整个优先级游戏，因为它是明确分隔的。那么，让我们这样做：

```rust
fn expr_bp(lexer: &mut Lexer, min_bp: u8) -> S {
    let mut lhs = match lexer.next() {
        Token::Atom(it) => S::Atom(it),
        Token::Op('(') => {
            let lhs = expr_bp(lexer, 0);
            assert_eq!(lexer.next(), Token::Op(')'));
            lhs
        }
        Token::Op(op) => {
            let ((), r_bp) = prefix_binding_power(op);
            let rhs = expr_bp(lexer, r_bp);
            S::Cons(op, vec![rhs])
        }
        t => panic!("bad token: {:?}", t),
    };
    loop {
        let op = match lexer.peek() {
            Token::Eof => break,
            Token::Op(op) => op,
            t => panic!("bad token: {:?}", t),
        };
        if let Some((l_bp, ())) = postfix_binding_power(op) {
            if l_bp < min_bp {
                break;
            }
            lexer.next();
            lhs = if op == '[' {
                let rhs = expr_bp(lexer, 0);
                assert_eq!(lexer.next(), Token::Op(']'));
                S::Cons(op, vec![lhs, rhs])
            } else {
                S::Cons(op, vec![lhs])
            };
            continue;
        }
        if let Some((l_bp, r_bp)) = infix_binding_power(op) {
            if l_bp < min_bp {
                break;
            }
            lexer.next();
            let rhs = expr_bp(lexer, r_bp);
            lhs = S::Cons(op, vec![lhs, rhs]);
            continue;
        }
        break;
    }
    lhs
}
fn prefix_binding_power(op: char) -> ((), u8) {
    match op {
        '+' | '-' => ((), 5),
        _ => panic!("bad op: {:?}", op),
    }
}
fn postfix_binding_power(op: char) -> Option<(u8, ())> {
    let res = match op {
        '!' | '[' => (7, ()), // 1️⃣
        _ => return None,
    };
    Some(res)
}
fn infix_binding_power(op: char) -> Option<(u8, u8)> {
    let res = match op {
        '+' | '-' => (1, 2),
        '*' | '/' => (3, 4),
        '.' => (10, 9),
        _ => return None,
    };
    Some(res)
}
#[test]
fn tests() {
    ...
    let s = expr("x[0][1]");
    assert_eq!(s.to_string(), "([ ([ x 0) 1)");
}
```
*1️⃣注意我们为`!`和`[`使用了相同的优先级。一般来说，为了算法的正确性，非常重要的一点是，当我们做出决策时，优先级永远不应该相等。否则，我们可能会陷入之前为了解决结合性问题而做的微小调整相似的情形，那时有两个 equally-good 的缩减候选。然而，我们只比较右侧绑定力与左侧绑定力！因此，对于两个后缀操作符来说，拥有相同的优先级是可以接受的，因为它们都是右侧的。*

最终，所有运算符中最为复杂的，那个令人敬畏的三元运算符：

```plan/text
c ? e1 : e2
```

这难道是……全方位-fix运算符吗？好吧，让我们对三元运算符的语法进行微调：

```plan/text
c [ e1 ] e2
```

回想一下，a[i]实际上被解释为后缀运算符加上括号……因此，没错，? 和 : 实际上是一对非常特别的括号！我们就这样处理它吧！那么，关于优先级和结合性呢？在这种情况下，结合性甚至意味着什么？

```plan/text
a ? b : c ? d : e
```

要搞清楚，我们只需要简化括号部分：

```plan/text
a ?: c ?: e
```

这可以被解析为

```plan/text
(a ?: c) ?: e
```

或者

```plan/text
a ?: (c ?: e)
```

哪一个更有用？对于这样的 ?-链：

```plan/text
a ? b :
c ? d :
e
```

右结合性的解释更有用。从优先级角度看，三元运算符的优先级较低。在C语言中，只有=和,的优先级更低。既然我们已经提及，我们也加入C风格的右结合 = 运算符。

这是我们最完整、最完美的简易 Pratt 解析器版本：

```rust
use std::{fmt, io::BufRead};
enum S {
    Atom(char),
    Cons(char, Vec<S>),
}
impl fmt::Display for S {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        match self {
            S::Atom(i) => write!(f, "{}", i),
            S::Cons(head, rest) => {
                write!(f, "({}", head)?;
                for s in rest {
                    write!(f, " {}", s)?
                }
                write!(f, ")")
            }
        }
    }
}
#[derive(Debug, Clone, Copy, PartialEq, Eq)]
enum Token {
    Atom(char),
    Op(char),
    Eof,
}
struct Lexer {
    tokens: Vec<Token>,
}
impl Lexer {
    fn new(input: &str) -> Lexer {
        let mut tokens = input
            .chars()
            .filter(|it| !it.is_ascii_whitespace())
            .map(|c| match c {
                '0'..='9'
                | 'a'..='z' | 'A'..='Z' => Token::Atom(c),
                _ => Token::Op(c),
            })
            .collect::<Vec<_>>();
        tokens.reverse();
        Lexer { tokens }
    }
    fn next(&mut self) -> Token {
        self.tokens.pop().unwrap_or(Token::Eof)
    }
    fn peek(&mut self) -> Token {
        self.tokens.last().copied().unwrap_or(Token::Eof)
    }
}
fn expr(input: &str) -> S {
    let mut lexer = Lexer::new(input);
    expr_bp(&mut lexer, 0)
}
fn expr_bp(lexer: &mut Lexer, min_bp: u8) -> S {
    let mut lhs = match lexer.next() {
        Token::Atom(it) => S::Atom(it),
        Token::Op('(') => {
            let lhs = expr_bp(lexer, 0);
            assert_eq!(lexer.next(), Token::Op(')'));
            lhs
        }
        Token::Op(op) => {
            let ((), r_bp) = prefix_binding_power(op);
            let rhs = expr_bp(lexer, r_bp);
            S::Cons(op, vec![rhs])
        }
        t => panic!("bad token: {:?}", t),
    };
    loop {
        let op = match lexer.peek() {
            Token::Eof => break,
            Token::Op(op) => op,
            t => panic!("bad token: {:?}", t),
        };
        if let Some((l_bp, ())) = postfix_binding_power(op) {
            if l_bp < min_bp {
                break;
            }
            lexer.next();
            lhs = if op == '[' {
                let rhs = expr_bp(lexer, 0);
                assert_eq!(lexer.next(), Token::Op(']'));
                S::Cons(op, vec![lhs, rhs])
            } else {
                S::Cons(op, vec![lhs])
            };
            continue;
        }
        if let Some((l_bp, r_bp)) = infix_binding_power(op) {
            if l_bp < min_bp {
                break;
            }
            lexer.next();
            lhs = if op == '?' {
                let mhs = expr_bp(lexer, 0);
                assert_eq!(lexer.next(), Token::Op(':'));
                let rhs = expr_bp(lexer, r_bp);
                S::Cons(op, vec![lhs, mhs, rhs])
            } else {
                let rhs = expr_bp(lexer, r_bp);
                S::Cons(op, vec![lhs, rhs])
            };
            continue;
        }
        break;
    }
    lhs
}
fn prefix_binding_power(op: char) -> ((), u8) {
    match op {
        '+' | '-' => ((), 9),
        _ => panic!("bad op: {:?}", op),
    }
}
fn postfix_binding_power(op: char) -> Option<(u8, ())> {
    let res = match op {
        '!' => (11, ()),
        '[' => (11, ()),
        _ => return None,
    };
    Some(res)
}
fn infix_binding_power(op: char) -> Option<(u8, u8)> {
    let res = match op {
        '=' => (2, 1),
        '?' => (4, 3),
        '+' | '-' => (5, 6),
        '*' | '/' => (7, 8),
        '.' => (14, 13),
        _ => return None,
    };
    Some(res)
}
#[test]
fn tests() {
    let s = expr("1");
    assert_eq!(s.to_string(), "1");
    let s = expr("1 + 2 * 3");
    assert_eq!(s.to_string(), "(+ 1 (* 2 3))");
    let s = expr("a + b * c * d + e");
    assert_eq!(s.to_string(), "(+ (+ a (* (* b c) d)) e)");
    let s = expr("f . g . h");
    assert_eq!(s.to_string(), "(. f (. g h))");
    let s = expr(" 1 + 2 + f . g . h * 3 * 4");
    assert_eq!(
        s.to_string(),
        "(+ (+ 1 2) (* (* (. f (. g h)) 3) 4))",
    );
    let s = expr("--1 * 2");
    assert_eq!(s.to_string(), "(* (- (- 1)) 2)");
    let s = expr("--f . g");
    assert_eq!(s.to_string(), "(- (- (. f g)))");
    let s = expr("-9!");
    assert_eq!(s.to_string(), "(- (! 9))");
    let s = expr("f . g !");
    assert_eq!(s.to_string(), "(! (. f g))");
    let s = expr("(((0)))");
    assert_eq!(s.to_string(), "0");
    let s = expr("x[0][1]");
    assert_eq!(s.to_string(), "([ ([ x 0) 1)");
    let s = expr(
        "a ? b :
         c ? d
         : e",
    );
    assert_eq!(s.to_string(), "(? a b (? c d e))");
    let s = expr("a = 0 ? b : c = d");
    assert_eq!(s.to_string(), "(= a (= (? 0 b c) d))")
}
fn main() {
    for line in std::io::stdin().lock().lines() {
        let line = line.unwrap();
        let s = expr(&line);
        println!("{}", s)
    }
}
```

代码也可以在这个[仓库](https://github.com/matklad/minipratt)中找到，Eof :-)
