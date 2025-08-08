# Embedded-HAL :跨平台生态


[Embedded-HAL](https://github.com/rust-embedded/embedded-hal/tree/master)（Hardware Abstraction Layer，硬件抽象层）是Rust嵌入式生态系统的基础，它定义了一系列标准trait（特质），用于抽象常见嵌入式外设的操作接口。其核心目标是：

- 提供硬件无关的外设抽象接口
- 允许编写可在不同芯片和平台上运行的通用驱动程序
- 建立一个统一的嵌入式，乃至系统软件开发生态，促进代码重用

## 与传统C/C++嵌入式库的区别
传统嵌入式开发中，不同厂商和平台的HAL库通常互不兼容：
```c
// STM32 HAL库示例
HAL_StatusTypeDef HAL_GPIO_WritePin(GPIO_TypeDef *GPIOx, uint16_t GPIO_Pin, GPIO_PinState PinState);

//MSP432库示例
void GPIO_setOutputHighOnPin(uint_fast8_t selectedPort,uint_fast16_t selectedPins);

```
这种差异导致：
- 驱动程序必须为每个平台重写
- 代码无法轻松迁移到不同硬件
- 生态系统碎片化，重复劳动

Embedded-HAL通过定义统一的trait接口解决了这一问题，很像Arduino的库比如```digitalWrite```等等，但Rust的trait体系更强大，可以实现更复杂的抽象。

```rust
// Embedded-HAL定义的统一接口
pub trait OutputPin {
    type Error;
    fn set_high(&mut self) -> Result<(), Self::Error>;
    fn set_low(&mut self) -> Result<(), Self::Error>;
}

// 不同平台实现同一接口
// 某平台实现
impl OutputPin for PA0 {
    type Error = Infallible;
    fn set_high(&mut self) -> Result<(), Self::Error> { ... }
    fn set_low(&mut self) -> Result<(), Self::Error> { ... }
}

// 另一个平台实现
impl OutputPin for GpioPin<Output, IO0> {
    type Error = Infallible;
    fn set_high(&mut self) -> Result<(), Self::Error> { ... }
    fn set_low(&mut self) -> Result<(), Self::Error> { ... }
}
```

可以看到不同平台实现同一接口，但接口的实现方式不同，但接口的用法是相同的。

## Trait是什么

[Trait(rust语言官方文档)](https://doc.rust-lang.org/book/ch10-02-traits.html)是Rust语言中定义共享行为的机制，类似于其他语言中的接口或抽象类，但具有更强大的特性：

1. **静态分发**：编译器在编译时确定具体的实现，避免了动态查找实现的运行时开销
2. **关联类型**：允许在trait中定义依赖类型，增强类型安全
3. **泛型约束**：trait可用作泛型约束，提供编译时保证
4. **组合能力**：trait可以组合形成更复杂的抽象

如果Rust应用较少这个概念可能看着比较抽象，在这里我们只需要知道Trait是定义共享行为的机制

C语言也能实现类似接口比如

```c
// C语言模拟接口
typedef struct {
    void (*set_high)(void* ctx);
    void (*set_low)(void* ctx);
    void* ctx;
} gpio_ops_t;

// 使用示例
void toggle_led(gpio_ops_t* gpio) {
    gpio->set_high(gpio->ctx);
    // 延时
    gpio->set_low(gpio->ctx);
}
```

这种方式有以下缺点：
- 运行时开销（函数指针调用）
- 无编译时类型检查
- 接口实现错误只能在运行时发现
- 需要手动管理上下文指针

Rust的Trait结合上面我们提到的类型系统提供了编译时安全保证且避免了动态查找实现的运行时开销。

## Embedded-HAL生态系统
Embedded-HAL生态系统由多个相互关联的crate组成：

1. **embedded-hal**：定义基本阻塞式接口
2. **embedded-hal-async**：异步版本的接口
3. **embedded-hal-nb**：基于nb（非阻塞）模式的接口
4. **embedded-can**：CAN总线专用接口
5. **embedded-io/embedded-io-async**：通用I/O接口

具体实现参考[Embedded-HAL官方储存库](https://github.com/rust-embedded/embedded-hal/tree/master)

使用embedded-hal编写的驱动程序可以在任何实现了相应trait的硬件上运行：
例如我们等会示例中将会用到的SSD1306驱动

```rust
// 通用SSD1306 OLED显示驱动
pub struct Ssd1306<I2C> {
    i2c: I2C,
    // ...其他字段
}

impl<I2C, E> Ssd1306<I2C>
where
    I2C: embedded_hal::i2c::I2c<Error = E>,
{
    // 任何实现了I2c trait的设备都可以使用这个驱动
    pub fn new(i2c: I2C) -> Self {
        // ...初始化代码
    }
    
    pub fn display(&mut self, buffer: &[u8]) -> Result<(), E> {
        // ...使用I2C显示数据
    }
}
```

I2C作为泛型参数，这个驱动可以同时用于：

- STM32系列
- ESP32
- nRF52系列
- RP2040 (Raspberry Pi Pico)
- 等众多实现了I2c trait的硬件

在实际项目中，我们能在不同硬件上使用同一个驱动，这对于产品开发迭代非常有帮助。

## 优雅的错误处理

在embedded-hal代码中可以看到Rust中常见的错误处理方式，比如

```rust
// 使用Result类型返回错误
pub fn write(&mut self, address: u8, data: &[u8]) -> Result<(), Error> {
    // ...
}
```

Rust的错误处理相比C语言有显著优势。让我们通过一个伪代码示例来对比 - 假设我们需要从多个传感器读取数据，处理它们，然后发送结果：

```c
// C语言错误处理 - 使用返回码
typedef enum {
    ERR_OK = 0,
    ERR_I2C_BUS,
    ERR_SENSOR_TIMEOUT,
    ERR_INVALID_DATA,
    ERR_UART_BUSY,
    // ...更多错误类型
} error_t;

// 读取传感器
error_t read_temperature_sensor(float* result) {
    // 初始化I2C总线
    if (i2c_init() != ERR_OK) {
        return ERR_I2C_BUS;
    }
    // 读取传感器
    uint8_t raw_data[4];
    error_t err = i2c_read(SENSOR_ADDR, raw_data, sizeof(raw_data));
    if (err != ERR_OK) {
        return err; // 传播错误
    }
    // 检查数据有效性
    if (raw_data[3] != calculate_checksum(raw_data, 3)) {
        return ERR_INVALID_DATA;
    }
    // 转换数据
    *result = convert_to_temperature(raw_data);
    return ERR_OK;
}
// 使用示例
void process_sensor_data(void) {
    float temperature;
    error_t err = read_temperature_sensor(&temperature);
    // 错误处理 - 常见的if-else分支
    if (err == ERR_OK) {
        printf("Temperature: %.2f C\n", temperature);
    } else if (err == ERR_I2C_BUS) {
        printf("I2C bus error\n");
        // 可能忘记处理某种错误
    } else if (err == ERR_SENSOR_TIMEOUT) {
        printf("Sensor timeout\n");
    } else if (err == ERR_INVALID_DATA) {
        printf("Invalid data received\n");
    } else {
        printf("Unknown error: %d\n", err);
    }
    // 容易忘记检查错误
    error_t another_err = send_data(&temperature);
    // 这里缺少对another_err的检查
}
```

现在看看Rust的等效实现：

```rust
// Rust错误处理 - 使用枚举和Result类型
#[derive(Debug)]
enum SensorError {
    I2cBus(I2cError),      // 包含底层错误
    SensorTimeout,
    InvalidData,
    UartBusy(UartError),   // 包含底层错误
}
// 自动从I2C错误转换，避免手动映射错误
impl From<I2cError> for SensorError {
    fn from(err: I2cError) -> Self {
        SensorError::I2cBus(err)
    }
}
// 读取传感器
fn read_temperature_sensor() -> Result<f32, SensorError> {
    // 初始化I2C总线 - ?操作符自动传播错误
    let i2c = i2c_init()?;
    // 读取传感器 - ?操作符自动处理和转换错误
    let raw_data = i2c.read(SENSOR_ADDR, 4)?;
    // 检查数据有效性
    if raw_data[3] != calculate_checksum(&raw_data[0..3]) {
        return Err(SensorError::InvalidData);
    }
    // 转换数据
    Ok(convert_to_temperature(&raw_data))
}
// 使用示例
fn process_sensor_data() -> Result<(), SensorError> {
    // 读取传感器 - ?自动传播错误
    let temperature = read_temperature_sensor()?;
    // 可以立即使用结果，因为已经处理了错误
    println!("Temperature: {:.2} C", temperature);
    // 发送数据 - 错误会自动转换
    send_data(&temperature)?;
    // 一切正常时返回成功
    Ok(())
}
// 主函数中使用模式匹配处理错误
fn main() {
    match process_sensor_data() {
        Ok(()) => println!("处理完成"),
        // 全面的错误匹配，编译器确保所有情况都被处理
        Err(err) => match err {
            SensorError::I2cBus(bus_err) => {
                println!("I2C总线错误: {:?}", bus_err);
                // 还可以进一步处理具体的I2C错误
            },
            SensorError::SensorTimeout => println!("传感器超时"),
            SensorError::InvalidData => println!("接收到无效数据"),
            SensorError::UartBusy(uart_err) => {
                println!("UART忙: {:?}", uart_err);
            },
        },
    }
}
```
对于上面代码可以看到，Rust的实现有以下优势：

1. **`?`操作符简化错误传播**：相比C语言每次调用后都要检查返回值，Rust的`?`操作符让错误传播变得简洁。

2. **错误类型化与组合**：Rust的错误是类型化的，可以包含额外信息，而不仅仅是整数码。这使错误处理更精确、更有表现力。

3. **编译器强制错误处理**：如果不处理`Result`，编译器会发出警告，防止遗漏错误检查。

4. **模式匹配确保全面性**：使用`match`处理错误时，编译器确保所有可能的错误类型都得到处理。

5. **无性能损失**：Rust的错误处理模型在编译时解析，几乎没有运行时开销，与手动检查错误代码的C代码性能相当。

6. **错误转换自动化**：通过`From` trait实现，可以自动在不同错误类型间转换，减少手动映射代码。

在嵌入式系统中，可靠的错误处理尤为重要，因为错误往往意味着硬件交互问题，可能导致系统不稳定。Rust的错误处理机制既保持了底层控制的能力，又提供了更高层次的抽象，使代码更安全、更可维护，同时不引入额外的运行时开销。
