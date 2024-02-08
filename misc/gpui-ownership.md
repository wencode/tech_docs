# GPUI中的所有权与数据流

*原文:[Ownership and data flow in GPUI](https://zed.dev/blog/gpui-ownership)*

在构建Zed用户界面的过程中，我们最初面临的一个挑战是Rust的严格所有权系统。Rust中，每个对象都有唯一的所有者，这促使数据以树形结构组织，避免循环引用或共享所有权。在开发Zed之前，我大多数关于GUI编程的经验来源于web技术，而在JavaScript中，由于有垃圾回收机制，开发者不需要过多考虑所有权问题。例如，很容易将一个鼠标事件监听器附加到DOM节点上，这个节点捕获对this的引用，我的大部分UI构建直觉都是基于这种方式。然而，在Rust中，事件监听器中捕获self远不那么直接。

因此，当我们在2019年开始开发Zed时，很明显我们需要重新考虑以往在使用web和其他框架时学到的很多知识。我们需要一个既适合Rust，又能够动态表现真实世界图形界面的系统。例如，Zed的工作区可以显示不同类型的模态对话框，这些对话框需要能向工作区发出事件，告知何时关闭。我们还需要能够异步更新子树，比如当文件系统发生变化时，在项目面板中进行更新。当然还有许多其他例子，我们希望能够处理所有这些问题，而不是依赖于特殊的数据结构来表示应用状态。我们尽可能地避免使用宏，而是使用普通的Rust结构体。

在最初尝试使用Rc等内置类型未能成功后，我们开始探索一种方法，这种方法至今仍在Zed定制的UI框架GPUI中使用。在GPUI中，应用程序中的每个模型或视图实际上都由一个称为AppContext的顶级对象所拥有。当你创建一个新模型或视图（我们统称之为实体）时，你需要将状态的所有权转交给应用程序，这样它就能参与到各种应用服务中，并与其他实体进行交互。

以下面的简易应用为例。我们通过调用run函数并传入一个回调函数来启动应用，这个回调函数会接收到一个对AppContext的引用，AppContext拥有应用的所有状态。AppContext是我们访问所有应用级服务的入口，比如打开窗口、展示对话框等。它还提供了一个new_model方法，我们在下面用它来创建一个模型，并将其所有权交给应用程序。

```rust
use gpui::{prelude::*, App, AppContext, Model};
 
struct Counter {
    count: usize,
}
 
fn main() {
    App::new().run(|cx: &mut AppContext| {
        let counter: Model<Counter> = cx.new_model(|_cx| Counter { count: 0 });
        // ...
    });
}
```

调用new_model会返回一个模型句柄，这个句柄带有一个基于它所引用对象类型的类型参数。这个Model<Counter>句柄本身并不提供对模型状态的直接访问。它只是一个静态的标识符加上一个编译时的类型标签，并且维护着对应用拥有的底层Counter对象的引用计数。

这个句柄在某种程度上类似于Rust标准库中的Rc，当句柄被复制时，引用计数会增加，当它被丢弃时会减少，这样可以实现对底层模型的共享所有权。但不同于Rc的是，它只在有AppContext引用的情况下才提供对模型状态的访问。句柄并不真正拥有状态，但可以用它从其真正的所有者AppContext那里访问状态。我们继续用这个简单的例子，利用上下文来增加计数器。为了简洁，我将省略一些设置代码。

```rust
App::new().run(|cx: &mut AppContext| {
    let counter = cx.new_model(|_cx| Counter { count: 0 });
    // Call `update` to access the model's state.
    counter.update(cx, |counter: &mut Counter, cx: &mut ModelContext<Counter>| {
        counter.count += 1;
    });
});
```

为了更新计数器，我们需要在句柄上调用update函数，传入上下文引用和一个回调函数。这个回调函数将会获得一个指向计数器的可变引用，我们可以利用它来改变状态。

此外，回调函数还会收到一个第二个ModelContext<Counter>引用。这个引用与传递给run函数的回调中的AppContext引用类似。ModelContext实际上是对AppContext的封装，但它包含了一些额外的数据，用以指示它与特定模型的关联，在我们的例子中就是计数器。

除了AppContext提供的应用级服务之外，ModelContext还提供访问模型级服务的能力。例如，我们可以使用它来通知这个模型的观察者，告知它们模型状态已经改变。让我们在示例中加入这个操作，通过调用cx.notify()。

```rust
App::new().run(|cx: &mut AppContext| {
    let counter = cx.new_model(|_cx| Counter { count: 0 });
    counter.update(cx, |counter, cx| {
        counter.count += 1;
        cx.notify(); // Notify observers
    });
});
```

接下来，我们来看看如何观察这些通知。在更新第一个计数器之前，我们会构建一个观察它的第二个计数器。每当第一个计数器发生变化，我们就将其计数值翻倍并赋给第二个计数器。注意我们是如何在第二个计数器的ModelContext上调用observe方法，以确保它在第一个计数器发送通知时能够收到通知的。调用observe会返回一个Subscription，我们解除它，以保持这种行为，只要两个计数器都存在。我们也可以选择存储这个订阅，并在适当的时候丢弃它，以取消这种行为。

observe回调会传递一个可变引用给观察者，并传递一个句柄给被观察的计数器，我们可以通过read方法来访问它的状态。

```rust
App::new().run(|cx: &mut AppContext| {
    let counter: Model<Counter> = cx.new_model(|_cx| Counter { count: 0 });
    let observer = cx.new_model(|cx: &mut ModelContext<Counter>| {
        cx.observe(&counter, |observer, observed, cx| {
            observer.count = observed.read(cx).count * 2;
        })
        .detach();
 
        Counter {
            count: 0,
        }
    });
 
    counter.update(cx, |counter, cx| {
        counter.count += 1;
        cx.notify();
    });
 
    assert_eq!(observer.read(cx).count, 2);
});
```

在更新第一个计数器之后，你会发现观察计数器的状态是根据我们的订阅进行维护的。

除了observe和notify，这两者表示实体状态的变化外，GPUI还提供了subscribe和emit，这允许实体发出具有特定类型的事件。要使用这个系统，发出事件的对象必须实现EventEmitter特性。

现在，我们来介绍一种叫做CounterChangeEvent的新事件类型，并指出Counter可以发出这种类型的事件。

```rust
struct CounterChangeEvent {
    increment: usize,
}
 
impl EventEmitter<CounterChangeEvent> for Counter {}
```

接下来，我们将更新我们的示例，用订阅来替换观察。每次我们增加计数器时，我们会发出一个表明增量的Change事件。

```rust
App::new().run(|cx: &mut AppContext| {
    let counter: Model<Counter> = cx.new_model(|_cx| Counter { count: 0 });
    let subscriber = cx.new_model(|cx: &mut ModelContext<Counter>| {
        cx.subscribe(&counter, |subscriber, _emitter, event, _cx| {
            subscriber.count += event.increment * 2;
        })
        .detach();
 
        Counter {
            count: counter.read(cx).count * 2,
        }
    });
 
    counter.update(cx, |counter, cx| {
        counter.count += 2;
        cx.emit(CounterChangeEvent { increment: 2 });
        cx.notify();
    });
 
    assert_eq!(subscriber.read(cx).count, 4);
});
```

现在，我们深入GPUI的内部，探索观察和订阅功能是如何实现的。

在深入了解GPUI事件处理的细节之前，我想回顾我过去在Atom编辑器上的一次有教育意义的经验。那时，我在JavaScript中实现了一个自定义的事件系统。我设计了一个看似简单的事件发射器，事件监听器被保存在一个数组中，每当发出事件时，就会依次调用这些监听器。

然而，这种简单性导致了一个微妙的错误，直到代码在生产环境中广泛使用后才被发现。当一个监听函数向其订阅的同一个发射器发出事件时，会不经意地触发重入现象，即在函数执行完成前，它就被再次调用。这种递归般的行为与我们对线性函数执行的预期相违背，使我们陷入了意外的状态。虽然JavaScript的垃圾回收机制确保了内存安全，但它的宽松所有权模型让我很容易编写出这个错误。

Rust的限制让这种简单方法变得更加困难。我们被强烈引导走向不同的路径，以避免上述描述的重入问题。在GPUI中，当你调用emit或notify时，并不会调用任何监听器。相反，我们将数据推入effect队列。在每次更新的末尾，我们会处理这些effect，从队列前端弹出直到队列为空，然后将控制权返回给事件循环。任何effect处理程序都可以推入更多effect，但系统最终会趋于平稳。这为我们提供了无重入错误的运行到完成语义，并且与Rust很好地协同工作。

以下是来自app.rs的这种方法的核心部分。我将在下面进行解释。

```rust
impl AppContext {
    pub(crate) fn update<R>(&mut self, update: impl FnOnce(&mut Self) -> R) -> R {
        self.pending_updates += 1;
        let result = update(self);
        if !self.flushing_effects && self.pending_updates == 1 {
            self.flushing_effects = true;
            self.flush_effects();
            self.flushing_effects = false;
        }
        self.pending_updates -= 1;
        result
    }
 
    fn flush_effects(&mut self) {
        loop {
            self.release_dropped_entities();
            self.release_dropped_focus_handles();
 
            if let Some(effect) = self.pending_effects.pop_front() {
                match effect {
                    Effect::Notify { emitter } => {
                        self.apply_notify_effect(emitter);
                    }
 
                    Effect::Emit {
                        emitter,
                        event_type,
                        event,
                    } => self.apply_emit_effect(emitter, event_type, event),
 
                    // A few more effects, elided for clarity
                }
            } else {
                for window in self.windows.values() {
                    if let Some(window) = window.as_ref() {
                        if window.dirty {
                            window.platform_window.invalidate();
                        }
                    }
                }
 
                break;
            }
        }
    }
 
    // Lots more methods...
}
```

AppContext::update方法进行了一些准备工作，使其能够被重入地调用。在退出最顶层调用之前，它会调用flush_effects。flush_effects方法是一个循环。在每次循环中，我们会释放被丢弃的实体和焦点句柄，这导致引用计数降至0的资源的所有权被放弃。然后，我们从队列中取出下一个effect并应用之。如果没有下一个效果，我们会遍历窗口，并对任何被标记为脏的窗口使其平台窗口失效，从而安排它们在下一帧进行重绘。之后，我们就退出循环。

下一步，我们将使用AppContext::update来实现update_model。我将在下面搭建它的基础结构，这样我们就可以在继续实现之前讨论其签名。

```rust
impl AppContext {
    fn update_model<T: 'static, R>(
        &mut self,
        model: &Model<T>,
        update: impl FnOnce(&mut T, &mut ModelContext<'_, T>) -> R,
    ) -> R {
        todo!()
    }
}
```

该方法接收一个回调，这个回调需要两个可变引用，一个是对通过给定句柄引用的模型状态的引用，另一个是对ModelContext的引用。正如我之前提到的，ModelContext实际上是对AppContext的封装。由于AppContext拥有这个模型，这最初看起来似乎需要对同一数据进行多次可变借用，但Rust不允许这样做。

我们的解决方案是暂时从AppContext那里“租借”模型状态，将其从上下文中移除，并转移到栈上。在调用回调之后，我们结束租借，将所有权还原给上下文。

```rust
impl AppContext {
    fn update_model<T: 'static, R>(
        &mut self,
        model: &Model<T>,
        update: impl FnOnce(&mut T, &mut ModelContext<'_, T>) -> R,
    ) -> R {
        self.update(|cx| {
            let mut entity = cx.entities.lease(model);
            let result = update(&mut entity, &mut ModelContext::new(cx, model.downgrade()));
            cx.entities.end_lease(entity);
            result
        })
    }
}
```

如果你尝试重入地更新一个实体，这确实可能会导致问题，但实际上我们发现避免这种情况是相当可行的，并且一旦我们犯了错误，通常能够迅速且容易地发现。

现在我已经介绍了GPUI中状态管理的基本原理，下一步将是讲述我们如何利用视图在屏幕上展示这些状态。但这需要等到我们的下一篇文章。在那之前，请浏览我们的源代码，并且今天在Zed中加入我们的首次Fireside Hack直播活动。今天恰好是我的生日，我觉得没有比和大家一起在Zed中共度更好的方式了。
