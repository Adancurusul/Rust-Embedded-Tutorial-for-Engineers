# Using Vec and Other Containers in Embedded Systems

Since Rust embedded is mostly in no_std environments, many functions from the standard library cannot be used, such as Vec. However, many people find writing Rust this way too troublesome. Here are two solutions:

1. Using heapless
2. Using a global allocator

## heapless

[heapless](https://github.com/rust-embedded/heapless) is an important library in the Rust embedded ecosystem, designed as a collection of static data structures for resource-constrained environments. It provides containers similar to the standard library (such as Vec, String, etc.), but completely independent of heap memory allocation, with all memory allocated at compile time on the stack or in static areas. When using it, you need to specify the maximum capacity through type parameters. Although capacity is fixed and cannot be dynamically expanded, this design ensures predictable memory usage, avoiding memory fragmentation and allocation failure problems common in embedded systems, making it particularly suitable for real-time systems and no-OS environments.

Like this:

```rust
use heapless::Vec;
let mut vec = Vec::<u8, 10>::new();
vec.push(1).unwrap();
vec.push(2).unwrap();
println!("{:?}", vec);
```

## Using a Global Allocator

For embedded systems, [embedded-alloc](https://github.com/rust-embedded/embedded-alloc) is recommended.
Usage is very simple:

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

Simply put, this creates a global allocator that can implement malloc, free, realloc and other operations, allowing us to use containers like Vec, Box that require dynamic memory allocation. You can also create your own implementation of the [GlobalAlloc](https://doc.rust-lang.org/std/alloc/trait.GlobalAlloc.html) trait as a memory pool.

Using embedded-alloc requires critical section support, so Cargo.toml needs to enable this in cortex-m:

```toml
cortex-m = {version = "0.7.7", features = ["critical-section-single-core"]}
```

Note: Using dynamic memory allocation in embedded systems requires attention to memory fragmentation and out-of-memory (OOM) issues, so it should be used carefully or with sufficient testing.