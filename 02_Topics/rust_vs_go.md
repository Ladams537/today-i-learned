# Rust vs. Go: The Systems Programmer's Rosetta Stone

## 1. The Core Philosophy: "Boring" vs. "Strict"

| Feature | **Go (The Professional Kitchen)** | **Rust (The High-Tech Lab)** |
| :--- | :--- | :--- |
| **Goal** | Throughput, Readability, Speed of Development. | Correctness, Memory Safety, Speed of Execution. |
| **Motto** | "Boring is good." / "Keep it simple." | "If it compiles, it works." |
| **Memory** | **Garbage Collected.** You create trash; the runtime cleans it up later. | **Ownership & Borrowing.** You clean up your own trash (automatically) when variables go out of scope. |
| **Complexity** | Low floor, low ceiling. Code looks the same regardless of who wrote it. | High floor, high ceiling. Powerful abstractions allow for expressive (but complex) code. |

---

## 2. Data Structures (The Containers)

This is the most important translation table for LeetCode.

| Concept | Go Implementation | Rust Implementation | Key Difference |
| :--- | :--- | :--- | :--- |
| **Growable List** | `[]int` (Slice) | `Vec<i32>` | Go slices are "windows" into arrays. Rust Vecs are owners of heap memory. |
| **Fixed List** | `[5]int` (Array) | `[i32; 5]` (Array) | Arrays are rarely used in Go. In Rust, they are used for stack allocation performance. |
| **Hash Map** | `map[string]int` | `HashMap<String, i32>` | Go has built-in syntax. Rust requires `use std::collections::HashMap`. |
| **String** | `string` | `String` (Owned) vs `&str` (View) | Go strings are immutable bytes. Rust forces you to decide if you own the text or are just reading it. |
| **Tuple** | N/A (Uses Structs) | `(i32, bool)` | Rust uses tuples often for grouping data without a full struct definition. |

---

## 3. Iteration: Loops vs. Pipelines

### Go: "There is only one loop."
Go is imperative. You tell the computer *how* to do it step-by-step.

```go
// The standard loop
nums := []int{1, 2, 3}
for i, v := range nums {
    // i is index, v is value
    fmt.Println(i, v)
}

// The "While" loop equivalent
x := 0
for x < 10 {
    x++
}
```

### Rust: "Everything is an Iterator." 
Rust is functional/declaritive. You describe what you want to happen. 

```
let nums = vec![1, 2, 3]; 

// 1. The Imperative Way (like Go)
for (i, v) in nums.iter().enumerate() {
    println!("{} {}", i, v);
}

// 2. The Functional Way (Preferred)
// "Map" transforms, "Filter" selects, "Collect" builds the result.
let doubled: Vec<i32> = nums.iter()
    .map(|x| x * 2)
    .filter(|x| x > 4)
    .collect();
```

## 4. Null Safety: The Billion Dollar Mistake
### Go: Implicit Nulls (```nil```)
Go uses ```nil```. You must remember to check for it, or your program will panic at runtime. 

```go
var ptr *int = nil

if ptr != nil {
    fmt.Println(*ptr)
}
// If you forget the check -> Runtime Panic!
```

### Rust: Explicit Option (```Option<T>```)
Rust does not have ```null```. It has the ```Option``` enum. You literally cannot use the value without hndling the "null" case.

```rust
let val: Option<i32> = None;

// You MUST unwrap or match
match val {
    Some(x) => println!("{}", x),
    None    => println!("Nothing here"),
}
```

## 5. Error Handling: Checking vs Unwrapping
### Go: Errors are Values
Errors are just variables returned alongside your data. 
```go
// Function returns (Value, Error)
file, err := os.Open("file.txt")
if err != nil {
    return err // Manual propagation
}
// Do stuff with file...
```

### Rust: Errors are Types (```Result<T, E>```)
Errors are wrapped in a Result enum. You the ```?``` operator to propagate them automatically. 

```rust
// The '?' means: "If error, return error immediately. If ok, give me the value."
let file = File::open("file.txt")?;
```

## 6. Object Orientation: Ducks vs ID Cards
Both languages reject Inheritance (Classes). Both use Compostion (Structs). The difference is **Polymorphism**. 

### Go: Implicit Interfaces (Duck Typing)
- **Philosophy:** "If it walks like a duck, it is a duck."
- You do not need to say ```Dog implements Animal```. You just write the methods

```go
type Animal interface { Speak() }

type Dog struct {}
func (d Dog) Speak() { fmt.Println("Woof") } 

// Dog is automatically an Animal because it has the Speak method.
```

### Rust: Explicit Traits (The ID Card)
- **Philosophy:** "Show me your License."
- You must explicitly implement a Trait for a Struct

```
trait Animal { fn speak(&self); }

struct Dog;
impl Animal for Dog { // Explicit linkage
    fn speak(&self) { println!("Woof"); }
}
```

## 7. Projects Structure and Visibility
### Go: The "Bucket" Method
- **Logic:** The **Folder** is the package
- **Visibility:** Capitalized functions (```Func```) are Public. Lowercase (```func```) are Private to the package
- **Imports:** All files in the same folder see each other automatically

### Rust: The "Tree" Method
- **Logic:** The Module Graph is explicit
- **Visibility:** Everything is Private by default. You must use pub to share
- **Imports:** You must manually register files in ```mod.rs``` or ```lib.rs``` (```mod my_file;```) to make the compiler see them

## 8. Concurrenct: Green vs OS Threads
### Go: Goroutines
- **Model:** "Green Threads" managed by the Go Runtime
- **Cost:** Cheap. You can spawn 100,000 goroutines easily
- **Communication:** Channels (```chan```). "Don't communicate by sharing memory; share memory by communicating."

### Rust: OS Threads / Async
- **Model:** 1:1 mapping to Operating System threads (```std::thread```) OR State Machine futures (Async/Tokio)
- **Cost:** OS threads are heavy (context switching). Async is zero-cost at runtime but complex to write
- **Safety:** Rust guarantees thread safety at compile time using the ```Send``` and ```Sync``` traits. If you have a data race, the code won't compile

