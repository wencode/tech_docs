# ESP32 Embedded Rust at the HAL：通过定时器检测实现按钮控制闪烁功能

## 目录
* [介绍](#介绍)
    - 📚 [必备知识](#必备知识)
    - 💾 [软件配置](#软件配置)
    - 🛠 [硬件搭建](#硬件搭建)
        +  [材料清单](#材料清单)
* 👨‍🎨 [软件设计](#软件设计)
* 👨‍💻 [代码实现](#代码实现)
    - 📥 [引入Crate库](#引入Crate库)
    - 🎛 [外围设备配置代码](#外围设备配置代码)
    - 📱 [应用代码](#应用代码)
* 📱 [完整的程序代码](#完整的程序代码)
* 🔬 [进一步的实验与创意思考](#进一步的实验与创意思考)
* [总结](#总结)

    本篇博客是一个系列文章中的第二篇，我将在这一系列中利用HAL层的嵌入式Rust语言，探讨ESP32C3微控制器的各种外围设备。请留意，后续文章中的某些概念可能会基于之前文章中介绍的概念。

前面的文章包括（按照发布顺序排列）：
1. [ESP32 Embedded Rust at the HAL：利用GPIO按钮控制LED闪烁](esp32-embedded-rust-at-the-hal-gpio-button-controlled-blinking.md)

## 介绍

在本篇文章中，我将通过使用定时器/计数器外围设备来优化我之前文章中的GPIO按钮控制LED闪烁项目。在上一篇文章中，重点讨论的是GPIO外围设备，我用连接到GPIO输入的按钮来控制连接到GPIO输出的LED的闪烁速度。之前的延迟是通过编码算法实现的，即通过一段循环代码来生成必要的延迟。文章中也提到，使用软件方式来产生延迟并不理想，因为这种方式不易扩展，而硬件方法（例如使用定时器外设）则更为合适。在本文中，我将通过使用定时器/计数器外设来控制延迟，以此来增强原有代码。这将使得延迟更加确定且在不同平台间更易于扩展。再次强调，我将不使用中断，而是通过轮询定时器/计数器来检测经过的时间。

### 📚 必备知识

要理解本文的内容，你需要具备以下几点：

* Rust编程的基础知识。

* 熟悉使用Rust为ESP32开发嵌入式应用程序的基本模板（[《Rust on ESP》](https://esp-rs.github.io/book/)一书是一份很好的参考资料）。

### 💾 软件配置

本文中提到的所有代码都可以在[apollolabs ESP32C3](https://github.com/apollolabsdev/ESP32C3) git仓库中找到。请注意，如果仓库中的代码有些许不同，这可能是为了提升代码质量或适应HAL/Rust的更新而做的修改。

此外，完整的项目（包括代码和模拟）可以在[Wokwi](https://wokwi.com/projects/363050473642243073)的网站上找到。

### 🛠 硬件搭建
#### 所需材料
* [ESP32-C3-DevKitM](https://docs.espressif.com/projects/esp-idf/en/latest/esp32c3/hw-reference/esp32c3/user-guide-devkitm-1.html)
![](https://cdn.hashnode.com/res/hashnode/image/upload/v1681796942083/da81fb7b-1f90-4593-a848-11c53a87821d.jpeg?auto=compress,format&format=webp)

* 任意颜色的LED灯

* 限流电阻

* 按键

#### 🔌 连接方式
📝 注意

    *所有的连接细节也展示在了[Wokwi示例](https://wokwi.com/projects/362145427195567105)中。*

连接方式如下：

* LED的正极通过一个电阻连接到开发套件的第4号引脚。这个引脚将用作输出端。LED的负极将连接到地线。

* 按钮的一端连接到开发套件的第0号引脚。这个引脚将用作输入端。按钮的另一端也将连接到地线。


## 👨‍🎨 软件设计
本文中的应用程序采取了与我之前[文章](esp32-embedded-rust-at-the-hal-gpio-button-controlled-blinking.md)中相同的算法流程，但进行了一些小的修改。这里，我不再通过更新循环变量来检查是否达到最大值，而是通过轮询一个定时器/计数器来判断是否达到了预定的延迟时间。现在，让我们把这些调整融入流程图中，来看看它的最新样貌：
![](https://cdn.hashnode.com/res/hashnode/image/upload/v1656580071411/5mWZAhHmk.png?auto=compress,format&format=webp)

🚨 重要说明：

    *延迟方法有两种类型：阻塞和非阻塞。阻塞是指在延迟结束之前，控制器将处于空闲状态（操作被阻塞）。非阻塞则意味着允许恢复操作，控制器可以同时执行其他任务。代码会不断返回去检查（轮询）定时器是否结束了延迟。这意味着对于我们这种需要在时间流逝的同时检查按钮是否被按下的轮询方法，必须采用非阻塞的方式。需要指出的是，如果我们使用中断，这一切都不会是问题，因为我们将有一个中断服务例程来告知我们按钮被按下。值得注意的是，除非禁用了抢占功能，否则中断不会受到阻塞延迟的影响。*

现在，让我们开始实现这一算法。

## 👨‍💻 代码实现

📥 引入Crate库 

在这个实现中，需要如下三个crates：

esp_backtrace crate，用于定义程序发生惊恐时的行为。

esp32c3_hal crate，用于引入ESP32C3设备的硬件抽象层。

```rust
use esp32c3_hal::{clock::ClockControl, pac::Peripherals, prelude::*, timer::TimerGroup, Rtc, Delay, IO};
use esp_backtrace as _;
```

📝 笔记

    - 每个crate的导入都需要在Cargo.toml文件中对应地添加依赖项。

    - 在早期版本的esp32c3_hal中，需要导入riscv_rt crate以支持#[entry]属性宏的启动代码。但从0.7.0版本开始，#[entry]属性已被整合进esp32c3_hal中。这意味着为了支持#[entry]属性，不再需要单独导入riscv_rt crate。

### 🎛 外设配置代码 
在我们的应用程序代码之前，需要通过以下步骤来配置外设：

1️⃣ 获取设备外围设备的句柄：在嵌入式Rust编程中，作为单例设计模式的一部分，我们首先需要获取PAC级别的外设。这是通过使用take()方法来完成的。在这里，我创建了一个名为dp的设备外设处理程序，如下所示：

```rust
let peripherals = Peripherals::take();
```

📝 注意

    *这是与esp32c3_hal早期版本相比的另一个不同点。在以前的实现中，Peripherals是从pac模块导入的，现在则是从peripherals模块导入的。虽然两者都使用take方法，但请注意，在更新的实现中，它不会返回Result类型，因此无需进行解包处理。*

2️⃣ 禁用看门狗：与之前的文章一样，ESP32C3默认启用了看门狗，需要将其禁用。如果不禁用，设备将不断重启。为了避免这个问题，需要加入以下代码：

```rust
let system = peripherals.SYSTEM.split();
let clocks = ClockControl::boot_defaults(system.clock_control).freeze();

// Instantiate and Create Handles for the RTC and TIMG watchdog timers
let mut rtc = Rtc::new(peripherals.RTC_CNTL);
let timer_group0 = TimerGroup::new(peripherals.TIMG0, &clocks);
let mut wdt0 = timer_group0.wdt;
let timer_group1 = TimerGroup::new(peripherals.TIMG1, &clocks);
let mut wdt1 = timer_group1.wdt;
```

3️⃣ 实例化并创建IO句柄：我们需要将LED引脚配置为push-pull输出，并获得一个句柄来控制它。我们还需要获得按钮输入引脚的句柄。在我们为LED和按钮获得任何句柄之前，我们需要创建一个IO结构实例。IO结构实例提供了一个HAL设计的结构体，允许我们访问所有的GPIO引脚，使我们能为单独的引脚创建句柄。这与其他HAL中使用的split方法概念相似（更多细节参见[此处](https://apollolabsblog.hashnode.dev/demystifying-rust-embedded-hal-split-and-constrain-methods）。我们通过在IO结构上调用new()实例方法来实现这一点，如下所示：

```rust
let io = IO::new(peripherals.GPIO, peripherals.IO_MUX);
```

4️⃣ 获取LED的句柄并配置为输出：如前所述，LED连接到第4号引脚（gpio4）。因此，我们需要为LED引脚创建一个句柄，并使用into_push_pull_output()方法将gpio4配置为push-pull输出。我们将这个句柄命名为led，并按以下方式配置：

```rust
let mut led = io.pins.gpio4.into_push_pull_output();
```

[HAL文档](https://docs.rs/esp32c3-hal/latest/esp32c3_hal/gpio/struct.GpioPin.html)页面提供了GpioPin类型支持的所有方法的完整列表。

5️⃣ 获取并配置输入按钮的句柄：如前所述，按钮连接到0号引脚（gpio0）。此外，在按钮被按下时，它会连接到地线。对于按钮未被按下的状态，需要加入一个上拉电阻，使引脚变高。我们可以使用into_pull_up_input()方法为引脚配置一个内部上拉，如下所示：

```rust
let button = io.pins.gpio0.into_pull_up_input();
```

需要注意的是，与LED输出不同，这里的按钮句柄不需要是可变的，因为我们只需要读取它。

6️⃣ 获取并配置定时器：我们首先需要获取对定时器外设的访问权限以使用其方法。在ESP32中，定时器存在于所谓的定时器组内。请注意，在第2步禁用看门狗时，我们实际上已经为timer_group0创建了一个句柄，即let timer_group0 = TimerGroup::new(peripherals.TIMG0, &clocks);。因此，我们现在需要做的就是获取对timer0的访问权，并按以下方式创建一个句柄：

```rust
let mut timer0 = timer_group0.timer0;
```

这将使我们能够访问timer0的方法。

这就是配置部分的全部内容。现在让我们继续进入应用程序代码的编写。

### 📱 应用程序代码

按照之前描述的设计，我首先需要初始化一个延迟变量del_var，并设置LED的输出。del_var需要是可变的，因为它将在延迟循环中被修改。我还选择默认将LED的初始输出电平设置为低。使用之前提到的相同Pin方法，有一个set_low()方法，我使用它来实现这一目标。

```rust
        // Initialize LED to on or off
    led.set_low().unwrap();

    // Create and initialize a delay variable to manage delay duration
    let mut del_var = 2000_u32.millis();
```

需要注意的是，对于del_var，我使用的是Duration类型。millis()是一个将数字转换为Duration的方法。

接下来，在程序循环中，我首先启动计数器。查阅[定时器文档](https://docs.rs/esp32c3-hal/0.8.0/esp32c3_hal/timer/struct.Timer.html)中可用的方法，有一个start方法，其签名如下：

```rust
fn start<Time>(&mut self, timeout: Time)
where
    Time: Into<<Timer<T> as CountDown>::Time>,
```

这个方法启动定时器，开始计算一个指定的超时持续时间。timeout参数是一个泛型Time。在应用程序循环中，定时器接着按以下方式启动：

```rust
timer0.start(del_var);
```

在启动定时器之后，我现在需要持续轮询定时器以获取已过去的时间。作为定时器可用[方法](https://docs.rs/stm32f4xx-hal/latest/stm32f4xx_hal/timer/counter/struct.Counter.html)的一部分，有一个wait方法，其签名如下：

```
fn wait(&mut self) -> Result<(), Error<Void>>
```

这个方法非阻塞地“等待”倒计时结束，并返回一个Result类型。如果Result是Ok()，则表示倒计时已完成（定时器到期）。这样就形成了以下应用程序循环：

```rust
// Application Loop
loop {
    // Start counter with with del_var duration
    timer0.start(del_var);
    // Enter loop and check for button press until counter reaches del_var
    while timer0.wait() != Ok(()) {
        if button.is_low().unwrap() {
            // If button pressed decrease the delay value by 500 ms
            del_var = del_var - 500_u32.millis();
            // If updated delay value drops below 500 ms then reset it back to starting value to 2 secs
            if del_var.to_millis() < 1000_u32 {
               del_var = 2000_u32.millis();
            }
            // Exit delay loop since button was pressed
            break;
        }
    }
    // Toggle LED
    led.toggle().unwrap();
}
```

在这里，你可以看到我创建了一个while循环，不断轮询timer0，直到它达到等同于2秒的del_var。如设计部分所述，如果循环自然结束，则del_var保持不变。在延迟期间的任何时刻，如果按钮被按下，我可以使用is_low()方法来检测。在这种情况下，我将del_var减少500.millis()的持续时间。如果最终得到的del_var值小于500_u32，则我将恢复初始设定的2001.millis()的值。

🚨 重要说明：

    *与上一篇文章一样，一旦运行代码，你会看到LED在闪烁，但可能会注意到一些异常行为。闪烁频率似乎会以随机顺序变化。这是因为机械按钮上的“抖动”效应。在Wokwi中，也可以消除抖动效应。有关更多细节，请查看下面的实验思路部分。*


## 📱 完整的应用程序代码

这是本文描述实现的完整代码。你还可以在[apollolabs ESP32C3](https://github.com/apollolabsdev/ESP32C3) git仓库上找到这个完整项目和其他项目。此外，Wokwi项目可以在[这里](https://wokwi.com/projects/362145427195567105)访问。

```rust
#![no_std]
#![no_main]

use esp32c3_hal::{
    clock::ClockControl, peripherals::Peripherals, prelude::*, timer::TimerGroup, Rtc, IO,
};
use esp_backtrace as _;

#[entry]
fn main() -> ! {
    // Take Peripherals, Initialize Clocks, and Create a Handle for Each
    let peripherals = Peripherals::take();
    let system = peripherals.SYSTEM.split();
    let clocks = ClockControl::boot_defaults(system.clock_control).freeze();

    // Instantiate and Create Handles for the RTC and TIMG watchdog timers
    let mut rtc = Rtc::new(peripherals.RTC_CNTL);
    let timer_group0 = TimerGroup::new(peripherals.TIMG0, &clocks);
    let mut wdt0 = timer_group0.wdt;
    let timer_group1 = TimerGroup::new(peripherals.TIMG1, &clocks);
    let mut wdt1 = timer_group1.wdt;

    // Disable the RTC and TIMG watchdog timers
    rtc.swd.disable();
    rtc.rwdt.disable();
    wdt0.disable();
    wdt1.disable();

    // Instantiate and Create Handle for IO
    let io = IO::new(peripherals.GPIO, peripherals.IO_MUX);

    // Instantiate and Create Handle for LED output & Button Input
    let mut led = io.pins.gpio4.into_push_pull_output();
    let button = io.pins.gpio0.into_pull_up_input();

    // Instantiate and Create Handle for Timer
    let mut timer0 = timer_group0.timer0;

    // Initialize LED to on or off
    led.set_low().unwrap();

    // Create and initialize a delay variable to manage delay duration
    let mut del_var = 2000_u32.millis();

    // Application Loop
    loop {
        // Start counter with with del_var duration
        timer0.start(del_var);
        // Enter loop and check for button press until counter reaches del_var
        while timer0.wait() != Ok(()) {
            if button.is_low().unwrap() {
                // If button pressed decrease the delay value by 500 ms
                del_var = del_var - 500_u32.millis();
                // If updated delay value drops below 500 ms then reset it back to starting value to 2 secs
                if del_var.to_millis() < 1000_u32 {
                    del_var = 2000_u32.millis();
                }
                // Exit delay loop since button was pressed
                break;
            }
        }
        // Toggle LED
        led.toggle().unwrap();
    }
}
```

## 结论
在本文中，我们利用ESP32C3的GPIO和计数器外设创建了一个LED控制应用程序。所有代码都是基于轮询（不使用中断）实现的，这意味着同样利用了非阻塞计数器。所有代码都是在HAL层级，并使用[esp32c3-hal](https://docs.rs/esp32c3-hal/latest/esp32c3_hal/)创建的。
