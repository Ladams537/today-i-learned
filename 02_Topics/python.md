## Functions
### args and kwargs
* Star is doing the heavy lifting in this
* Only handles **positional** argument
* Star is what tells Python to pack everything extra into the object called "args"
* Collected into tuple

**kwargs
* allows for keyword collection
* collects extra arguments into a **dictionary**
* double-star does the work here. Parameter name does not matter

-> can combine these together to collect everything
-> must always place **positional** before **keyword** arguments
-> applies both to the function definition and calls

#### * Operators
* in function definitions they pack 
* in function calls they unpack

Writing your own code use these only where you need **flexibility**. If you know in advance what arguments a function **needs**, use explicit parameters.

### Decorators
Similar to *Attributes* in Rust e.g. `#[derive()]` using the @ symbol.

### Lambda Functions
Similar to **Closures** in Rust. Use once functions that do not have a signature and have several powerful integrated tools for manipulating data.

## Memory Model
### Allocation & Copy
* everything in Python is an **object** on the **Heap**
* variables are just **references (pointers)** on the **Stack**
-> Rust, default operation is `move`
-> Python: default operation is `copy`

### Interning
* pre-allocates (caches) common immutable objects to save memory
* instead of creating a new object every time, it points you to the existing one
  * applies to:
  * small integers (-5 to 256)
  * short strings (like variable names)

`is` checks for **memory address identity**
`==` checks for **value identity**

### Reference Counting
* in Rust memory is freed when a variable goes of scope (`Drop` trait) -> deterministic
* in Python, memory management relies primarily on **reference counting**

* **ref counting**: every object has a counter on its struct (`ob_refcnt`). When you do `y = x`, the count goes up. When `y` goes out of scope or is reassigned, the count goes down. When it hits 0, memory is freed immediately
* **cycle detector**: if object A points to B, and B points to A, their ref counts never hit 0. Python has a background GC that periodically pauses execution to hunt for these isolated cycles and kill them

#### Larger? -> blew my mind
In C or Rust, a `char` is just a byte (or 4 for `char` in Rust), and an integer is 4 or 8 bytes. Tiny.

In Python, seeing as they are objects:
* **Int(`28`)**: ~28bytes (Header + Value)
* **String(`"a"`): ~50bytes (Header + + Length + Hash + Encoding info + Value)

Strings are heavier because they pre-calculate and store their hash (to make them fast dictionary keys) and track encoding.

## Data Types
**Sets** are distinct in Python:
* Unordered
* Unique (no duplicates)
* Implemented as a **Hash Map** (just like `dict`, but with dummy values)

nums.sort() -> modifies list **in-place** and returns `None`.
sorted(nums) -> creates and returns a **new list**, leaving the original untouched.

### Four Built-In Types
* List: [] -> ordered, changeable, duplicates
* Tuple: (..,) -> ordered, unchangeable, duplicates
* Set: {..,} -> unordered, unchangeable, unindexed
* Dict: {"..":"..",} -> ordered, changeable, no duplicates

## Error Handling
