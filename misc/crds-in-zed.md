# CRDTs如何成为Zed多人文本编辑核心特征

* 原文:[How CRDTs make multiplayer text editing part of Zed's DNA](https://zed.dev/blog/crdts)

![](https://zed.dev/img/post/crdts/preview.png)

对程序员而言，文本编辑器是他们每天都要面对的基础工具。在代码最终成型、提交或发布之前，每一行代码的敲打都离不开键盘和屏幕。编辑器在软件创造的过程中，对于提供直观和触感上的体验影响深远。

尽管编辑器在我的程序员生涯中占据了不可或缺的地位，但我始终未能找到一个令我完全满意的编辑器。于是，16年前，我下定决心自己来创建一个。这个过程充满挑战，经历了一次失败尝试、众多宝贵教训以及得到了许多有才华的朋友的协助，如今，我心中理想的工具终于通过135k行Rust代码，逐渐成为了Zed。

我们对Zed的首要目标很简明：打造一个我们自己也愿意使用的编辑器。这个编辑器要性能卓越，使用流畅；既能有效地辅助我们工作，又能在不需要时自我隐藏；外观吸引人，同时又能淡入背景。我们希望Zed能够在文本编辑领域推陈出新，汲取其他编辑器的长处，避免其短板，并且做到更好。任何不达标的设想都不足以付诸实施。

除了打磨基础功能外，我们也希望从根本上改善开发者的协作方式。通过把协作设计为代码编写环境的核心部分，Zed让任何文本的讨论变得更加便捷，无论这段代码是去年还是刚刚完成的。Zed还将使得实时与其他开发者编写和讨论代码成为一种流畅的体验。

对协作的这种深度整合是Zed的一大特色。因此，在我们的首篇博客中，我们想深入探讨那些被融入到编辑器核心、让这一切成为可能的技术。

## 协作编辑技术的早期历史
1968年12月，道格拉斯·恩格尔巴特在旧金山一次座无虚席的演讲中展示了一系列革命性技术，包括交互式编辑、超文本和鼠标。他的这些创意深刻地影响了现代计算的发展。然而，当我首次观看他那场著名的演示时，意外地了解到，那个震惊所有人的系统，竟然是[一款协作文本编辑器](https://youtu.be/8UQyQ7Gvi4U?t=286)——正是我一直梦想打造的工具，在1968年就已存在。

![](https://user-images.githubusercontent.com/1789/199089574-148fdec3-7b8c-476b-b29f-7d398887907d.png)
*比尔·帕克斯顿与道格拉斯·恩格尔巴特通过视频聊天，在互动计算的初期，共同编辑文本。*

恩格尔巴特和他的团队为了打造这款协作编辑器，不得不自己开发编程语言、分时操作系统和阴极射线管显示屏。与他们的挑战相比，我们构建Zed的过程在几乎所有方面都轻松许多。然而，我们面临着一个他们未曾遇到的问题：异步协作。

在恩格尔巴特的系统中，所有协作者通过个别终端连接至同一台机器。虽然我不确定他们的工具是否支持精细的并行编辑，但理论上，通过互斥锁机制同步共享缓冲区的编辑在那样的系统设置下是可行的。但现代计算机的工作方式已经完全不同。我们不再是通过直连终端共享单台机器，而是通过互联网连接的个人电脑在全球范围内协作。哪怕是光速，也无法避免在跨大洋合作时因共享缓冲区的同步访问而产生的编辑延迟。

## 异步协调面临的挑战
在互联网上进行协作要求我们找到一种方式，使得每个人可以独立地在自己的文档副本上进行编辑，并且当他们异步交换数据之后，能够使各自的文档内容趋于一致。这实际上是一个相当复杂的问题。

下方的动画演示了这一挑战的本质。我们从两个文本副本“In 1968,”开始。接着，我们在每个副本中并行插入不同的文本，并向对方副本传输我们编辑操作的描述。但如果我们在不考虑并发更改的情况下直接应用远程编辑，就可能会把它应用到一个错误的位置，导致两个副本的内容发生偏离。

[视频](https://zed.dev/img/post/crdts/divergence.webm)  
*在并发操作的环境下，简单地复制操作会导致内容分歧。*

## 利用CRDTs实现最终一致的文本编辑
我们可以通过转换传入的编辑来反应并行发生的更改作为一种解决方案。在下方的动画展示中，可以看到我们是如何调整蓝色插入操作的位置，从8号位改到20号位。

[视频](https://zed.dev/img/post/crdts/ot.webm)  
操作转换技术的核心在于对传入的操作进行调整，以适应并发发生的编辑操作。

虽然这个概念听起来简单，但要定义一个既正确又高效的操作转换函数却非常复杂，这成为了计算机科学研究中被称为操作转换（OT）的一个专门分支的研究主题。我们在2017年初次探索协作编辑时试验了这种方法，但最终决定采用一个称为冲突无自由复制数据类型（CRDTs）的不同理论框架，我们认为它更加强大和直观。

采用CRDTs，我们不需要转换并发操作以便它们能以不同的顺序被应用，而是设计数据结构使得并发操作天然地具有可交换性，这使得我们能够直接在任何副本上应用这些操作，无需进行转换。但如何实现文本编辑的可交换性呢？

关键在于用逻辑位置而不是绝对偏移来表达编辑操作。如果我们不是用数字偏移量来标示插入位置，而是依据内容来描述，会怎样呢？这样并发编辑导致的文本移动就不再是问题，因为我们依赖的是内容来定位远程编辑的位置。

[视频](https://zed.dev/img/post/crdts/content-based-addresses.webm)  
*如果我们能够根据内容来定位编辑的位置，就可以直接应用操作，而无需进行转换。*

然而，这种方法在实际中并不可行。比如文本“68,”可能在多处出现，或者并发编辑可能已经将其完全删除。为了采用这种基于内容的逻辑定位方式，我们需要找到一种即使在并发更改的情况下也能保持稳定的方法。但具体怎么做呢？

## 在变动文本中创建稳定的引用

表达逻辑位置时依据缓冲区当前内容的问题在于文本本身的不稳定性。但有一样东西是稳定的：编辑历史。我们可以认为，任何一次插入的文本都是不可改变的。尽管后续的编辑可能会将这些文本分割或部分删除，但这并不改变最初插入的文本本身。通过为每一次插入赋予一个独一无二的标识符，我们就能够利用这个标识符和插入文本中的偏移量，明确无误地指向一个逻辑位置。我们把这样的（插入ID，偏移量）组合称作锚点。

为了创建这些独特的标识符，我们在每个副本创建时中心化地分配一个唯一的ID，再将其与一个递增的序列号结合。这样，利用副本ID的唯一性，各个副本可以同时生成ID，而不会发生冲突。

[视频](https://zed.dev/img/post/crdts/id-distribution.webm)  
*在中心化分配副本ID之后，各个副本就可以独立地生成唯一的ID了。*

在协作会话开始时，参与者被分配了副本ID。副本0将0.0作为缓冲区初始文本的标识符，然后把这个副本传给了副本1。这个初始文本片段0.0，作为第一次插入，将在缓冲区的整个生命周期中保持不变。

[视频](https://zed.dev/img/post/crdts/crdt-start-collaborating.webm)  
*缓冲区的起始文本总是由主机分配ID 0.0。主机即副本0，它会将缓冲区的一个副本发送给加入的协作者。*

此时，每位参与者都在并行地插入文本，并以相对于初始插入0.0的偏移量来描述插入位置。每一次新的插入都会被赋予一个独一无二的ID。当副本0在插入0.0的偏移量3处插入“December of”时，标记为0.0的文本片段被分割为两部分。副本1在插入0.0的偏移量8处添加了“Douglas Engelbart”。两位参与者也会把他们的操作传递给对方。

[视频](https://zed.dev/img/post/crdts/crdt-concurrent-insertion-part-1.webm)  
*插入操作被赋予了唯一的ID，并且它们的位置是相对于一个已存在的插入位置描述的，这里指的是初始插入0.0。*

现在，副本开始应用彼此的操作。首先，副本1整合了ID为0.1的红色插入操作，就像副本0原始插入这段文本时那样，将插入0.0分割成两部分。随后，副本0整合了ID为1.0的蓝色插入操作。

[视频](https://zed.dev/img/post/crdts/crdt-concurrent-insertion-part-2.webm)  
*为了应用远端操作，我们需要在本地文档中查找包含父插入指定偏移的片段。*

这个过程包括遍历文档的各个片段，寻找属于插入0.0且偏移量为8的部分。第一个遇到的片段属于0.0，但长度仅为3个字符。第二个片段属于另一个插入，编号为0.1，因此被略过。最后，我们找到了第二个含有插入0.0文本的片段，该片段包含了偏移量8，于是我们在此处插入蓝色文本。通过这种方式，各副本的内容开始趋于一致。

这一过程可以递归地进行，随着一个插入基于另一个构建，形成一种树状结构。在下面的动画中，两个副本在id为1.0的蓝色插入文本的不同偏移位置插入了额外的文本。要应用远程操作，我们再次遍历文档，寻找包含特定偏移的插入1.0的片段。

[视频](https://zed.dev/img/post/crdts/crdt-concurrent-insertion-part-3.webm)  
*先前的插入可以成为新插入的基础。*

虽然我们的例子中一次插入了多个字符，但实际上，协作者们往往是逐个字符进行插入，而非粘贴整个词组。虽然跟踪每个字符的所有这些元数据似乎是巨大的负担，但在现代计算设备上，这实际上并不构成问题。即使是庞大的编辑历史，与Zed因不采用Electron而获得的内存节约相比，也显得微不足道。

你可能会问，像这样遍历整个文档以应用每个远程编辑的操作速度不会非常慢吗？我将在未来的文章中解释我们如何利用写时复制的B树来索引这些片段，从而避免进行线性扫描，但这个简化的说明应该能帮你构建一个基础框架，以理解Zed中协作编辑的工作原理。

## 删除操作
如果每次插入都是永久不变的，那么用户在需要删除文本时，我们如何能够从文档中移除文本呢？我们选择的解决方案并非修改已插入的文本，而是通过墓碑符号来标记被删除的片段。这些带墓碑的片段在展示给用户的文本中被隐藏，但仍可用于解析逻辑锚点到文档中的确切位置。

如下动画所示，我们在副本1中插入文本的同时，在副本0中该文本位置被删除。由于被删除的文本实际上并未被彻底抛弃，而是仅仅被隐藏，因此当插入操作抵达副本0时，我们依旧可以执行该插入。

[视频](https://zed.dev/img/post/crdts/insert-delete.webm)  
*被删除的片段通过墓碑符号被隐藏。*

如果删除操作仅定义了一个区间，而在该删除区间内同时有文本被插入，就可能出现内容分歧。在下方的例子中，可以看到黄色的“C.”在副本0中是可见的，而在副本1中则被隐藏了。

[视频](https://zed.dev/img/post/crdts/delete-divergence.webm)  
*如果我们不明确指出在执行删除操作时哪些操作是可见的，那么在并发删除的范围内插入文本将引起内容分歧。*

为了避免这种情况，我们也将删除操作与一个向量时间戳关联起来，这个时间戳记录了每个副本的最新序列号。通过这种方式，我们可以排除同时发生的插入操作，仅仅隐藏执行删除操作的用户实际上能看到的文本。

下面的动画和之前的很相似，不同之处在于我们这次增加了一个版本向量来增强删除操作。当我们在副本1应用删除操作时，我们排除了黄色的插入，因为它的ID包含一个未被删除版本包括的序列号。这使得黄色插入在两个副本上都保持可见，维护了执行删除操作用户的原意。

[视频](https://zed.dev/img/post/crdts/delete-convergence.webm)  
*将删除操作与版本向量关联，使我们能够排除同时发生的插入操作，避免它们被错误地标记为删除。*

与插入操作一样，删除操作也关联着独一无二的标识符，这些标识符被记录在墓碑上。我们将在后续讨论撤销和重做操作时，探讨这些删除标识符的使用方式。

## 在同一位置同时进行的插入操作
当多个插入操作同时发生在同一位置时，虽然这些插入的顺序如何并不重要，但保证所有副本对这些插入的排序保持一致是至关重要的。一种确保排序一致性的方法是根据它们的ID对同一位置的所有插入进行排序。

[视频](https://zed.dev/img/post/crdts/order-insertions-by-id.webm)  
*根据ID对同一位置的插入进行排序，可以在所有副本上实现一致的顺序。*

但是，这种方法存在一个问题：某些副本在已经观察到某个插入后，就无法在该插入之前加入文本。

[视频](https://zed.dev/img/post/crdts/order-by-id-intention-violation.webm)  
*根据ID对同一位置的插入进行排序，并没有保留用户希望在已有插入之前进行插入的意图。*

我们需要一个能够尊重因果关系的并发插入的一致排序方法。我们采用的解决方案是为插入增加Lamport时间戳。这些逻辑时间戳基于每个副本上维护的Lamport时钟，该时钟是标量值。每当一个副本生成操作时，它通过增加自己的Lamport时钟来得到一个Lamport时间戳。每当一个副本接收到一个操作时，它会将自己的Lamport时钟调整到当前值和接收到的操作的时间戳中较大的一个。

[视频](https://zed.dev/img/post/crdts/lamport-clock.webm)  
*如果一个操作是在另一个操作已经被观察到之后产生的，那么它一定会被赋予一个更高的Lamport时间戳。*

这种安排确保了，如果一个操作在另一个操作发生时已经存在，它就会被赋予一个较低的时间戳。换句话说，Lamport时间戳让我们能够按照因果关系顺序来排序这些操作。但反过来则不一定成立，即便操作A的Lamport时间戳比操作B低，并不意味着它在因果上先于操作B，因为我们无法确保并行操作之间Lamport时间戳的相互关系。但正如我们之前所确认的，对于并行插入的排序顺序我们并不关心，只要这个排序是一致的就好。

通过按Lamport时间戳的降序对插入操作进行排序，并在必要时通过副本ID来打破同位，我们实现了一个既尊重因果关系又保持一致性的排序方案。

[视频](https://zed.dev/img/post/crdts/order-by-lamport.webm)  
*当我们对同一位置的插入操作按其Lamport时间戳降序排序时，我们既保留了用户的原意，同时又确保了所有副本之间的排序一致性。*

## 撤销与重做机制
在单用户系统中，撤销和重做历史通常通过一系列简单的编辑操作堆栈来表示。想要撤销一个操作时，只需从撤销堆栈中取出顶部的编辑，对当前文本应用其逆向操作，并将其推送到重做堆栈中。但这种方式仅适用于为整个文档维护一个全局的撤销历史。历史记录中任何操作的偏移只对该操作被执行时的文档状态有效，这意味着必须按照操作发生的反向顺序来依次撤销操作。

然而，在多人协作编辑的环境下，为整个文档维护一个单一的撤销历史是行不通的。进行撤销操作时，用户期待的是撤销自己输入的文本。因此，每个参与者都需要有自己的撤销堆栈。这意味着我们需要能够按任意顺序进行撤销和重做操作。仅依靠一个共享的全局编辑操作堆栈是不够的。

作为替代，我们采用了一个撤销映射的方式，它将操作ID与计数值关联起来。如果计数为零，表示该操作尚未被撤销；如果计数为奇数，表示操作已被撤销；若为偶数，则表示操作已被重做。撤销和重做操作简单地通过更新这个映射中特定操作ID的计数来执行。在判断某个文本片段是否可见时，我们首先检查该插入是否已被撤销（撤销计数为奇数），然后检查是否存在任何删除标记，以及这些删除操作的撤销计数是否为偶数。

在发送撤销/重做操作时，可以直接指定这些撤销计数。如果两个用户同时撤销同一操作，他们会将该操作的撤销计数设置为相同的值，这样做保持了他们的意图，因为他们都希望撤销或重做该操作。目前，我们仅允许用户撤销自己的操作，但未来可能会加入允许撤销协作者操作的功能。

结语
显然，关于这个主题还有很多细节值得深入探讨。我们如何高效实现这一方案？我们怎样将CRDTs融入到能够营造共享工作空间假象的更广泛的系统中？我们如何确保这个复杂的分布式系统的可靠性？除了用于协作之外，CRDTs还有什么其他用途呢？此外，还有编辑器的其他组成部分——Ropes，我们的[GPU加速UI框架](../rust/leveraging-rust-and-the-gpu-to-render-user-interfaces-at-120-fps.md)，Tree-sitter，集成的终端，等等。

我们期待在未来的月份和年份中讨论这些话题。更重要的是，我们期待利用这项技术推出一个能让您感到更加满意和提高生产力的编辑器。感谢您的阅读！