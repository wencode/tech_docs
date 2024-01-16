# ESP32 Embedded Rust at the HAL：利用GPIO按钮控制LED闪烁

## 目录
* [介绍](#介绍)
    - 📚 [背景知识](#背景知识)
    - 💾 [软件配置](#软件配置)
    - 🛠 [硬件搭建](#硬件搭建)
        +  [材料清单](#材料清单)
        + 🔌 [连接方式](#连接方式)
* 👨‍🎨 [软件设计](#软件设计)
* 👨‍💻 [代码实现](#代码实现)
    - 📥 [引入Crate库](#引入Crate库)
    - 🎛 [外设配置代码](#外设备配置代码)
    - 📱 [应用程序代码](#应用程序代码)
* 📱 [完整的程序代码](#完整的程序代码)
* 🔬 [进一步的实验与创意思考](#进一步的实验与创意思考)
* [总结](#总结)

## 介绍
这篇文章开启了一个新系列，我将在此系列中探索在硬件抽象层（HAL）上使用ESP32进行嵌入式Rust开发。对于那些一直关注我的读者来说，你们可能知道我在之前的系列文章中已经探讨了与STM32结合的嵌入式Rust开发，涉及到硬件抽象层（HAL）和外设访问Crate（PAC）层面（你可以在这里回顾STM32系列）。现在，我选择转向ESP32，原因如下：

1. **官方支持:** Rust得到Espressif的官方支持，这意味着可以获得良好维护的crate和详尽的文档。正如我在[另一文](https://apollolabsblog.hashnode.dev/what-the-hal-the-quest-for-finding-a-suitable-embedded-rust-hal)中提到的，如果我重新选择，ESP将是我的首选。

2. **Wokwi支持**: 对硬件组件的访问并不总是那么容易或经济。此外，组件的连接也可能带来麻烦。[Wokwi](https://wokwi.com/)提供了一个很棒的替代方案，可以在浏览器中模拟项目，它还支持在Rust环境下使用ESP。Wokwi不仅仅是一个普通的模拟器，它在功能上更接近真实的嵌入式硬件，并且得到社区的不断改进。Wokwi中创建的项目也可以直接在真实硬件上使用，唯一的区别在于使用真实硬件时需要配置调试/烧写工具链。

3. **项目示例**：基于上述原因，未来可以创建更多复杂的项目。此外，Wokwi还支持模拟Wifi连接，这将有助于创建物联网（IoT）项目示例。

4. **探索不同平台之间的差异**：我对在Rust实现上ESP和其他平台（例如STM32）之间的差异很感兴趣。这可能体现在文档或使用如embassy这样的框架上。比如，ESP32并没有像STM32或nRf设备那样支持embassy框架的HAL，但这并不意味着embassy框架不能用于ESP32。这些是我想要深入探索的内容。

考虑到上述原因，在Rust结合ESP32系列的开头，我打算重复我之前[STM32系列](https://apollolabsblog.hashnode.dev/series/stm32f4-embedded-rust-hal)中的许多示例。这样做可以在深入探讨更复杂的示例之前进行初步比较。我将专注于硬件抽象层（HAL）的工作，特别是专注于[esp32c3-hal](https://docs.rs/esp32c3-hal/latest/esp32c3_hal/)。在本文中，我将从GPIO外设着手。我们会了解如何在HAL层配置GPIO，读取输入信号，并控制输出。这里的示例是基础的闪烁程序的一个更高级版本。

### 📚 背景知识
为了理解本文内容，你需要具备以下知识：

* 具有Rust编程的基础知识。
* 熟悉使用Rust创建嵌入式应用的基本模板。

### 💾 软件配置
本文展示的所有代码都可以在[apollolabs ESP32C3](https://github.com/apollolabsdev/ESP32C3) Git仓库中找到。请注意，如果Git仓库中的代码有所不同，这可能是为了提高代码质量或适应任何硬件抽象层（HAL）或Rust的更新而做的修改。

另外，基于Wokwi的完整的项目（包括代码和模拟）可以在[这里](https://wokwi.com/projects/362145427195567105)访问。

### 🛠 硬件搭建
#### 所需材料
* [ESP32-C3-DevKitM](https://docs.espressif.com/projects/esp-idf/en/latest/esp32c3/hw-reference/esp32c3/user-guide-devkitm-1.html)
![](https://cdn.hashnode.com/res/hashnode/image/upload/v1681796942083/da81fb7b-1f90-4593-a848-11c53a87821d.jpeg?auto=compress,format&format=webp)

* 任意颜色的LED灯

* 限流电阻

* 按键

#### 🔌 连接方式
📝 注意

    *所有的连接细节也展示在了[Wokwi示例]中。*

连接内容如下：

* LED的阳极通过一个电阻连接到开发板的第4号引脚。这个引脚将作为输出端使用。LED的阴极连接到地线。

* 按键的一端连接到开发板的第0号引脚。这个引脚将作为输入端使用。按键的另一端连接到地线。

## 👨‍🎨 软件设计

在本文中开发的应用中，我的目标是基于按钮按压来切换LED灯闪烁的几种频率。也就是说，每次我按下板载按钮，LED灯的开关速度就会变化。在这一节中，我将重点讨论应用程序算法的设计，而不是配置层面。

在这个设计中，我做了一些假设：

* 仅使用GPIO外围设备。即便是为了延时，我也不会使用任何定时器外围设备。

* 设计将采用基于轮询的方法（而非中断）来检测按钮的按压事件。这在算法上会带来一些挑战（稍后将详细解释）。

关于第二个假设，值得注意的是，使用中断会让整个过程更为便捷。但我之所以不使用中断，有两个原因：一是中断通常被视为一个更高级的概念；二是与C或C++的传统方法相比，在Rust中实现中断要更具挑战性。因此，为了让本文尽可能简单，便于理解基本概念，我决定不采用中断。未来，我可能会专门撰写一篇文章，介绍同一应用的基于中断的实现方法。

下面，让我们尝试用流程图来描述我们的算法。这里是一种可能的方法：
![](https://cdn.hashnode.com/res/hashnode/image/upload/v1655396060993/OAkNDSj1e.png?auto=compress,format&format=webp)

现在来分析一下这里的情况。在配置GPIO引脚之后，我将初始化一个延迟值（或变量），这是我用来以算法方式创建延迟的值。此外，我也会初始化LED的输出状态（开或关）。随后，我将计数变量设置为零，并进入一个循环，不断增加计数，直到达到我设定的延迟值。在延迟循环中，我也会轮询检查按钮是否被按下。如果按钮在任何时候被按下，我需要减小延迟值，从而增加闪烁的频率。但同时，我也必须确保新的延迟值不会变成负数。因此，如果延迟值低于某个阈值，我将把它重置为最初的值。检查完成后，我就可以切换LED的输出，并重新开始初始化延迟循环。

🚨 重要说明：

    1️⃣ 注意我必须在延迟循环中检查按钮按压。这是因为如果等到循环结束，尤其在延迟较长时，可能会错过按钮按压。这也是我之前提到中断会更便捷的原因。通过中断，我可以立即暂停当前操作来响应按钮按压事件。

    2️⃣ 由于我是用算法来创建延迟，注意这段代码在不同设备之间是不可移植的，也不具备可扩展性。这意味着你会在不同设备上看到不同的延迟，这取决于设备参数和代码的响应性。这通常是怎么解决的？一般来说，延迟不是通过软件创建的，而是通过硬件资源，比如计时器/计数器来实现的。

接下来，我们将着手实现这个算法。

## 👨‍💻 代码实现

📥 引入Crate库 
在本次实现中，需要引入以下三个Crate库：

* riscv_rt crate，它提供RISC-V微控制器的启动代码和最小运行时环境。

* esp_backtrace crate，用于定义程序出现异常时的行为。

* esp32c3_hal crate，用来导入ESP32C3设备的硬件抽象层。

```rust
use esp32c3_hal::{clock::ClockControl, pac::Peripherals, prelude::*, timer::TimerGroup, Rtc, Delay, IO};
use esp_backtrace as _;
use riscv_rt::entry;
```

📝 注意

    每个Crate库的导入都需要在Cargo.toml文件中有一个相应的依赖配置。

### 🎛 外设配置代码 
在我们的应用程序代码之前，需要通过以下步骤来配置外设：

1️⃣ 获取设备外围设备的句柄：在嵌入式Rust编程中，作为单例设计模式的一部分，我们首先需要获取PAC层级的设备外围设备。这通过使用take()方法来完成。这里，我创建了一个名为dp的设备外围设备处理器，步骤如下：

```rust
let dp = Peripherals::take().unwrap();
```

2️⃣ 禁用看门狗：ESP32C3默认启用了看门狗，我们需要将其禁用。如果不这么做，设备会不断重置。这里不详细展开，但需要说明的是，看门狗需要应用程序软件定期进行“踢”操作以防止重置。虽然这超出了本示例的范围，但为避免这个问题，需要加入以下代码：

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

3️⃣ 实例化并创建IO的句柄：我们需要将LED引脚配置为push-pull输出，并获取该引脚的句柄，以便我们能够控制它。我们还需要获取按钮输入引脚的句柄。在获取LED和按钮的句柄之前，我们需要创建一个IO结构的实例。IO结构的实例是HAL设计的结构，它允许我们访问所有gpio引脚，从而为单个引脚创建句柄。这与其他HAL中使用的split方法类似（更多详情见[这里](https://apollolabsblog.hashnode.dev/demystifying-rust-embedded-hal-split-and-constrain-methods)）。我们通过在IO结构上调用new()实例方法来实现，如下：
    
```rust
let io = IO::new(peripherals.GPIO, peripherals.IO_MUX);
```

请注意，new方法需要传递GPIO和IO_MUX外围设备。

4️⃣ 获取LED的句柄并配置为输出：如前所述，LED连接到4号引脚（gpio4）。因此，我们需要为LED引脚创建一个句柄，该引脚使用into_push_pull_output()方法配置为push-pull输出。我们将这个句柄命名为led，并按如下方式进行配置：

```rust
let mut led = io.pins.gpio4.into_push_pull_output();
```

5️⃣ 获取并配置输入按钮的句柄：如前所述，按钮连接到0号引脚（gpio0）。此外，在按下状态，按钮将会接地。对于按钮未按下的状态，需要包含一个上拉电阻，以便引脚变为高电平。可以使用into_pull_up_input()方法为引脚配置内部上拉电阻，如下所示：

```rust
let button = io.pins.gpio0.into_pull_up_input();
```

请注意，与LED输出不同，这里的按钮句柄不需要是可变的，因为我们只会对其进行读取。

### 📱 应用程序代码
按照之前描述的设计，我首先需要初始化一个延迟变量del_var，并设置LED的输出。del_var需要是可变的，因为它将在延迟循环中被修改。我还选择默认将初始输出电平设置为低。使用之前提到的Pin方法，我用set_low()方法来实现这一点。需要注意的是，set_low()方法返回一个Result类型，这需要被解包处理。

```rust
    // Create and initialize a delay variable to manage delay loop
    let mut del_var = 10_0000_u32;

    // Initialize LED to on or off
    led.set_low().unwrap();
```

在程序循环中，我创建了一个循环，持续运行直到达到del_var的值。正如设计部分所述，如果循环自然结束，则del_var保持不变。否则，在延迟期间的任何时刻，如果按键被按下，我可以使用is_low()方法（该方法同样返回一个Result类型，需要解包）来检测。此时，我会将del_var减少2_5000_u32。如果del_var的值最终低于2_5000_u32，我会将其恢复到初始的10_0000_u32。延迟完成后，我使用toggle()方法来切换led的状态，这也是GpioPin类型提供的方法之一。

为什么我选择使用10_0000_u32和2_5000_u32这两个值？这其实是通过试错法得出的。我不断尝试不同的值，直到找到令LED以令人满意的方式闪烁的值。正如之前提到的，因为我是通过算法来创建延迟，所以延迟的持续时间实际上取决于使用的平台。

```rust
// Application Loop
loop {
   for _i in 1..del_var {
      // Check if button got pressed
      if button.is_low().unwrap() {
         // If button pressed decrease the delay value
         del_var = del_var - 2_5000_u32;
         // If updated delay value = zero reset to start value
         if del_var < 2_5000 {
            del_var = 10_0000_u32;
         }
       }
    }
    // Toggle LED
    led.toggle();
}

```

🚨 重要说明：

    当你运行代码时，会看到LED开始闪烁，但你可能会注意到一些异常行为。闪烁频率似乎会以随机的顺序不断改变。这是因为机械按键上存在所谓的“抖动”效应。在Wokwi中，如果你想的话，可以选择消除这种抖动效应。更多细节请查看下面的进一步实验/想法部分。

## 📱 完整的应用程序代码
以下是本文描述的实现的完整代码。你还可以在[apollolabs ESP32C3](https://github.com/apollolabsdev/ESP32C3)的Git仓库中找到此完整项目及其他项目。Wokwi项目也可以在[这里](https://wokwi.com/projects/362145427195567105)访问。

```rust
#![no_std]
#![no_main]

use esp32c3_hal::{clock::ClockControl, pac::Peripherals, prelude::*, timer::TimerGroup, Rtc, Delay, IO};
use esp_backtrace as _;
use riscv_rt::entry;

#[entry]
fn main() -> ! {
    // Take Peripherals, Initialize Clocks, and Create a Handle for Each
    let peripherals = Peripherals::take().unwrap();
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


    // Create and initialize a delay variable to manage delay loop
    let mut del_var = 10_0000_u32;

    // Initialize LED to on or off
    led.set_low().unwrap();

    // Application Loop
    loop {
    for _i in 1..del_var {
        // Check if button got pressed
        if button.is_low().unwrap() {
            // If button pressed decrease the delay value
            del_var = del_var - 2_5000_u32;
            // If updated delay value reaches zero then reset it back to starting value
            if del_var < 2_5000 {
                del_var = 10_0000_u32;
            }
        }
    }
        // Toggle LED
        led.toggle();

    }
}
```

## 🔬 进一步的实验与创意思考

* 大多数机械式按键需要进行所谓的去抖动处理。按键在被按下时会有“抖动”效应，这可能导致检测到多次按压。因此，需要进行去抖动处理，这可以通过硬件或软件来实现。使用示波器观察引脚输出可以很好地查看这种效应。查看[Jack Ganssle](http://www.ganssle.com/debouncing.htm)的页面了解更多关于按键抖动及消除这种效应的算法。如果你仔细寻找，甚至可能会找到一个可以用于按键去抖动的crate 😉

* 作为Rust的练习，尝试编写相同的代码，但将loop_delay函数体整合到应用程序循环中。

* 连接多个LED输出，创建不同的LED灯光模式。你可以使用按钮来切换不同的模式。

* 将LED替换为蜂鸣器，在输出端生成不同的音调。你可以使用多个按钮输入来增加或减少音调的频率。

## 总结
在这篇文章中，我们利用ESP32C3微控制器的GPIO外设，创建了一个LED控制应用程序。所有的代码都是在硬件抽象层（HAL）上，使用esp32c3-hal来编写的。
 