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

<div style="height:1rem"></div>

.grey[Bonus: Refactoring is safe and painless]

---

thread safety


---
sicherere Abstraktionen für unsafe operations


---
crates.io dependencies

---
freundliche und nicht-elitäre community

---
gute cross-platform tools,
