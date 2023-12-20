# 用ESP32和WS2812打造一个恐龙灯
我在Thingiverse上找到一个很有趣的项目，决定换一种芯片并用Rust编程语言来实现它

【Part 1原文】: https://nereux.blog/posts/esp32-ws2812-dino-light/  
【Part 2原文】: https://nereux.blog/posts/esp32-ws2812-dino-light-2/

**(Part 1)**  
我在thingiverse上发现了这个项目，决定用Rust编程和ESP32芯片来制作它。
这是它的最终成果：
![](https://nereux.blog/posts/esp32-ws2812-dino-light/images/dino_light.jpg)

## 硬件部分介绍
我建议选择像素密度低于144像素/米的WS2812灯条，因为高密度焊接对于像我这样焊接经验较少的人来说相当困难。至于ESP32，我推荐使用ESP32-C型号，这款芯片内置了RISC-V处理器，与基于Xtensa架构的板子相比，对Rust语言的支持更好。

如果你的微控制器有5V输出端口，我们会将其连接到LED灯条的5V输入端，否则你需要另外找到供应5V电源的方法。同时，将微控制器的GND（地）引脚连接到LED的GND引脚。至于数据引脚，查阅你主板的引脚布局图。以我的情况为例，GPIO15引脚很合适，因为它不参与内部闪存操作，而且与SPI（串行外设接口）协同工作得很好。

## 软件部分介绍
这一整个部分现在不再必要了。你可以直接使用[espup](https://github.com/esp-rs/espup)，它会帮你设置好所需的一切。

设置工具链和编译器
我在设置这部分时遇到了很多困难，但遵循这篇文章的指引，你应该能避免不少挫折。

如果你像上文提到的那样选择了ESP32-C，那么几乎可以省去所有这些设置步骤。

使用以下命令下载rust-build：

```bash
git clone https://github.com/esp-rs/rust-build
```

然后通过这个命令来安装它：

```bash
cd rust-build; ./install-rust-toolchain.sh
```

脚本执行完毕后，按照屏幕终端端的说明操作。

现在用rustup default esp命令（除非你使用的是C型号，在这种情况下你可以使用Rust nightly版本）来设置默认编译环境，并安装cargo install cargo-espflash（用于固件烧录）和cargo pio（用于解码堆栈追踪）。

```bash
curl -fsSL https://raw.githubusercontent.com/platformio/platformio-core-installer/master/get-platformio.py -o get-platformio.py
python3 get-platformio.py
cargo install cargo-pio
```

通过以下步骤进行编译和固件烧录（在Windows操作系统上，你需要自行查找相应的命令）：

```bash
cargo espflash --release /dev/ttyUSB0
```
如果使用cargo espflash进行烧录失败，你可以尝试在执行命令时按下启动按钮，这种方法对我来说一直有效。

我们可以通过以下命令来监控串口输出：

cargo pio espidf monitor /dev/ttyUSB0

## 代码部分
在Cargo.toml文件中，我们需要包含以下依赖项：

```toml
[dependencies]
# For the ESP32
esp32 = "0.11.0"
esp32-hal = "0.3.0"
xtensa-lx = { version = "0.4.0", features = ["lx6"] }
xtensa-lx-rt = { version = "0.7.0", optional = true, features = ["lx6"] }
# For the LEDs
ws2812-spi = "0.4.0"
smart-leds = "0.3.0"
```

在main.rs文件的起始部分，我们需要添加一些特殊的代码，因为我们在使用微控制器，并且没有为其准备的标准库（[虽然有一个专门为ESP32设计的标准库项目](https://github.com/ivmarkov/rust-esp32-std-demo)）。

```rust
#![no_std]
#![no_main]
```

现在来看main函数：

```rust
#[entry]
fn main() -> ! {
    loop{
        
    }
}
```

我们需要使用#[entry]属性来定义程序的起始点，"!"表示该函数不会返回，这也是为什么我们需要一个无限循环。

接下来是实际用于设置代码：

```rust
#[entry]
fn main() -> ! {
    let dp = target::Peripherals::take().unwrap();
    let (_, dport_clock_control) = dp.DPORT.split();

    let clkcntrl = ClockControl::new(
        dp.RTCCNTL,
        dp.APB_CTRL,
        dport_clock_control,
        XTAL_FREQUENCY_AUTO,
    )
    .unwrap();
    let (clkcntrl_config, _) = clkcntrl.freeze().unwrap();
    let pins = dp.GPIO.split();
    let data_out = pins.gpio15.into_push_pull_output();
    // We need SPI for the WS2812 library
    let spi: SPI<_, _, _, _> = SPI::<esp32::SPI2, _, _, _, _>::new(
        dp.SPI2,
        spi::Pins {
            sclk: pins.gpio14,
            sdo: data_out,
            sdi: Some(pins.gpio25),
            cs: None,
        },
        spi::config::Config {
            baudrate: 3.MHz().into(),
            bit_order: spi::config::BitOrder::MSBFirst,
            data_mode: spi::config::MODE_0,
        },
        clkcntrl_config,
    )
    .unwrap();
    let ws = Ws2812::new(spi);
    loop{
    }
}
```

现在我们创建一些结构体，方便对灯条进行控制：

```rust
const NUM_LEDS: usize = 23;

struct LightData {
    leds: [RGB8; NUM_LEDS],
}
struct Strip {
    ws: Ws2812<SPI<SPI2, Gpio14<Unknown>, Gpio15<Output<PushPull>>, Gpio25<Unknown>>>,
    data: LightData,
    brightness: u8,
}
```

还有一些函数用于操控内部数据：
```rust
impl LightData {
    fn empty() -> Self {
        Self {
            leds: [RGB8::new(0, 0, 0); NUM_LEDS],
        }
    }
    fn write_to_strip(
        &self,
        strip: &mut Ws2812<SPI<SPI2, Gpio14<Unknown>, Gpio15<Output<PushPull>>, Gpio25<Unknown>>>,
    ) {
        strip.write(self.leds.iter().cloned()).unwrap();
    }
    
    fn get_led(&self, index: usize) -> RGB8 {
        self.leds[index]
    }
    fn set_color_all(&mut self, color: RGB8) {
        for i in 0..NUM_LEDS {
            self.set_color(i, color);
        }
    }
    fn set_red(&mut self, index: usize, red: u8) {
        self.leds[index].r = red;
    }
    fn set_green(&mut self, index: usize, green: u8) {
        self.leds[index].g = green;
    }
    fn set_blue(&mut self, index: usize, blue: u8) {
        self.leds[index].b = blue;
    }
    fn set_color(&mut self, led: usize, color: RGB8) {
        self.leds[led] = color;
    }
}
impl Default for LightData {
    fn default() -> Self {
        Self {
            leds: [RGB8::new(STEPS, STEPS, STEPS); NUM_LEDS],
        }
    }
}
impl Strip {
    fn write(&mut self) {
        self.data.write_to_strip(&mut self.ws);
    }
    fn set_color(&mut self, color: RGB8, index: usize) {
        self.data.set_color(index, color);
        self.write();
    }
    fn set_solid(&mut self, color: RGB8) {
        self.data.set_color_all(color);
        self.write();
    }
    fn get_brightness(&mut self) {
        self.data.get_brightness();
    }
}
```

接着，我们可以将这些代码添加到main函数中：

```rust
    let mut strip = Strip {
        ws,
        data: LightData::from_gradient(RGB8::new(40, 0, 0), RGB::new(0, 0, 40)),
        brightness: 10,
    };
    loop {
        strip.set_solid(RGB8::new(25, 0, 0));
        delay(40_000_000);
        for i in 0..NUM_LEDS {
            strip.set_color(RGB8::new(0, 0, 25), i);
            delay(40_000_000);
        }
    }
```

这会将将LED初始设置为红色，并逐个转换为蓝色

最后，我们需要一个#[panic_handler]，对于这个示例来说，以下代码就足够了：

```rust
#[panic_handler]
fn panic(info: &PanicInfo) -> ! {
    dprintln!("\n\n*** {:?}", info);
    loop {}
}
```

现在，我们可以通过以下方式进行固件烧录：
```bash
cargo espflash --release /dev/ttyUSB0
```

完整的源代码可以在[这里](https://github.com/esp-rs/esp32-hal/blob/master/examples/leds.rs)找到

如果你读到了这里：非常感谢，这对我来说非常重要！

**(Part 2)**  
嗨，已经有一段时间了。自从第一部分（去年）以来，esp-rs生态系统有了显著改进，包括支持无标准库的WiFi和异步编程等许多新特性。你可能会问：“异步编程？在嵌入式设备上？”确实如此，而且实际上相当棒。

##介绍Embassy
[Embassy](https://embassy.dev/)是一个为裸机Rust设计的异步运行时环境，它免除了像FreeRTOS（或者TockOS, [最近也开始支持esp32c3](https://www.tockos.org/)）这类RTOS的需要。虽然它还在初期开发阶段，但已经足够实用。然而，作为前沿技术，它还有一些不足之处。本质上，它类似于为裸机设计的tokio或async-std。你有一个执行器，它运行你的任务，如果代码中出现await，它会暂停该任务并轮询所有任务，直到有任务准备好继续执行。遗憾的是，目前这些任务还不支持泛型，但我在这个项目中找到了一个变通方法。

## 编写ESP32程序

我花了相当长的时间来理解这一点，但是esp-rs matrix频道中的人们给了我很大的帮助。这里是许多必需的依赖项及其特性：

```toml
[dependencies]
# Much, much esp-stuff, this time with async support
hal = { package = "esp32-hal", version="0.11.0", features = [
    "embassy",
    "async",
    "rt",
    "embassy-time-timg0",
] }
esp-backtrace = { version = "0.6.0", features = [
    "esp32",
    "panic-handler",
    "exception-handler",
    "print-uart",
] }
esp-println = { version = "0.4.0", features = ["esp32", "log"] }
esp-alloc = { version = "0.2.0", features = ["oom-handler"] }
esp-wifi = { git = "https://github.com/esp-rs/esp-wifi", features = [
    "esp32",
    "esp32-async",
    "async",
    "embedded-svc",
    "embassy-net",
    "wifi",
] }
embedded-svc = { version = "0.23.1", default-features = false, features = [] }
embedded-io = "0.4.0"
# Embassy, our async runtime
embassy-sync = "0.1.0"
embassy-time = { version = "0.1.0", features = ["nightly"] }
embassy-executor = { package = "embassy-executor", git = "https://github.com/embassy-rs/embassy/", rev = "cd9a65b", features = [
    "nightly",
    "integrated-timers",
] }
embassy-net-driver = { git = "https://github.com/embassy-rs/embassy", rev = "26474ce6eb759e5add1c137f3417845e0797df3a" }
embassy-net = { git = "https://github.com/embassy-rs/embassy", rev = "26474ce6eb759e5add1c137f3417845e0797df3a", features = [
    "nightly",
    "tcp",
    "udp",
    "dhcpv4",
    "medium-ethernet",
] }
futures-util = { version = "0.3.17", default-features = false }

# Serde_json without needing allocations
serde = { version = "1.0", default-features = false }
serde-json-core = "0.5.0"
# LED strip driver, this time without even needing SPI
smart-leds = "0.3.0"
esp-hal-smartled = {version = "0.1.0", features = ["esp32"]}
# For global channel(Workaround for lifetime restrictions on tasks) 
lazy_static = { version = "1.4.0", features = ["spin_no_std"] }

# This is necessary in order for WIFI to work
[profile.dev.package.esp-wifi]
opt-level = 3
[profile.release]
opt-level = 3
lto="off"
```

通过以下命令安装最新版本的espflash：cargo install espflash --git https://github.com/esp-rs/espflash。 我们还需要在.cargo/config.toml文件中添加以下内容：

```toml
[target.xtensa-esp32-none-elf]
runner = "espflash flash --monitor"

[build]
rustflags = [
  "-C", "link-arg=-Tlinkall.x",
  # In order for esp-wifi to work we need this linker argument  
  "-C", "link-arg=-Trom_functions.x",
  "-C", "link-arg=-nostartfiles",
]
# ESP32 fist-gen
target = "xtensa-esp32-none-elf"

[unstable]
# Strictly speaking alloc was not necessary for this project but it's really useful if you ever want to use a heap
build-std = ["alloc", "core"]
```

最后，为了避免每次使用cargo时都要输入cargo +esp，可以直接使用cargo：

```bash
echo "[toolchain]
channel = \"esp\"" > rust-toolchain.toml
```

最后，我们需要在main.rs文件中添加以下内容，以分配我们的堆内存：

```rust
#![no_std]
#![no_main]
#![feature(type_alias_impl_trait)]
extern crate alloc;

use alloc::vec;
use esp_backtrace as _;
use esp_println::println;
use hal::entry;
#[global_allocator]
static ALLOCATOR: esp_alloc::EspHeap = esp_alloc::EspHeap::empty();

fn init_heap() {
    const HEAP_SIZE: usize = 2 * 1024;

    extern "C" {
        static mut _heap_start: u32;
    }
    unsafe {
        let heap_start = &_heap_start as *const _ as usize;
        ALLOCATOR.init(heap_start as *mut u8, HEAP_SIZE);
    }
}
#[entry]
fn main() -> ! {
    init_heap();
    // And we can use the heap!
    println!("Vec element 0: {}", vec![1, 2, 3][0]);
    loop {}
}
```

当我们执行它时，我们看到了这个：

```bash
Vec element 0: 1
```

这意味着我们可以在这个微型微控制器上使用堆内存！在此之后不久，它会随着以下输出进行重置：
```bash
ets Jun  8 2016 00:22:57

rst:0x10 (RTCWDT_RTC_RESET),boot:0x13 (SPI_FAST_FLASH_BOOT)
configsip: 0, SPIWP:0xee
clk_drv:0x00,q_drv:0x00,d_drv:0x00,cs0_drv:0x00,hd_drv:0x00,wp_drv:0x00
mode:DIO, clock div:2
load:0x3fff0030,len:7024
0x3fff0030 - _stack_end_cpu0
    at ??:??
load:0x40078000,len:15400
0x40078000 - ets_delay_us
    at ??:??
load:0x40080400,len:3816
0x40080400 - _init_end
    at ??:??
entry 0x40080648
0x40080648 - _ZN14esp_hal_common9interrupt6xtensa8vectored17handle_interrupts17hb0c80caf10c2f321E
    at ??:??
## The rest normal boot sequence follows after this
```
这确实很奇怪，因为我们并没有使用任何中断...

如果有人能帮我解决这个问题就好了...

***酷酷的鸭子说** ![](https://nereux.blog/static/cool_duck.svg)  
就好像你的代码因为没有执行任何实际有用的操作而被重置一样。*

看来你是对的。

***酷酷的鸭子说** ![](https://nereux.blog/static/cool_duck.svg)  
在嵌入式设备中，经常会有一个看门狗定时器，如果不定期给它“喂食”，为了避免死锁，它会自动重置设备。
*

明白了，我需要找到一种方法来解决它。

  
***酷酷的鸭子说** ![](https://nereux.blog/static/cool_duck.svg)  
确实，这是一种合适的处理方式，或者你也可以选择将其禁用。取决于你的选择。*

真是一只聪明的鸭子。我现在会先禁用它（不仅是因为我懒，而且因为esp-wifi在他们的[示例](https://github.com/esp-rs/esp-wifi/blob/main/examples-esp32/examples/embassy_dhcp.rs)中也是这么做的）。在仔细阅读示例后，我们发现以下操作可以禁用看门狗：

```rust
#[entry]
fn main() -> ! {
    // All peripherals of our chip
    let peripherals = Peripherals::take();
    // Take the Systemparts, containing the clock control, cpo_control and even radio_clock_control, which we're gonna get to later
    let mut system = peripherals.DPORT.split();
    let clocks = ClockControl::configure(system.clock_control, CpuClock::Clock240MHz).freeze();
    let mut rtc = Rtc::new(peripherals.RTC_CNTL);
    rtc.rwdt.disable();
}
```

现在让我们来实现WI-FI功能。应该不会太难，对吗？

```rust
// Straight up copied from the dhcp example
const SSID: &str = env!("SSID");
const PASSWORD: &str = env!("PASSWORD");

macro_rules! singleton {
    ($val:expr) => {{
        type T = impl Sized;
        static STATIC_CELL: StaticCell<T> = StaticCell::new();
        let (x,) = STATIC_CELL.init(($val,));
        x
    }};
}
fn main() -> ! {
    init_heap();

    let peripherals = Peripherals::take();
    let mut system = peripherals.DPORT.split();
    let clocks = ClockControl::configure(system.clock_control, CpuClock::Clock240MHz).freeze();
    let mut rtc = Rtc::new(peripherals.RTC_CNTL);
    rtc.rwdt.disable();

    let timer = TimerGroup::new(peripherals.TIMG1, &clocks).timer0;
    initialize(
        timer,
        Rng::new(peripherals.RNG),
        system.radio_clock_control,
        &clocks,
    )
        .unwrap();
    // Now we get to use the radio peripherals!
    let (wifi, _) = peripherals.RADIO.split();
    let (wifi_interface, controller) = esp_wifi::wifi::new_with_mode(wifi, WifiMode::Sta);

    let timer_group0 = TimerGroup::new(peripherals.TIMG0, &clocks);
    embassy::init(&clocks, timer_group0.timer0);

    let config = Config::Dhcp(Default::default());

    let seed = 123456; // Change this for your own project

    // Init network stack
    let stack = &*singleton!(Stack::new(
        wifi_interface,
        config,
        singleton!(StackResources::<3>::new()),
        seed
    ));

    // Initialize the embassy executor
    let executor = EXECUTOR.init(Executor::new());
    executor.run(|spawner| {
        spawner.spawn(connection(controller)).ok();
        spawner.spawn(net_task(stack)).ok();
    });
}
/// I really don't know why we need this but it's necessary
#[embassy_executor::task]
async fn net_task(stack: &'static Stack<WifiDevice<'static>>) {
    stack.run().await
}
#[embassy_executor::task]
async fn connection(mut controller: WifiController<'static>) {
    println!("start connection task");
    println!("Device capabilities: {:?}", controller.get_capabilities());
    loop {
        match esp_wifi::wifi::get_wifi_state() {
            WifiState::StaConnected => {
                // wait until we're no longer connected
                controller.wait_for_event(WifiEvent::StaDisconnected).await;
                println!("Wifi disconnected!");
                Timer::after(Duration::from_millis(5000)).await
            }
            _ => {}
        }
        if !matches!(controller.is_started(), Ok(true)) {
            let client_config = Configuration::Client(ClientConfiguration {
                ssid: SSID.into(),
                password: PASSWORD.into(),
                ..Default::default()
            });
            controller.set_configuration(&client_config).unwrap();
            println!("Starting wifi");
            controller.start().await.unwrap();
            println!("Wifi started!");
        }
        println!("About to connect...");

        match controller.connect().await {
            Ok(_) => println!("Wifi connected!"),
            Err(e) => {
                println!("Failed to connect to wifi: {e:?}");
                Timer::after(Duration::from_millis(5000)).await
            }
        }
    }
}
```

现在，当我们通过控制台设置SSID和PASSWORD后，我们就可以连接到wifi了！
```bash
export SSID="your SSID" PASSWORD="your password"
cargo run -q --release
```

现在我们可以连接并使用我们的LED灯条了。回想上一部分，我们使用了SPI（串行外设接口）来实现这一点，但那相当繁琐。幸运的是，Espressif Rust团队开发了esp-hal-smartled，它使我们能够使用RMT输出通道，同时保留了smart-leds库的便捷功能。

```rust
// in our main
let io = IO::new(peripherals.GPIO, peripherals.IO_MUX);
let pulse = PulseControl::new(peripherals.RMT, &mut system.peripheral_clock_control).unwrap();
let mut led = <smartLedAdapter!(23)>::new(pulse.channel0, io.pins.gpio33);
// Turn the lights off by default
led.write([RGB8::default(); 23].into_iter()).unwrap();
```

我们可以为此创建一个单独的任务，它总是等待一个通道接收到新消息：

```rust
// Since lifetime limitations don't allow us to pass a receiver and sender to the webserver and
// led_task threads, this is an acceptable workaround. This channel is over a
// CriticalsectionRawMutex, since lazy_static requires the Mutex to be Thread-safe. In there we
// store up 3 RGB8 values and when full, sending will (a)wait until a message is received.
lazy_static! {
    static ref CHANNEL: Channel<CriticalSectionRawMutex, OwnRGB8, 3> =
        embassy_sync::channel::Channel::new();
}
#[embassy_executor::task]
// This is a really long type, but since embassy doesn't support generics yet, we have to specify the type fully
async fn led_task(
    mut leds: SmartLedsAdapter<
        ConfiguredChannel0<
            'static,
            GpioPin<
                hal::gpio::Unknown,
                Bank1GpioRegisterAccess,
                DualCoreInteruptStatusRegisterAccessBank1,
                InputOutputAnalogPinType,
                Gpio33Signals,
                33,
            >,
        >,
        553,
    >,
) {
    loop {
        println!("Waiting for color...");
        let receiver = CHANNEL.receiver();
        let color: RGB8 = receiver.recv().await.into();
        // If you send an array of colours you could also color the LEDs differently but in what is typical Go fashion
        // that is left as an exercise to the reader(See https://fasterthanli.me/articles/lies-we-tell-ourselves-to-keep-using-golang#go-as-a-prototyping-starter-language)
        leds.write([color; 23].into_iter()).unwrap();
    }
}
```

最后，我们编写了服务器代码。得益于embassy和esp-hal提供的抽象，构建一个裸机硬件上的HTTP服务器变得相当简单。

```rust
#[embassy_executor::task]
async fn task(stack: &'static Stack<WifiDevice<'static>>) {
    // Wait until Wifi is connected
    loop {
        if stack.is_link_up() {
            break;
        }
        Timer::after(Duration::from_millis(500)).await;
    }
    // Wait for DHCP to get an IP address
    println!("Waiting to get IP address...");
    loop {
        if let Some(config) = stack.config() {
            println!("Got IP: {}", config.address);
            break;
        }
        Timer::after(Duration::from_millis(500)).await;
    }
    println!("Starting web server...");
    // Get our sender
    let sender = CHANNEL.sender();
    let mut rx_buffer = [0; 4096];
    let mut tx_buffer = [0; 4096];

    loop {
        if stack.is_link_up() {
            break;
        }
        Timer::after(Duration::from_millis(500)).await;
    }

    let mut socket = TcpSocket::new(stack, &mut rx_buffer, &mut tx_buffer);
    socket.set_timeout(Some(embassy_net::SmolDuration::from_secs(10)));
    loop {
        println!("Wait for connection...");
        // We can easily bind our socket to port 80, and accept connections
        let r = socket
            .accept(IpListenEndpoint {
                addr: None,
                port: 80,
            })
            .await;
        println!("Connected...");

        if let Err(e) = r {
            println!("connect error: {:?}", e);
            continue;
        }

        use embedded_io::asynch::Write;

        let mut buffer = [0u8; 512];
        let mut pos = 0;
        loop {
            match socket.read(&mut buffer).await {
                Ok(0) => {
                    println!("read EOF");
                    break;
                }
                Ok(len) => {
                    let to_print =
                        unsafe { core::str::from_utf8_unchecked(&buffer[..(pos + len)]) };

                    println!("read {} bytes: {}", len, to_print);
                    if to_print.starts_with("POST") {
                        let r = socket.write_all(b"HTTP/1.0 200 OK\r\n\r\n").await;
                        if let Err(e) = r {
                            println!("write error: {:?}", e);
                        }
                        let r = socket.flush().await;
                        if let Err(e) = r {
                            println!("flush error: {:?}", e);
                        }
                        // I couldn't find a library for no_std http parsing but this works as well and is fairly simple
                        if let Some(body) =
                            to_print.lines().into_iter().find(|l| l.starts_with("{"))
                        {
                            println!("Body: {}", body);
                            if let Ok((color, _)) = serde_json_core::from_str::<OwnRGB8>(body) {
                                println!("Got color: {:?}", color);
                                sender.send(color).await;
                            }
                        }
                    }

                    pos += len;
                }
                Err(e) => {
                    println!("read error: {:?}", e);
                    break;
                }
            };
            Timer::after(Duration::from_millis(100)).await;
            socket.close();
            Timer::after(Duration::from_millis(100)).await;
            socket.abort();
        }
    }
}
```

现在我们可以通过以下curl POST请求来控制它：

```bash
curl -v -d '{"r":0,"g":0,"b":200}' http://<ESP-IP-ADDRESS>
```

***酷酷的鸭子说**  
LED灯变成了蓝色，真是太棒了！我确信你也能找出如何让它变成红色和绿色的方法。*

我们可以用这个项目做更多的事，比如使用embassy计时器来实现彩虹色效果，接受GET请求来展示一个简单的web客户端（已在仓库中实现），或者允许用户按照特定的模式点亮LED灯。

这就是这篇文章的内容。我希望你喜欢它，并且也许还学到了一些东西。我想让我未来的文章更加深入和技术性，但我不确定是否能做到这一点而不让文章变得太长。如果你有任何建议，欢迎在仓库中提出问题或通过[Mastodon](https://infosec.exchange/@Nereuxofficial)联系我。如果你想支持我和我的工作，可以在[这里](https://github.com/sponsors/Nereuxofficial)做到。

## 代码
和往常一样，这个项目的仓库是免费公开的，地址在这里：[https://github.com/Nereuxofficial/nostd-wifi-lamp](https://github.com/Nereuxofficial/nostd-wifi-lamp)

该仓库还支持一些非常棒的功能，如Wokwi，它提供了一个基于(esp-template](https://github.com/esp-rs/esp-template)创建的模拟ESP32-C3环境。

特别感谢：
[bjoernQ](https://github.com/bjoernQ)修复了一个栈溢出到堆的错误，我之前一直不清楚为什么程序会崩溃。

[esp-rs](https://github.com/esp-rs)团队为ESP32微控制器提供了出色的工具和支持。


