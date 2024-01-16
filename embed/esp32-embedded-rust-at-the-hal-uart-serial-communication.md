# ESP32 Embedded Rust at the HAL：UART串行通信

## 目录
* [介绍](#介绍)
    - 📚 [背景知识](#背景知识)
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

前面的文章包括（按照发布顺序排列）：
1. [ESP32 Embedded Rust at the HAL：利用GPIO按钮控制LED闪烁](esp32-embedded-rust-at-the-hal-gpio-button-controlled-blinking.md)
2. * [2.ESP32 Embedded Rust at the HAL：通过定时器检测实现按钮控制闪烁功能](esp32-embedded-rust-at-the-hal-button-controlled-blinking-by-timer-polling.md)

### 📚 背景知识
为了理解本文内容，你需要具备以下知识：

* 具有 Rust 编程的基础知识。

* 熟悉使用在 Rust 中创建嵌入式应用的基本模板。

* 了解 UART 通信的[基本原理](https://en.wikipedia.org/wiki/Universal_asynchronous_receiver-transmitter)。

### 💾 软件配置
这篇文章中展示的所有代码都可在 [apollolabs ESP32C3](https://github.com/apollolabsdev/ESP32C3) git 仓库找到。请注意，如果 git 仓库中的代码有所不同，可能是因为进行了修改，以提升代码质量或适应 HAL/Rust 的最新更新。

另外，完整的项目（包括代码和仿真）可在 [Wokwi](https://wokwi.com/projects/363774125451149313) 网站上找到。

此外，如果你使用的是实体硬件，你需要在你的主机电脑上安装串行通信终端软件。针对不同操作系统的推荐软件包括：

Windows 系统：

* [PuTTy](https://www.putty.org/)

* [Teraterm](https://ttssh2.osdn.jp/index.html.en)

Mac 和 Linux 系统：

* [minicom](https://wiki.emacinc.com/wiki/Getting_Started_With_Minicom)

关于不同操作系统安装这些软件的指导可在 [Discovery Book](https://docs.rust-embedded.org/discovery/microbit/06-serial-communication/index.html) 中找到。

如果你使用的是 Wokwi，那么串行终端已经集成在仿真窗口里。

### 🛠 硬件搭建
#### 所需材料
* [ESP32-C3-DevKitM](https://docs.espressif.com/projects/esp-idf/en/latest/esp32c3/hw-reference/esp32c3/user-guide-devkitm-1.html)
![](https://cdn.hashnode.com/res/hashnode/image/upload/v1681796942083/da81fb7b-1f90-4593-a848-11c53a87821d.jpeg?auto=compress,format&format=webp)

* 任意颜色的LED灯

* 限流电阻

* 按键

#### 🔌 连接方式
📝 注意

    所有的连接细节也展示在了[Wokwi示例](https://wokwi.com/projects/363774125451149313)中。

连接方式如下:

* 将 LED 的阳极通过一个电阻接到开发板的第 4 号引脚。这个引脚将作为输出使用。LED 的阴极接地。

* 按钮的一端接到开发板的第 0 号引脚。这个引脚将作为输入使用。按钮的另一端接地。

## 👨‍🎨 软件设计

现在，让我们开始实现这一算法。

这篇文章中的应用程序采取了与我之前文章中相同的算法策略。不过，在这里，我会增加两个功能：

* 我会使用 debouncr 包（没错，就是这么拼写 😀）来消除按钮的抖动效应。

* 我将向 PC 终端发送或显示一个用于跟踪按钮按压次数的数值。

更新后的流程图将会是这样的：
![](https://cdn.hashnode.com/res/hashnode/image/upload/v1657001444071/is1tg2bU-.png?auto=compress,format&format=webp&auto=compress,format&format=webp)

现在，让我们开始实现这一算法。

## 👨‍💻 代码实现

📥 Crate 导入
在这个实现中，需要导入以下三个 crates：

* esp_backtrace crate，用于定义应对惊恐行为的处理。

* esp32c3_hal crate，用于导入 ESP32C3 设备的硬件抽象层。

* debouncr crate，用于消除按钮抖动。

* core::fmt::Write crate，将使我们能够使用 writeln! 宏，便于打印输出。

```rust
use esp32c3_hal::{clock::ClockControl, pac::Peripherals, prelude::*, timer::TimerGroup, Rtc, Delay, IO};
use esp_backtrace as _;
use debouncr::{debounce_3, Edge};
use core::fmt::Write;
```

📝 注意

    * 每个 crate 的导入都需要在 Cargo.toml 文件中有对应的依赖声明。

    * 早期版本的 esp32c3_hal 需要导入 riscv_rt crate，以支持 #[entry] 属性宏的启动代码。从 0.7.0 版本起，#[entry] 属性已集成至 esp32c3_hal 中。这意味着现在不再需要单独导入 riscv_rt 来支持 #[entry] 属性。

### 🎛 外设配置代码 
在我们的应用程序代码之前，需要通过以下步骤来配置外设：

1️⃣ 获取设备外围设备的句柄：在嵌入式Rust编程中，作为单例设计模式的一部分，我们首先需要获取PAC级别的外设。这是通过使用take()方法来完成的。在这里，我创建了一个名为dp的设备外设处理程序，如下所示：

```rust
let peripherals = Peripherals::take();
```

📝 注意

    这是与esp32c3_hal早期版本相比的另一个不同点。在以前的实现中，Peripherals是从pac模块导入的，现在则是从peripherals模块导入的。虽然两者都使用take方法，但请注意，在更新的实现中，它不会返回Result类型，因此无需进行解包处理。

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

3️⃣ 实例化并创建IO句柄：我们需要将LED引脚配置为push-pull输出，并获得一个句柄来控制它。我们还需要获得按钮输入引脚的句柄。在我们为LED和按钮获得任何句柄之前，我们需要创建一个IO结构实例。IO结构实例提供了一个HAL设计的结构体，允许我们访问所有的GPIO引脚，使我们能为单独的引脚创建句柄。这与其他HAL中使用的split方法概念相似（更多细节参见[此处](https://apollolabsblog.hashnode.dev/demystifying-rust-embedded-hal-split-and-constrain-methods)）。我们通过在IO结构上调用new()实例方法来实现这一点，如下所示：

```rust
let io = IO::new(peripherals.GPIO, peripherals.IO_MUX);
```

4️⃣ 获取LED的句柄并配置为输出：如前所述，LED连接到第4号引脚（gpio4）。因此，我们需要为LED引脚创建一个句柄，并使用into_push_pull_output()方法将gpio4配置为push-pull输出。我们将这个句柄命名为led，并按以下方式配置：

```rust
let mut led = io.pins.gpio4.into_push_pull_output();
```

[HAL文档](https://docs.rs/esp32c3-hal/latest/esp32c3_hal/gpio/struct.GpioPin.html)页面提供了GpioPin类型支持的所有方法的完整列表。

5️⃣ 获取并配置输入按钮的句柄：如前所述，按钮连接到0号引脚（gpio0）。此外，在按钮被按下时，它会连接到地线。对于按钮未被按下的状态，需要加入一个上拉电阻，使引脚电平变高。我们可以使用into_pull_up_input()方法为引脚配置一个内部上拉，如下所示：

```rust
let button = io.pins.gpio0.into_pull_up_input();
```

需要注意的是，与LED输出不同，这里的按钮句柄不需要是可变的，因为我们只需要读取它。

6️⃣ 获取控制句柄并配置 UART 通道：在 ESP32C3 上，RX 和 TX 引脚默认配置为连接到 UART0。因此，无需为 UART 操作配置新的引脚。为了实例化 UART，在文档中，提供了一个新方法来实例化 UART 通道，其方法签名如下：

```rust
pub fn new(uart: impl Peripheral<P = T> + 'd) -> Uart<'d, T>
```

这个方法使用默认配置来实例化 UART。默认配置可以在[源代码](https://github.com/esp-rs/esp-hal/blob/main/esp-hal-common/src/uart.rs)中找到，具有以下参数：

```rust
impl Default for Config {
    fn default() -> Config {
        Config {
            baudrate: 115_200,
            data_bits: DataBits::DataBits8,
            parity: Parity::ParityNone,
            stop_bits: StopBits::STOP1,
        }
    }
}
```

由于我们需要为 UART0 实例化一个 UART 实例，因此将产生以下代码：

```rust
let mut uart0 = Uart::new(peripherals.UART0);
```

🚨 重要说明：

    为了了解默认配置的具体内容，我不得不查看源代码并找到 Default trait 的实现。不幸的是，HAL 文档本身并没有清晰地指出默认配置的内容。

    如果希望使用不同参数配置 UART，可以在创建 UART 实例时调用 new_with_config 函数。

💰 小贴士：

    在 Wokwi 上，有一个判断引脚是否正确配置的有效方法。在点击播放按钮后，仿真运行时点击暂停。每个引脚旁边将打印出活跃的引脚功能。

### 📱 应用程序代码

根据之前描述的设计，我首先需要初始化一个延迟变量 del_var，并设置 LED 的输出。del_var 需要是可变的，因为它将在延迟循环中被修改。我还选择默认将 LED 的初始输出电平设置为低。使用之前提及的 Pin 方法，我利用 set_low() 方法来实现这一点。

```rust
// Create and initialize a delay variable to manage delay loop
let mut del_var = 10_0000_u32;

// Initialize LED to on or off
led.set_low().unwrap();```

我还想初始化一个变量 value，用来记录按钮被按压的次数：

```rust
let mut value: u8 = 0;
```

接下来，还有一个小事项。我之前提到过，我将使用 debouncr crate 来减少按钮按压的抖动效果。这意味着我需要创建一个handler来使用这个 crate 的方法。根据 crate [文档](https://docs.rs/debouncr/latest/debouncr/)，为了实例化，我首先需要确定需要检测的状态数量。我选择了 16 个。

```rust
let mut debouncer = debounce_16(false);
```

我将 debouncer 初始化为 false，是因为文档中提到，如果按钮是低电平有效，就需要这么做。

🚨 重要说明：

    在选择去抖状态时，我通过实验发现，在实际硬件上，3 个状态就足以消除抖动效果。然而，在 Wokwi 上，我没有得到与实际硬件相一致的结果。尽管我设置了允许的最大状态数，抖动效果虽然有所减少，但并未完全消除。

接下来，在程序循环中，我首先开始调用一个延迟函数，在该函数内检查按钮是否被按下。延迟完成后，我使用 Pin 类型提供的 toggle() 方法来切换 LED。这是完整的应用程序循环：

```rust
loop {
    for _i in 1..del_var {
        if debouncer.update(button.is_low().unwrap()) == Some(Edge::Falling) {
            writeln!(tx, "Button Press {:02}\r", value).unwrap();
            value = value.wrapping_add(1);
            del_var = del_var - 2_5000_u32;
            if del_var < 2_5000_u32 {
                del_var = 10_0000_u32;
            }
            break;
        }
    }
    led.toggle();
}
```

整体结构与标准的[轮询式按钮控制闪烁应用程序](esp32-embedded-rust-at-the-hal-button-controlled-blinking-by-timer-polling.md)中的循环完全一致。外层的 for 循环通过 del_var 来追踪延迟。这里的一个不同点在于 if 语句的条件。对于这个条件，我使用了 debouncr crate 的 update 方法。在轮询引脚时，会反复调用 update 方法。update 方法返回一个 Option 枚举，当没有检测到按压时，它会不断提供 None。然而，如果检测到一个经过去抖处理的按压，update 方法会返回 Some(Edge::Falling) 或 Some(Edge::Rising)。由于 ESP32C3 使用的是低电平有效的按钮，因此按压通过下降沿来检测，成功去抖后会返回 Some(Edge::Falling)。

按钮检测完成后，我使用了之前导入的 core::fmt::Write 提供的 writeln! 宏。其使用方法与 Rust 中的 println! 进行格式化打印非常相似，只是有几个小的区别。来看看这个语句，

```rust
writeln!(uart0, "Button Press {:02}\r", value).unwrap();
```

如果你已经注意到，writeln! 需要三个参数，在 writeln! 的第一个参数中，我传递了 uart0 处理程序作为参数。为了让 writeln! 能够正常工作，uart0 必须为 Write trait 定义了 write_fmt 函数。此外，由于 writeln! 宏返回一个 Result 类型，所以需要对其进行解包。writeln! 的第三个参数包含了之前初始化的变量 value，该变量通过以下方式在每次调用时增加 1：

```rust
value = value.wrapping_add(1);
```

wrapping_add 方法，正如其名所示，对 value 执行包装加法，每次调用时增加 1，并在必要时进行环绕。剩余的代码负责递减 del_var 的值以减少延迟，并确保它不会降至零以下。最后，在延迟循环外部，使用 Pin 的 toggle 方法来切换 LED。

💰 小贴士：

    在 ESP 上进行控制台打印的另一种方法是使用 [esp-println](https://github.com/esp-rs/esp-println) crate。esp-println 在多个 Espressif 设备上提供了 print!、println! 和 dbg! 的实现，具有更好的可移植性。

## 📱 完整应用程序代码
以下是本文描述的实现的完整代码。你还可以在 [apollolabs ESP32C3](https://github.com/apollolabsdev/ESP32C3) git 仓库找到完整的项目和其他项目。此外，Wokwi 项目也可以在[这里](https://wokwi.com/projects/363774125451149313)访问。

```rust
#![no_std]
#![no_main]

use core::fmt::Write; // allows use to use the WriteLn! macro for easy printing
use debouncr::{debounce_16, Edge};
use esp32c3_hal::{
    clock::ClockControl, peripherals::Peripherals, prelude::*, timer::TimerGroup, uart::Uart, Rtc,
    IO,
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

    // Initialize LED to on or off
    led.set_low().unwrap();

    // Create UART instance with default config
    let mut uart0 = Uart::new(peripherals.UART0);

    // Initialize debouncer to false because button is active low
    let mut debouncer = debounce_16(false);

    // Create and initialize a delay variable to manage delay loop
    let mut del_var = 10_0000_u32;

    // Variable to keep track of how many button presses occured
    let mut value: u8 = 0;

    // Application Loop
    loop {
        // Enter Delay Loop
        for _i in 1..del_var {
            // Keep checking if button got pressed
            if debouncer.update(button.is_low().unwrap()) == Some(Edge::Falling) {
                // If button is pressed print to derial and decrease the delay value
                writeln!(uart0, "Button Press {:02}\r", value).unwrap();
                // Increment value keeping track of button presses
                value = value.wrapping_add(1);
                // Decrement the amount of delay
                del_var = del_var - 2_5000_u32;
                // If updated delay value drops below threshold then reset it back to starting value
                if del_var < 2_5000_u32 {
                    del_var = 10_0000_u32;
                }
                // Exit delay loop since button was pressed
                break;
            }
        }
        led.toggle().unwrap();
    }
}
```

## 🔬 进一步实验/创意
* 如果你有实体硬件，尝试用不同的参数来配置 UART。

* 实现具备发送和接收功能的 UART。

## 结论
在这篇文章中，我创建了一个 LED 控制应用程序，它利用了 ESP32C3 的 GPIO 和 UART 外设。每当 GPIO 按钮被按下，UART 外设就会向主机 PC 发送状态更新。所有代码都是基于轮询方式实现的（没有使用中断）。所有代码都是在硬件抽象层（HAL）使用 [esp32c3-hal](https://docs.rs/esp32c3-hal/latest/esp32c3_hal/) 来编写的。

