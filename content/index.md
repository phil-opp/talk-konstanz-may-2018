class: center, middle
name: title
count: false

# The Rust Way of OS Development
???

Notes for the _first_ slide!

---

# About Me

- Computer science student at KIT .grey[(Karlsruhe)]
- Rust user since 2014


- _“Writing an OS in Rust”_ blog series
- Embedded Rust development


- `rust-osdev` organization on Github
    - Host and maintain tools/libraries for Rust OS development
    - Crates: `x86_64`, `acpi`, `multiboot2-elf64`, `bootloader`, `bootimage`
    - Join us!

---
<img src="content/images/rust-logo-blk.svg" alt="Rust logo" width="300rem" height="auto" style="position: absolute; right: 0rem; margin-top: -2rem;">

# Rust

- 3 year old programming language
- Memory safety without garbage collection
- Used by Mozilla, Dropbox, Cloudflare, …

```rust
enum Event {
    Load,
    KeyPress(char),
    Click { x: i64, y: i64 }
}

fn print_event(event: Event) {
    match event {
        Event::Load => println!("Loaded"),
        Event::KeyPress(c) => println!("Key {} pressed", c),
        Event::Click {x, y} => println!("Clicket at x={}, y={}", x, y),
    }
}
```

---

# OS Development

- “Bare metal” environment
    - No underlying operating system
    - No processes, threads, files, heap, …

**Goals**

- Abstractions
    - For hardware devices <span class="grey">(drivers, files, network sockets, …)</span>
    - For concurrency <span class="grey">(threads, synchronization primitives, IPC, …)</span>
- Isolation <span class="grey">(processes, containers, virtual machines, …)</span>
- Security <span class="grey">(access control, ASLR, …)</span>

---

# OS Develoment in Rust

- **Writing an OS in Rust**: Tutorials for basic functionality
    - Booting, testing, CPU exceptions, page tables
    - No C dependencies
    - Works on Linux, Windows, macOS

<img class="center" alt="Screenshot of Writing an OS in Rust website" src="content/images/writing-an-os-in-rust.png" style="margin-left: auto; margin-right: auto; width: auto; height: 23rem;">

---

# OS Develoment in Rust

- **Writing an OS in Rust**: Tutorials for basic functionality
- **Redox OS**: Most complete Rust OS, microkernel design

<img class="center" alt="Screenshot of Redox running a webbrowser and a file manager" src="content/images/redox-screenshot.png" style="margin-left: auto; margin-right: auto; width: auto; height: 25rem;">
---

# OS Develoment in Rust

- **Writing an OS in Rust**: Tutorials for basic functionality
- **Redox OS**: Most complete Rust OS, microkernel design
- **Nebulet**: Experimental WebAssembly kernel
    - WebAssembly is a binary format for executable code in web pages
    - Idea: Run wasm applications instead of native binaries
    - Wasm is sandboxed, so it can safely run in ring 0
    - A bit slower than native code
    - But no context switches

---

# OS Develoment in Rust

- **Writing an OS in Rust**: Tutorials for basic functionality
- **Redox OS**: Most complete Rust OS, microkernel design
- **Nebulet**: Experimental WebAssembly kernel
- **Tock**: Operating system for embedded systems

<img class="center" alt="Screenshot of Tock website" src="content/images/tock-screenshot.png" style="margin-left: auto; margin-right: auto; width: auto; height: 22rem;">


---

# OS Develoment in Rust

- **Writing an OS in Rust**: Tutorials for basic functionality
- **Redox OS**: Most complete Rust OS, microkernel design
- **Nebulet**: Experimental WebAssembly kernel
- **Tock**: Operating system for embedded systems

<div style="height:5rem"></div>

.center[**What does using Rust mean for OS development?**]

---
class: center, middle

.rust-means[Rust means…]

# Memory Safety

---

# Memory Safety

- No invalid memory accesses
    - No buffer overflows
    - No dangling pointers
    - No race conditions
- Guaranteed by Rust's ownership system
.grey[- Violations are only possible using `unsafe`]

--

In C:

- Array capacity is not checked on access
    - Easy to get buffer overflows
- Every `malloc` needs exactly one `free`
    - Easy to get _use-after-free_ or _double-free_ bugs


- Vulnerabilities caused by memory unsafety are still common


---
# Memory Safety

<img alt="Buffer Overflow Vulnerabilities in Linux" src="content/images/Buffer Overflow Vulnerabilities in Linux.svg" style="margin-top: 0rem; margin-bottom: -2rem;">

<img src="content/images/Tux.svg" alt="Linux logo" width="120rem" height="auto" style="display:block; position: absolute; top: 1rem; right: 3rem;">

.grey[.small[Source: https://www.cvedetails.com/product/47/Linux-Linux-Kernel.html?vendor_id=33]]


???

.grey[
- CVE = Common Vulnerabilities and Exposures
]

---
# Memory Safety

<img alt="Linux CVEs in 2018" src="content/images/Linux CVEs in 2018.svg" style="margin-top: 0rem; margin-bottom: -2rem;">

<img src="content/images/Tux.svg" alt="Linux logo" width="120rem" height="auto" style="display:block; position: absolute; top: 1rem; right: 3rem;">

.grey[.smaller[Source: https://www.cvedetails.com/vulnerability-list/vendor_id-33/product_id-47/year-2018/Linux-Linux-Kernel.html]]

---
# Memory Safety: A Strict Compiler

- It can take some time until your program compiles
    - “Fighting the borrow checker”
- Lifetimes can be complicated
    - “error: `x` does not live long enough”


However:
- “If it compiles, it usually works”
- Far less debugging
    - No data races!
- Refactoring is safe and painless

--

<div style="height:1rem"></div>

What about `unsafe`?

---

class: center, middle

.rust-means[Rust means…]

# Encapsulating Unsafety

---

# Encapsulating Unsafety

- Sometimes you need `unsafe` in a kernel
    - Writing to the VGA text buffer at `0xb8000`
    - Modifying CPU configuration registers
    - Switching the address space (reloading `CR3`)
- Goal: Provide safe abstractions that encapsulate unsafety


Example:

```rust
/// Invalidate the TLB completely by reloading the CR3 register.
pub fn flush_tlb() { // safe interface
    use registers::control::Cr3;
    let (frame, flags) = Cr3::read();
    unsafe { Cr3::write(frame, flags) }
}
```

⇒ Function can't be used in an `unsafe` way

---
# Encapsulating Unsafety

Not possible in all cases:

```rust
/// Write a new root table address into the CR3 register.
pub fn write_cr3(page_table_frame: PhysFrame, flags: Cr3Flags) {
    let addr = page_table_frame.start_address();
    let value = addr.as_u64() | flags.bits();
    unsafe { asm!("mov $0, %cr3" :: "r" (value) : "memory"); }
}
```

--

**Problem**: Passing an invalid `PhysFrame` could break memory safety!

- A frame that is no page table
- A page table that maps all pages to the same frame
- A page table that maps two random pages to the same frame

---
count: false

# Encapsulating Unsafety

Not possible in all cases:

```rust
/// Write a new root table address into the CR3 register.
pub `unsafe` fn write_cr3(page_table_frame: PhysFrame, flags: Cr3Flags) {
    let addr = page_table_frame.start_address();
    let value = addr.as_u64() | flags.bits();
    asm!("mov $0, %cr3" :: "r" (value) : "memory");
}
```

**Problem**: Passing an invalid `PhysFrame` could break memory safety!

- A frame that is no page table
- A page table that maps all pages to the same frame
- A page table that maps two random pages to the same frame

⇒ Function needs to be .mark[`unsafe`] because it depends on valid input

---
class: extra-spacing

# Encapsulating Unsafety

Edge Cases: Functions that…

- … disable paging?
--
<span style="margin-left:3rem;"></span> `unsafe`
- … disable CPU interrupts?
--
<span style="margin-left:3rem;"></span> `safe`
- … might cause CPU exceptions?
--
<span style="margin-left:3rem;"></span> `safe`
- … can be only called from privileged mode?
--
<span style="margin-left:3rem;"></span> `safe`
- … assume certain things about the hardware? <span style="margin-left:3rem;"></span> .hidden[`depends`]
    - E.g. there is a VGA text buffer at `0xb8000`
--
class: no-hide

---
class: center, middle

.rust-means[Rust means…]

# High Level Code

---

# High Level Code: Mutexes

<table>
    <thead><tr><th width="50%" style="text-align: center">C++</th><th width="50%">Rust</th></tr></thead>
    <tbody>
        <tr>
            <td>
<pre><code class="C++">std::vector<int> data = {1, 2, 3};
// mutex is unrelated to data
std::mutex mutex;

// unsynchronized access possible
data.push_back(4);

mutex.lock();
data.push_back(5);
mutex.unlock();
</pre></code>
            </td><td>
<pre><code class="Rust">let data = vec![1, 2, 3];
// mutex owns data
let mutex = Mutex::new(data);

// compilation error: data was moved
data.push(5);

let mut d = mutex.lock().unwrap();
d.push(4);
// released at end of scope
</pre></code>
            </td>
        </tr>
    </tbody>
</table>

⇒ Rust ensures that Mutex is locked before accessing data

---

# High Level Code: Page Table Abstractions

From the `x86_64` crate:

```Rust
pub trait Mapper<S: PageSize> {
    fn map_to<A>(
        &mut self,
        page: Page<S>,              // map this page
        frame: PhysFrame<S>,        // to this frame
        flags: PageTableFlags,
        frame_allocator: &mut A,    // we might need frames for new page tables
    ) -> Result<MapperFlush<S>, MapToError>
    where
        A: FnMut() -> Option<PhysFrame<Size4KB>>;
}

impl PageSize for Size4KB {…} // standard page
impl PageSize for Size2MB {…} // “huge” 2MB page
impl PageSize for Size1GB {…} // “giant” 1GB page (only on some architectures)
```

---

# High Level Code: Page Table Abstractions

From the `x86_64` crate:

```Rust
#[must_use = "Page Table changes must be flushed or ignored."]
pub struct MapperFlush<S: PageSize>(Page<S>);

impl<S: PageSize> MapperFlush<S> {
    // Flush the TLB entry for this page
    pub fn flush(self) {
        tlb::flush(self.0.start_address());
    }

    pub fn ignore(self) {}
}
```

If not used: .grey[_“warning: unused `MapperFlush` which must be used: Page Table changes must be flushed or ignored.”_]

---

# High Level Code

Allows to:

- Make erroneous code impossible
.grey[- Data behind a Mutex can't be accessed without locking]
- Represent contracts in code instead of documentation
.grey[
- Page size of page and frame parameters must match in `map_to`
- The TLB entry must be flushed after mapping
- Additional frames are needed (from any allocator)
]

<div style="height:1rem"></div>

Everything happens at compile time ⇒ _No run-time cost!_

---
class: center, middle

.rust-means[Rust means…]

# Easy Dependency Management

---
# Easy Dependency Management

.float-right[![Cargo logo](content/images/cargo-logo.png)]

- Over 15000 crates on **crates.io**
- Simply specify the desired version
    - Add single line to `Cargo.toml`
- Cargo takes care of the rest

--

It works the same for OS kernels:

- Over 350 crates in the `no_std` category
    - Many more can be trivially made `no_std`
- Dependencies of the Redox kernel:
 - `bitflags`: .grey[A macro for generating structures with single-bit flags.]
 - `linked_list_allocator`: .grey[A simple memory allocator.]
 - `spin`: .grey[Spinning synchronization primitives such as spinlocks.]
 - `x86`: .grey[Structures, registers, and instructions specific to `x86` CPUs.]
 - etc.

---

class: center, middle

.rust-means[Rust means…]

# Great Tooling

---

# Great Tooling

- **rustup**: Use multiple Rust versions for different directories
- **cargo**: Automatically download, build, and link dependencies
- **rustfmt**: Format Rust code according to style guidelines
--

- **Rust Playground**: Run and share code snippets in your browser

![Rust Playground screenshot](content/images/playground.png)

---

# Great Tooling

- **clippy**: Additional warnings for dangerous or unidiomatic code

```rust
fn equal(x: f32, y: f32) -> bool {
    if x == y { true } else { false }
}
```
--

```
error: `strict comparison of f32 or f64`
 --> src/main.rs:2:8
  |
2 | if x == y { true } else { false }
  |    ^^^^^^ help: `consider comparing them within some error: (x - y).abs() < err`

warning: `this if-then-else expression returns a bool literal`
 --> src/main.rs:2:5
  |
2 | if x == y { true } else { false }
  | ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^ help: `you can reduce it to: x == y`
```

---

# Great Tooling

- **bors**: Test merges of pull requests before pushing them
    - Ensure that master branch is always green

<table>
    <thead><tr><th width="50%" style="text-align: center">Without bors</th><th width="50%">With bors</th></tr></thead>
    <tbody>
        <tr>
            <td style="padding:1rem">
                <img src="content/images/without-bors.svg">
            </td><td style="padding:1rem">
                <img src="content/images/with-bors.svg">
            </td>
        </tr>
    </tbody>
</table>

<div style="height:2rem"></div>

.grey[.small[See https://bors.tech/essay/2017/02/02/pitch/]]

---

# Great Tooling

- **proptest**: A property testing framework

```rust
fn parse_date(s: &str) -> Option<(u32, u32, u32)> {
    // […] check if valid YYYY-MM-DD format
    let year = &s[0..4];
    let month = &s[`6`..7]; // BUG: should be 5..7
    let day = &s[8..10];
    convert_to_u32(year, month, date)
}
proptest! {
    #[test]
    fn parse_date(`y in 0u32..10000`, `m in 1u32..13`, `d in 1u32..32`) {
        let (y2, m2, d2) = parse_date(
            &format!("{:04}-{:02}-{:02}", y, m, d)).unwrap();
        prop_assert_eq!((y, m, d), (y2, m2, d2));
    }
}
```

---

<pre style="margin-left: 2rem;"><code style="line-height: 0.9; font-size: 20px;">
- try random values                     y = 2497, m = 8, d = 27     passes
    |                                   y = 9641, m = 8, d = 18     passes
    | (failed test case found)          `y = 7360, m = 12, d = 20`    fails

- reduce y to find simpler case         y = 3680, m = 12, d = 20    fails
    |                                   y = 1840, m = 12, d = 20    fails
    |                                   y = 920, m = 12, d = 20     fails
    |                                   y = 460, m = 12, d = 20     fails
    |                                   y = 230, m = 12, d = 20     fails
    |                                   y = 115, m = 12, d = 20     fails
    |                                   y = 57, m = 12, d = 20      fails
    |                                   y = 28, m = 12, d = 20      fails
    |                                   y = 14, m = 12, d = 20      fails
    |                                   y = 7, m = 12, d = 20       fails
    |                                   y = 3, m = 12, d = 20       fails
    |                                   y = 1, m = 12, d = 20       fails
    | (simplest y case still fails)     `y = 0, m = 12, d = 20`       fails

- reduce m to find simpler case         y = 0, m = 6, d = 20        passes
    |                                   y = 0, m = 9, d = 20        passes
    |                                   y = 0, m = 11, d = 20       fails
    | (minimum failure value found)     `y = 0, m = 10, d = 20`       fails

- reduce d to find simpler case         y = 0, m = 10, d = 10       fails
    |                                   y = 0, m = 10, d = 5        fails
    |                                   y = 0, m = 10, d = 3        fails
    |                                   y = 0, m = 10, d = 2        fails
    | (reduced test case found)         `y = 0, m = 10, d = 1`        fails
</code></pre>

.grey[.small[See <https://github.com/altsysrq/proptest>]]

---

# Great Tooling for OS Development

In C:

- First step is to build a **cross compiler**
    - A `gcc` that compiles for your target system
    - Lots of build dependencies
- On Windows, you have to use **cygwin**
    - Required for using the GNU build tools <span class="grey">(e.g. `make`)</span>
    - The _Windows Subsystem for Linux_ might also work

In Rust:

- Rust works natively on Linux, Windows, and macOS
- The Rust compiler `rustc` is already a cross-compiler
- For linking, we can use the cross-platform **`lld`** linker
    - By the LLVM project

---

# Great Tooling for OS Development

**bootimage**: Create a bootable disk image from a Rust kernel

- Cross-platform, no C dependencies
- Automatically downloads and compiles a bootloader
- **bootloader**: A x86 bootloader written in Rust and inline assembly
- **cargo-xbuild**: Automatically cross compile the `core` library
.grey[    - Fork of `xargo`, which is in maintainance mode]

**Goals**:

- Make building your kernel as easy as possible
- Let beginners dive immediately into OS programming
    - No hours-long toolchain setup
- Remove platform-specific differences
    - You shouldn't need Linux to do OS development

<div style="height:1rem"></div>

.grey[.small[See https://os.phil-opp.com/news/2018-03-09-pure-rust/]]

---

# Great Tooling for OS Development

In development: **bootimage test**

- Basic integration test framework
- Runs each test executable in an isolated QEMU instance
    - Tests are completely independent
    - Results are reported through the serial port
- Allows testing in target environment

Testing on real hardware?
---

class: center, middle

.rust-means[Rust means…]

# An Awesome Community
---
# An Awesome Community

- torvalds rant
- only positive comments
- Awesome developers: Redox

---
class: center, middle

.rust-means[Rust means…]

# No Elitism

---

# No Elitism

> “A **decade of programming**, including a few years of low-level coding in assembly language and/or a systems language such as C, is **pretty much the minimum necessary** to even understand the topic well enough to work in it.”

.right[.grey[From [**wiki.osdev.org**/Beginner_Mistakes](https://wiki.osdev.org/Beginner_Mistakes)]]

**_vs._**

> The book assumes that you have programmed in some language before, but not any particular one. In fact, **people who have not done low-level programming before are a specific target of this book**.

.right[.grey[From [**intermezzos.org**/book](http://intermezzos.github.io/book/second-edition/)]]

---
blog_os

---

class: center, middle

.rust-means[Rust means…]

# Exciting New Features
---
# Exciting New Features

- Futures
- Async / await
- webassembly?

---

# Futures

Result of an asynchronous computation:

``` rust
trait Future {
    type Item;
    type Error;
    fn poll(&mut self, cx: &mut Context) -> Result<Async<Self::Item>,
                                                   Self::Error>;
}

enum Async<T> {
    Ready(T),
    Pending,
}
```

- Instead of blocking, `Async::Pending` is returned
- Zero cost abstraction

---

# Futures: Implementation Details

- Futures do nothing until polled
- An `Executor` is used for polling multiple futures until completion
    - Like a scheduler
- If future is not ready when polled, a `Waker` is created
    - Notifies the `Executor` when the future becomes ready
    - Avoids continuous polling


**Combinators**

- Transform a future without polling it .grey[(similar to iterators)]
- Examples
    - `future.map(|v| v + 1)`<span class="grey">: Applies a function to the result</span>
    - `future_a.join(future_b)`<span class="grey">: Wait for both futures</span>
    - `future.and_then(|v| some_future(v))`<span class="grey">: Chain dependent futures</span>
---

# Async / Await

Traditional synchronous code:

```rust
fn get_user_from_database(user_id: u64) -> Result<User> {…}

fn handle_request(request: Request) -> Result<Response> {
    let user = get_user_from_database(request.user_id)?;
    generate_response(user)
}
```

- Thread blocked until database read finished
    - Complete thread stack unusable
- Number of threads limits number of concurrent requests

---

# Async / Await
Asynchronous variant:

```rust
​`async` fn get_user_from_database(user_id: u64) -> Result<User> {…}

​`async` fn handle_request(request: Request) -> Result<Response> {
    let user = `await!`(get_user_from_database(request.user_id))?;
    generate_response(user)
}
```

- Async functions return `Future<Item=T>` instead of `T`
- No blocking occurs
    - It only _looks_ blocking
    - Stack can be reused for handling other requests
- Thousands of concurrent requests possible

How does `await` work?

---

# Async / Await: Generators

- Functions that can suspend themselves via `yield`:

```rust
fn main() {
    let mut generator = || {
        println!("2");
        `yield`;
        println!("4");
    };

    println!("1");
    unsafe { generator.resume() };
    println!("3");
    unsafe { generator.resume() };
    println!("5");
}
```

--

- Compiled as state machines

---

```rust
let mut generator = {
    `enum Generator { Start, Yield1, Done, }`

    impl Generator {
        unsafe fn resume(&mut self) {
            match self {
                `Generator::Start` => {
                    println!("2");
                    *self = Generator::Yield1;
                }
                `Generator::Yield1` => {
                    println!("4");
                    *self = Generator::Done;
                }
                `Generator::Done` => panic!("generator resumed after completion")
            }
        }
    }
    Generator::Start
};
```

---

# Async / Await: Generators

- Generators can keep state:

```rust
fn main() {
    let mut generator = || {
        `let number = 42;`
        `let ret = "foo";`

        yield number; // yield can return values
        return ret
    };

    unsafe { generator.resume() };
    unsafe { generator.resume() };
}
```

Where are `number` and `ret` stored between `resume` calls?

---

```rust
let mut generator = {
    enum Generator {
        Start(`i32, &'static str`),
        Yield1(`&'static str`),
        Done,
    }
    impl Generator {
        unsafe fn resume(&mut self) -> GeneratorState<i32, &'static str> {
            match self {
                Generator::Start(`i, s`) => {
                    *self = Generator::Yield1(s); GeneratorState::`Yielded(i)`
                }
                Generator::Yield1(`s`) => {
                    *self = Generator::Done; GeneratorState::`Complete(s)`
                }
                Generator::Done => panic!("generator resumed after completion")
            }
        }
    }
    Generator::Start(ret)
};
```

---

# Async / Await: Generators

- Why is `resume` unsafe?

<table style="border: none; margin-top: -2rem; margin-bottom: -2rem;">
    <tbody>
        <tr style="vertical-align: top;">
            <td>
<pre><code class="Rust">fn main() {
    let mut generator = move || {
        let foo = 42;
        `let bar = &foo;`
        yield;
        return bar
    };
    unsafe { generator.resume() };
    `let heap_generator = Box::new(generator);`
    unsafe { heap_generator.resume() };
}</pre></code>
            </td><td style="text-align: left;">
<pre><code class="Rust">enum Generator {
    Start,
    Yield1(i32, &i32),
    Done,
}
</pre></code>
            </td>
        </tr>
    </tbody>
</table>

--

- Generator contains reference to itself
    - No longer valid when moved to the heap ⇒ undefined behavior
    - Must not be moved after first `resume`
---

# Async / Await: Implementation

```rust
async fn handle_request(request: Request) -> Result<Response> {
    let future = get_user_from_database(request.user_id);
    let user = `await!`(future)?;
    generate_response(user)
}
```

Compiles roughly to:

```rust
async fn handle_request(request: Request) -> Result<Response> { `GenFuture`(|| {
    let future = get_user_from_database(request.user_id);
    let user = loop { match `future.poll()` {
        Ok(Async::Ready(u)) => break Ok(u),
        Ok(Async::NotReady) => `yield`,
        Err(e) => break Err(e),
    }}?;
    generate_response(user)
})}
```

---

# Async / Await: Implementation

Transform Generator into Future:

```rust
struct GenFuture<T>(T);

impl<T: Generator> Future for GenFuture<T> {
    fn poll(&mut self) -> Poll<T::Item, T::Error> {
        match unsafe { self.0.resume() } {
            GeneratorStatus::Complete(Ok(result)) => Ok(Async::Ready(result)),
            GeneratorStatus::Complete(Err(e)) => Err(e),
            GeneratorStatus::Yielded => Ok(Async::NotReady),
        }
    }
}
```


---

# Async / Await: For OS Dev?




---

# Summary

Rust means:

- **Memory Safety**
- **Encapsulating Unsafety**
- **High Level Code**.grey[ &nbsp;&nbsp;&nbsp;Mutexes, Page Table Abstractions]


- **Easy Dependency Management**.grey[ &nbsp;&nbsp;&nbsp;cargo, crates.io]
- **Great Tooling**.grey[ &nbsp;&nbsp;&nbsp;clippy, bors, proptest, bootimage]


- **An Awesome Community**
- **No Elitism**


- **Exciting New Features**.grey[ &nbsp;&nbsp;&nbsp;Futures, Async / Await]

<div style="height:.5rem"></div>

.grey[Slides are available at <https://os.phil-opp.com/talks>]

--

---
class: center, middle

# Extra Slides

---

# Await: Just Syntactic Sugar?

Is `await` just syntactic sugar for the `and_then` combinator?

```rust
async fn handle_request(request: Request) -> Result<Response> {
    let user = `await!`(get_user_from_database(request.user_id))?;
    generate_response(user)
}
```

```rust
async fn handle_request(request: Request) -> Result<Response> {
    get_user_from_database(request.user_id).`and_then`(|user| {
        generate_response(user)
    })
}
```

In this case, both variants work.

---

# Await: Not Just Syntactic Sugar!

```rust
fn read_info_buf(socket: &mut Socket) -> [u8; 1024]
    -> impl Future<Item = [0; 1024], Error = io::Error> + `'static`
{
    let mut buf = [0; 1024];
    let mut cursor = 0;

    `while` cursor < 1024 {
        cursor += await!(socket.read(`&mut buf`[cursor..]))?;
    };
    buf
}
```

- We don't know how many `and_then` we need
    - But each one is their own type -> boxed trait objects required
- `buf` is a local stack variable, but the returned future is `'static`
    - Not possible with `and_then`
    - _Pinned types_ allow it for `await`
