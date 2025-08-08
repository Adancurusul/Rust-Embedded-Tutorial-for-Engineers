# 嵌入式中的Vec等容器使用


由于Rust嵌入式中大多是no_std环境，于是标准库中很多函数不能使用，比如Vec等，但是很多人会觉得这样写Rust太麻烦了，这里介绍两种方案可以解决：

1. 使用heapless
2. 使用全局的allocator

## heapless

[heapless](https://github.com/rust-embedded/heapless)是Rust嵌入式生态中的重要库，专为资源受限环境设计的静态数据结构集合。它提供了与标准库类似的容器（如Vec、String等），但完全不依赖堆内存分配，所有内存都在编译时分配在栈上或静态区域。使用时需要通过类型参数指定最大容量，虽然容量固定无法动态扩展，但这种设计确保了内存使用的可预测性，避免了嵌入式系统中常见的内存碎片和分配失败问题，特别适合实时系统和无操作系统环境。
类似这样：


```rust
use heapless::Vec;
let mut vec = Vec::<u8, 10>::new();
vec.push(1).unwrap();
vec.push(2).unwrap();
println!("{:?}", vec);
```

## 使用全局的allocator

嵌入式中推荐使用[embedded-alloc](https://github.com/rust-embedded/embedded-alloc)
用法非常简单

```rust
#![no_std]
#![no_main]
extern crate alloc;
use cortex_m_rt::entry;
use embedded_alloc::LlffHeap as Heap;
#[global_allocator]
static HEAP: Heap = Heap::empty();
#[entry]
fn main() -> ! {
    // Initialize the allocator BEFORE you use it
    {
        use core::mem::MaybeUninit;
        const HEAP_SIZE: usize = 1024;
        static mut HEAP_MEM: [MaybeUninit<u8>; HEAP_SIZE] = [MaybeUninit::uninit(); HEAP_SIZE];
        unsafe { HEAP.init(&raw mut HEAP_MEM as usize, HEAP_SIZE) }
    }
    // now the allocator is ready types like Box, Vec can be used.
    loop { /* .. */ }
}
```

简而言之就是创建了一个全局的allocator，能实现malloc、free、realloc等操作，这样我们就能使用需要动态内存分配的Vec、Box等容器了。也可以自己做一个实现了 [GlobalAlloc](https://doc.rust-lang.org/std/alloc/trait.GlobalAlloc.html) trait的内存池

embedded-alloc的使用需要critical section的支持，所以Cargo.toml需要在cortex-m中启用

```toml
cortex-m = {version = "0.7.7", features = ["critical-section-single-core"]}
```

注意：嵌入式系统中使用动态内存分配需要注意内存碎片和内存不足（OOM）等问题，因此需要谨慎使用或进行充分测试。