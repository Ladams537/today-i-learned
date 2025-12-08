# Rust vs. C++: The Successor vs. The Titan

## 1. The Core Philosophy: "Trust" vs. "Verification"

| Feature | **C++ (The Old Guard)** | **Rust (The Modern Standard)** |
| :--- | :--- | :--- |
| **Philosophy** | "Trust the programmer." C++ assumes you know what you are doing and will let you do unsafe things for performance. | "Verify the programmer." Rust assumes humans make mistakes and prevents memory errors at compile time. |
| **Legacy** | **Backward Compatibility.** Must support code written in 1985. This leads to complex, redundant features. | **Fresh Start.** No legacy baggage. Can enforce stricter rules without breaking 30-year-old codebases. |
| **Memory** | **Manual / RAII.** You manage memory. Smart pointers (`std::unique_ptr`) help, but raw pointers and use-after-free bugs are still possible. | **Ownership & Borrowing.** The compiler tracks every variable's lifetime. Memory safety is mathematically guaranteed without a Garbage Collector. |
| **Undefined Behavior** | **Ubiquitous.** Signed integer overflow, out-of-bounds access, and data races result in "Undefined Behavior" (anything can happen). | **Contained.** Safe Rust eliminates UB. To do dangerous things (raw pointers), you must wrap code in an `unsafe { ... }` block. |

---

## 2. Default Behavior: Copy vs. Move

This is one of the most jarring shifts for C++ developers.

| Concept | C++ Implementation | Rust Implementation | Key Difference |
| :--- | :--- | :--- | :--- |
| **Assignment** | **Copy by Default.** `auto b = a;` runs the Copy Constructor. `a` is still valid. | **Move by Default.** `let b = a;` transfers ownership. `a` is now invalid (unless type is `Copy` like `int`). |
| **Semantics** | Explicit `std::move(a)` required to prevent copying. | Explicit `.clone()` required to force copying. |
| **Performance** | Accidental deep copies are a common performance bug in C++. | Rust forces you to see every deep copy in the code. |

```cpp
// C++
std::vector<int> v1 = {1, 2, 3};
auto v2 = v1; // Deep Copy! Expensive.
// v1 is still usable

```rust
// Rust
let v1 = vec![1, 2, 3];
let v2 = v1; // Move (Pointer copy only). Cheap.
// println!("{:?}", v1); // Compile Error! v1 is gone.
```

## 3. Pointers and References: The Borrow Checker
### C++: References are distinct from Pointers
C++ has References (`&`) and Pointers (`*`). References are safer but can still dangle

```C++
int* ptr = nullptr;
{
    int x = 10;
    ptr = &x;
}
// DANGER: ptr is now a "Dangling Pointer" pointing to invalid stack memory.
// *ptr = 20; // Undefined Behavior!
```

### Rust: References check Liveness
Rust combines pointers and references into a system verified by "Lifetimes".

```rust
let r;
{
    let x = 10;
    r = &x;
} 
// Compile Error: `x` does not live long enough.
// You literally cannot compile code that points to dead memory.
```

## 4. Generic Programming: Templates vs Traits
### C++: Templates (Duck Typing)
C++ Templates are powerful but permissive. They work if the syntax is valid after substitution. Errors are historically verbose (hundreds of lines of "template vomit").

```C++
// Works as long as T supports the '+' operator
template <typename T>
T add(T a, T b) { return a + b; }
```

### Rust: Generics with Traits (Bounded)
Rust Generics require you to prove capabilities upfront using Traits. The compiler checks the function definition, not just its usage.

```rust
// You MUST declare that T implements Add
fn add<T: std::ops::Add<Output = T>>(a: T, b: T) -> T {
    a + b
}
```

## 5. Error Handling: Exceptions vs Results
### C++: Exceptions
C++ uses `try/catch`. Exceptions are invisible in the function signature. They incur runtime overhead (stack unwinding) only when thrown.

```C++
// Can this throw? Who knows.
int result = dangerous_function();
```

### Rust: Result Enum
Rust returns errors as data. This is zero-cost (it's just a union return value). You must handle the error to access the success value.

```rust
// Signature tells the truth: It returns an Int OR an Error.
fn dangerous_function() -> Result<i32, MyError> { ... }

match dangerous_function() {
    Ok(val) => println!("{}", val),
    Err(e) => eprintln!("Failed: {}", e),
}
```

## 6. Project Structure: Header Files vs Modules
### C++: The Preprocessor (#include)
C++ literally copy-pastes text from header files (`.h`) into source files (`.cpp`).
* **ODR (One Definition Rule)**: If you define a function in a header without inline, you get linker errors
* **Build Speeds:** Slow, because headers are re-parsed for every file that includes them

### Rust: The Module System
Rust parses files as a proper syntax tree.
* **No Headers:** Interfaces and implementation live in the same `.rs` file
* **Visibility:** You simply mark functions `pub`
* **Crates:** Pre-compiled libraries are linked efficiently

## 7. Concurrency: Data Races
### C++: "Be Careful"
C++11 added `std::thread`, but it does not prevent data races. If two threads write to the same `int` without a Mutex, the program is invalid, but it compiles fine

```C++
// C++: Compiles, runs, might crash randomly
int data = 0;
std::thread t1([&]() { data++; });
std::thread t2([&]() { data++; });
```

### Rust: "Fearless Concurrency"
Rust enforces thread safety at compile time using the `Send` and `Sync` traits.

```rust
// Rust: Compile Error!
let mut data = 0;
// "data" cannot be borrowed mutably by two closures at once.
std::thread::spawn(|| data += 1);
std::thread::spawn(|| data += 1);

// Fix: You MUST wrap it in a Mutex and Arc (Atomic Reference Counter)
let data = Arc::new(Mutex::new(0));
```

## 8. Build Systems: CMake vs Cargo
### C++: CMake / Make
There is no standard C++ package manager.
* **Dependency Hell:** You often manually copy libraries or use complex CMake scripts to find them on the system
* **Build Scripts:** `CMakeLists.txt` is its own complex scripting language

### Rust: Cargo
Cargo is the industry standard, included with Rust.
* **Dependencies:** Add a line to `Cargo.toml` (`serde = "1.0"`). Cargo downloads, compiles, and links it
* **Consistency:** The same command (`cargo build`) works on Windows, Linux, and macOS

