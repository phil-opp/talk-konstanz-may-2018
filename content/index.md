class: center, middle
name: title
count: false

# The Rust Way of OS Development

.grey[Philipp Oppermann]

.small[.grey[May 30, 2018 <span style="margin-left: 2rem"></span> HTWG Konstanz]]

???

Notes for the _first_ slide!

---

# About Me

- Computer science student at KIT .grey[(Karlsruhe)]
- _“Writing an OS in Rust”_ blog series <span class="grey">(<a href="https://os.phil-opp.com">os.phil-opp.com</a>)</span>
- Embedded Rust development

<div style="height: 2rem"></div>

Contact:

- phil-opp on GitHub
- Email: hello@phil-opp.com
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
        Event::Click {x, y} => println!("Clicked at x={}, y={}", x, y),
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
    - For hardware devices <span class="grey">(drivers, files, …)</span>
    - For concurrency <span class="grey">(threads, synchronization primitives, …)</span>
- Isolation <span class="grey">(processes, address spaces, …)</span>
- Security

---

# OS Development in Rust

- **Writing an OS in Rust**: Tutorials for basic functionality
    - Booting, testing, CPU exceptions, page tables
    - No C dependencies
    - Works on Linux, Windows, macOS

<img class="center" alt="Screenshot of Writing an OS in Rust website" src="content/images/writing-an-os-in-rust.png" style="margin-left: auto; margin-right: auto; width: auto; height: 23rem;">

---

# OS Development in Rust

- **Writing an OS in Rust**: Tutorials for basic functionality
- **Redox OS**: Most complete Rust OS, microkernel design

<img class="center" alt="Screenshot of Redox running a webbrowser and a file manager" src="content/images/redox-screenshot.png" style="margin-left: auto; margin-right: auto; width: auto; height: 25rem;">
---

# OS Development in Rust

- **Writing an OS in Rust**: Tutorials for basic functionality
- **Redox OS**: Most complete Rust OS, microkernel design
- **Tock**: Operating system for embedded systems

<img class="center" alt="Screenshot of Tock website" src="content/images/tock-screenshot.png" style="margin-left: auto; margin-right: auto; width: auto; height: 22rem;">

---

# OS Development in Rust

- **Writing an OS in Rust**: Tutorials for basic functionality
- **Redox OS**: Most complete Rust OS, microkernel design
- **Tock**: Operating system for embedded systems
- **Nebulet**: Experimental WebAssembly kernel
    - WebAssembly is a binary format for executable code in web pages
    - Idea: Run wasm applications instead of native binaries
    - Wasm is sandboxed, so it can safely run in kernel address space
    - A bit slower than native code
    - But no expensive context switches or syscalls

---

# OS Development in Rust

- **Writing an OS in Rust**: Tutorials for basic functionality
- **Redox OS**: Most complete Rust OS, microkernel design
- **Tock**: Operating system for embedded systems
- **Nebulet**: Experimental WebAssembly kernel

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
    - No data races
- Guaranteed by Rust's ownership system
    - At compile time

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


???

.grey[
- CVE = Common Vulnerabilities and Exposures
]

---
# Memory Safety: A Strict Compiler

- It can take some time until your program compiles
- Lifetimes can be complicated
    - “error: `x` does not live long enough”


However:
- “If it compiles, it usually works”
- Far less debugging
    - No data races!
- Refactoring is safe and painless

---

class: center, middle

.rust-means[Rust means…]

# Encapsulating Unsafety

---

# Encapsulating Unsafety

- Not everything can be verified at compile time
- Sometimes you need `unsafe` in a kernel
    - Writing to the VGA text buffer at `0xb8000`
    - Modifying CPU configuration registers
    - Switching the address space (reloading `CR3`)
--


- Rust has `unsafe` blocks that allow to
    - Dereference raw pointers
    - Call `unsafe` functions
    - Access mutable statics
    - Implement `unsafe` traits


- Goal: Provide safe abstractions that encapsulate unsafety
    - Like hardware abstractions in an OS
---

# Encapsulating Unsafety: Example

```rust
/// Read current page table
pub fn read_cr3() -> PhysFrame { … }
/// Load a new page table
pub `unsafe` fn write_cr3(frame: PhysFrame) { … }

/// Invalidate the TLB completely by reloading the CR3 register.
pub fn flush_tlb() { // safe interface
    let frame = read_cr3();
    `unsafe` { write_cr3(frame) }
}
```

- The `CR3` register holds the root page table address
    - Reading is safe
    - Writing is `unsafe` <span class="grey"> (because it changes the address mapping)</span>
- The `flush_tlb` function provides a safe interface
    - It can't be used in an `unsafe` way


---
class: center, middle

.rust-means[Rust means…]

# A Powerful Type System

---

# A Powerful Type System: Mutexes

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
data.push(4);

let mut d = mutex.lock().unwrap();
d.push(5);
// released at end of scope
</pre></code>
            </td>
        </tr>
    </tbody>
</table>

⇒ Rust ensures that Mutex is locked before accessing data

---

# A Powerful Type System: Page Table Methods

Add a page table mapping:

```Rust
fn map_to<S: PageSize>(
    &mut PageTable,
    page: Page<S>,              // map this page
    frame: PhysFrame<S>,        // to this frame
    flags: PageTableFlags,
) {…}

impl PageSize for Size4KB {…} // standard page
impl PageSize for Size2MB {…} // “huge” 2MB page
impl PageSize for Size1GB {…} // “giant” 1GB page (only on some architectures)
```

---
count: false

# A Powerful Type System: Page Table Methods

Add a page table mapping:

```Rust
fn map_to<`S: PageSize`>(
    &mut PageTable,
    page: Page<`S`>,              // map this page
    frame: PhysFrame<`S`>,        // to this frame
    flags: PageTableFlags,
) {…}

impl PageSize for `Size4KB` {…} // standard page
impl PageSize for `Size2MB` {…} // “huge” 2MB page
impl PageSize for `Size1GB` {…} // “giant” 1GB page (only on some architectures)
```

- Generic over the page size
.grey[    - 4KB, 2MB or 1GB]
- Page and frame must have the same size

---

# A Powerful Type System

Allows to:

- Make misuse impossible
.grey[- Impossible to access data behind a Mutex without locking]
- Represent contracts in code instead of documentation
.grey[
- Page size of page and frame parameters must match in `map_to`
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
.grey[    - Add single line to `Cargo.toml`]
- Cargo takes care of the rest
.grey[    - Downloading, building, linking]

--

<div style="height: 3rem;"></div>

- It works the same for OS kernels
    - Crates need to be `no_std`
    - Useful crates: `bitflags`, `spin`, `arrayvec`, `x86_64`, …

---
# Easy Dependency Management

- **bitflags**: A macro for generating structures with single-bit flags

```rust
#[macro_use] extern crate bitflags;

bitflags! {
    pub struct PageTableFlags: u64 {
        const PRESENT =         1 << 0;    // bit 0
        const WRITABLE =        1 << 1;    // bit 1
        const HUGE_PAGE =       1 << 7;    // bit 7
        …
    }
}

fn main() {
    let stack_flags = PageTableFlags::PRESENT | PageTableFlags::WRITABLE;
    assert_eq!(stack_flags.bits(), 0b11);
}
```

---
# Easy Dependency Management

- **bitflags**: A macro for generating structures with single-bit flags
- **spin**: Spinning synchronization primitives such as spinlocks

--
- **arrayvec**: Stack-based vectors backed by a fixed sized array

```rust
use arrayvec::ArrayVec;

let mut vec = ArrayVec::<[i32; 16]>::new();
vec.push(1);
vec.push(2);
assert_eq!(vec.len(), 2);
assert_eq!(vec.as_slice(), &[1, 2]);
```

---
# Easy Dependency Management

- **bitflags**: A macro for generating structures with single-bit flags
- **spin**: Spinning synchronization primitives such as spinlocks
- **arrayvec**: Stack-based vectors backed by a fixed sized array
- **x86_64**: Structures, registers, and instructions specific to `x86_64`
    - Control registers
    - I/O ports
    - Page Tables
    - Interrupt Descriptor Tables
    - …

---
# Easy Dependency Management

- **bitflags**: A macro for generating structures with single-bit flags
- **spin**: Spinning synchronization primitives such as spinlocks
- **arrayvec**: Stack-based vectors backed by a fixed sized array
- **x86_64**: Structures, registers, and instructions specific to `x86_64`


<div style="height: 3rem;"></div>

- Over 350 crates in the no_std category
    - Many more can be trivially made `no_std`

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
        let date_str = format!("{:04}-{:02}-{:02}", y, m, d);
        let (y2, m2, d2) = parse_date(&date_str).unwrap();
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
    - A `gcc` that compiles for a bare-metal target system
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

- Code of Conduct from the beginning
    - “We are committed to providing a **friendly, safe and welcoming environment** for all […]”
    - “We will exclude you from interaction if you insult, demean or harass anyone”
    - Followed on GitHub, IRC, the Rust subreddit, etc.
--
- It works!
    - No inappropriate comments for “Writing an OS in Rust” so far
    - Focused technical discussions

--

<div style="height: 1rem"></div>

**_vs_**:

> “So this patch is utter and absolute garbage, and should be shot in the
head and buried very very deep.”

<div class="right small grey" style="margin-top: -3rem;"><a href="https://lkml.org/lkml/2017/8/14/698">Linus Torvalds on 14 Aug 2017</a></div>

---
class: center, middle

.rust-means[Rust means…]

# No Elitism

---

# No Elitism

Typical elitism in OS development:

> “A **decade of programming**, including a few years of low-level coding in assembly language and/or a systems language such as C, is **pretty much the minimum necessary** to even understand the topic well enough to work in it.”

.right[.grey[From [wiki.osdev.org/Beginner_Mistakes](https://wiki.osdev.org/Beginner_Mistakes)]]

Most Rust Projects:

- It doesn't matter where you come from
    - C, C++, Java, Python, JavaScript, …
- It's fine to ask questions
    - People are happy to help

---

# No Elitism

- **IntermezzOS**: “People who have not done low-level programming before are a specific target of this book“

<img src="content/images/intermezzos.png" style="margin-left: auto; margin-right: auto; width: 20rem; display: block;">


- **Writing an OS in Rust**: Deliberately no particular target audience
    - People are able to decide themselves
    - Provide links for things not explained on the blog
        - E.g. for advanced Rust and OS concepts

---
class: center, middle

.rust-means[Rust means…]

# Exciting New Features
---
# Exciting New Features

- **Impl Trait**: Return closures from functions
- **Non-Lexical Lifetimes**: A more intelligent borrow checker
- **WebAssembly**: Run Rust in browsers

<div style="height: 1rem"></div>

--

In development: **Futures and async / await**

- Simple and fast asynchronous code


- How does it work?
- What does it mean for OS development?
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
    NotReady,
}
```

- Instead of blocking, `Async::NotReady` is returned

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
    let future = get_user_from_database(request.user_id);
    let user = `await!`(future)?;
    generate_response(user)
}
```

- Async functions return `Future<Item=T>` instead of `T`
- No blocking occurs
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
    Generator::Start(42, "foo")
};
```

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
        Err(e) => break Err(e),
        Ok(Async::NotReady) => `yield`,
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
    fn poll(&mut self, cx: &mut Context) -> Result<Async<T::Item>, T::Error> {
        match unsafe { self.0.resume() } {
            GeneratorStatus::Complete(Ok(result)) => Ok(Async::Ready(result)),
            GeneratorStatus::Complete(Err(e)) => Err(e),
            GeneratorStatus::Yielded => Ok(Async::NotReady),
        }
    }
}
```

---

# Async / Await: For OS Development?

- Everything happens at compile time
    - Can be used in OS kernels and on embedded devices
- Makes asynchronous code simpler

<div style="height: 2rem;"></div>

Use Case: **Cooperative multitasking**

- Yield when waiting for I/O
- Executor then polls another future
- Interrupt handler notifies `Waker`
- Only a single thread is needed
    - Devices with limited memory

---

# Async / Await: An OS Without Blocking?

A blocking thread makes its whole stack unusable

- Makes threads heavy-weight
- Limits the number of threads in the system

What if blocking was not allowed?

--

- Threads would return futures instead of blocking
- Scheduler would schedule futures instead of threads
- Stacks could be reused for different threads


- Only a few stacks are needed for many, many futures
- Task-based instead of thread-based concurrency
    - Fine grained concurrency at the OS level

---

# Summary

Rust means:

- **Memory Safety**.grey[ &nbsp;&nbsp;&nbsp;no overflows, no invalid pointers, no data races]
- **Encapsulating Unsafety**.grey[ &nbsp;&nbsp;&nbsp;creating safe interfaces]
- **A Powerful Type System**.grey[ &nbsp;&nbsp;&nbsp;make misuse impossible]


- **Easy Dependency Management**.grey[ &nbsp;&nbsp;&nbsp;cargo, crates.io]
- **Great Tooling**.grey[ &nbsp;&nbsp;&nbsp;clippy, proptest, bootimage]


- **An Awesome Community** .grey[ &nbsp;&nbsp;&nbsp;code of conduct]
- **No Elitism** .grey[ &nbsp;&nbsp;&nbsp;asking questions is fine, no minimum requirements]


- **Exciting New Features**.grey[ &nbsp;&nbsp;&nbsp;futures, async / await]

<div style="height:.5rem"></div>

.grey[Slides are available at <https://os.phil-opp.com/talks>]

--

---
class: center, middle

# Extra Slides

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
