# Rust vs. Python: The Systems Programmer's Deep Dive

## 1. The Core Philosophy: "Runtime Flexibility" vs. "Compile-Time Guarantee"

| Feature | **Python (The Dynamic Runtime)** | **Rust (The Static Machine)** |
| :--- | :--- | :--- |
| **Execution** | **Interpreted / Bytecode.** The interpreter carries a massive runtime environment. "Everything is an object." | **Ahead-of-Time Compiled.** Zero runtime (conceptually). You pay the cost during compilation, not execution. |
| **Typing** | **Dynamic & Strong.** Types are checked at runtime. A variable `x` can hold an Integer, then a String. | **Static & Strong.** Types are resolved at compile time. `let x` is bound to a specific memory layout forever. |
| **Memory** | **Reference Counting + GC.** The interpreter pauses to collect cycles. Developer has zero control over memory layout. | **Ownership + Lifetimes.** Deterministic destruction at the end of scope (`}`). Developer controls stack vs. heap explicitly. |
| **Mutability** | **Implicitly Mutable.** Most objects (lists, dicts, class instances) can be changed unless explicitly immutable (tuples). | **Immutable by Default.** You must explicitly opt-in to mutation with `mut`. |

---

## 2. Data Structures: The Tuple Distinction (And Others)

This is where the structural differences are most acute.

| Concept | Python Implementation | Rust Implementation | **Critical Difference** |
| :--- | :--- | :--- | :--- |
| **Tuple** | `t = (1, "a")` | `let t = (1, "a");` | **Python:** An immutable array. You can index dynamically (`t[i]`) and iterate (`for x in t`). <br>**Rust:** An **anonymous struct**. You access fields via compile-time constants (`t.0`, `t.1`). You **cannot** index with a variable (`t[i]` is illegal) and you **cannot** iterate over it (heterogeneous types). |
| **List/Vec** | `[1, "a", True]` | `Vec<i32>` | **Python:** Heterogeneous. It is an array of pointers to `PyObject`. Heavy memory overhead per item. <br>**Rust:** Homogeneous. Contiguous block of memory for a single type. To mix types, you must use an Enum wrapper (`Vec<MyEnum>`). |
| **Map/Dict** | `{'k': 'v'}` | `HashMap<String, String>` | **Python:** Keys must be "Hashable" (runtime check). Insertion order is preserved (modern Python). <br>**Rust:** Keys must implement `Eq + Hash` traits (compile-time check). Insertion order is **not** guaranteed. |
| **String** | `s = "Hello"` | `String` vs `&str` | **Python:** Strings are immutable sequences of Unicode code points. Hides encoding complexity. <br>**Rust:** Strings are explicit UTF-8 buffers. `String` is a growable heap buffer (like `Vec<u8>`); `&str` is a view into that buffer. |

---

## 3. The Type System: Duck Typing vs. Trait Bounds

### Python: "If it has the method, it works."
Python relies on runtime introspection. You don't declare what an object *is*; you just hope it has the method you call.

```python
def process(item):
    # Works for ANY object with a .save() method
    item.save()

### Rust: "Prove it can do the work."
Rust uses **Traits** to define shared behavior. You must restrict generics using "Trait Bounds."
```

```rust
// Compile-time error if T does not implement the Save trait
fn process<T: Save>(item: T) {
    item.save();
}
```

## 4. Control Flow: Iterators and Pattern Matching
### Python: The `for` loop
Python loops are sugar for the `__iter__` and `__next__` dunder methods.

```python
# Standard Iteration
for x in items:
    if x > 5:
        print(x)
```

### Rust: Combinators and Pattern Matching
Rust treats iteration as a functional pipeline. Furthermore, `match` (pattern matching) is strictly more powerful than Python's `if/elif` or `match/case` because it checks **exhaustiveness** (you must handle every possible state).

```rust
// Functional Iteration
items.iter()
     .filter(|x| **x > 5)
     .for_each(|x| println!("{}", x));

// Exhaustive Pattern Matching
match my_option {
    Some(x) => println!("Got {}", x),
    None    => println!("Got nothing"),
    // If you delete 'None', the code refuses to compile.
}
```

## 5. Metaprogramming: Reflection vs. Macros
### Python: Runtime Reflection
Python allows you to inspect and modify code while it is running.
* `getattr(obj, "method_name")`
* Adding methods to classes at runtime ("Monkey Patching")
* Decorators wrap functions at load time

### Rust: Compile-Time Macros
Rust has zero runtime reflection (you cannot ask an object "what methods do you have?" at runtime). Instead, it uses Macros to generate code before the compiler sees it.
* `#[derive(Debug)]`: Writes the code to print the struct for you
* `println!`: This is a macro, not a function, so it can type-check your format string

## 6. Error Handling: Invisible Exceptions vs. Visible Results
### Python: Exceptions
Errors interrupt control flow and bubble up until caught. It is difficult to know which errors a function might throw just by looking at its signature.

```python
# Signature tells you nothing about errors
def read_file(path): 
    ...
```

### Rust: Result<T,E>
Errors are part of the function signature. The return type tells you exactly what can go wrong.

```rust
// Signature explicitly warns: "I might fail with an io::Error"
fn read_file(path: &str) -> Result<String, io::Error> {
    ...
}
```

## 7. Concurrency: The GIL vs. `Send` + `Sync`
### Python: Parallelism is Hard
* **The GIL**: Only one thread executes Python bytecode at a time. Multi-threading is concurrency (context switching), not parallelism (simultaneous execution)
* **Solution**: You must use `multiprocessing` (separate processes, separate memory) to use multiple cores

### Rust: Fearless Concurrency
* **No GIL**: Threads run in true parallel on all cores
* **Send & Sync**: These are "Marker Traits."
	* `Send`: It is safe to move this data to another thread
	* `Sync`: It is safe for multiple threads to access this data via reference
* **Compiler Guarantee**: If you try to pass a non-thread-safe type (like a Reference Counted pointer `Rc`) to another thread, the compiler stops you

## 8. Project Structure: Files vs. Modules
### Python: Filesystem = Module System
* A file data.py is automatically a module named data
* __init__.py turns a folder into a package
* Imports are resolved at runtime (you can have circular import crashes)

### Rust: Explicit Module Tree
* File structure does not automatically dictate module structure
* You must explicitly build the tree in `lib.rs` or `mod.rs` using `pub mod my_module`;
* This allows strictly controlling visibility (`pub(crate)`, `pub(super)`) beyond just "public" or "private."