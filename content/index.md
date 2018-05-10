class: center, middle
name: title
count: false

# The Rust Way of OS Development
???

Notes for the _first_ slide!

---
<img src="content/images/rust-logo-blk.svg" alt="Rust logo" width="300rem" height="auto" style="float: right; margin-top: -2rem; margin-right: -2rem;">

# Rust

---

# OS Development


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

## Buffer Overflow Vulnerabitilies in Linux

<img src="content/images/Tux.svg" alt="Rust logo" width="100rem" height="auto" style="float: right; margin-top: -5rem;">

<div class="chart-container" style="position: relative; width:100%">
    <canvas id="linux-vulns"></canvas>
</div>

.grey[.small[Source: https://www.cvedetails.com/product/47/Linux-Linux-Kernel.html?vendor_id=33]]


???

.grey[
- CVE = Common Vulnerabilities and Exposures
]

---
# Memory Safety

## Linux CVEs in 2018 .grey[(Jan – Apr)]

<img src="content/images/Tux.svg" alt="Rust logo" width="100rem" height="auto" style="float: right; margin-top: -5rem;">

<div class="chart-container" style="position: relative; width:100%">
    <canvas id="linux-vulns-2018"></canvas>
</div>

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

class: center, middle

.rust-means[Rust means…]

# An Awesome Community
---
# An Awesome Community

freundliche und nicht-elitäre community

---

class: center, middle

.rust-means[Rust means…]

# Exciting New Features
---
# Exciting New Features

Futures, async/await
webassembly
