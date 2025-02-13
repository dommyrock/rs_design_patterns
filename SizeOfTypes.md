
# Size of Rust types

```rust
use std::{cell::{Cell, RefCell}, marker::PhantomData, mem, rc::Rc, sync::Arc};

struct Zero{}
enum EZero{}
trait TNotZero{}

//Zero sized types

let s1 = mem::size_of::<()>(); //0
println!("unit {:?}", s1);

//Adding a PhantomData<T> field to your type tells the compiler that your type acts as though it stores a value of type T, even though it doesn't really.

let s1 = mem::size_of::<Zero>(); //0
println!("Zero {:?}", s1);

let s1 = mem::size_of::<EZero>(); //0
println!("EZero {:?}", s1);

let s1 = mem::size_of::<PhantomData<()>>(); //0
println!("phantomData () {:?}", s1);

let s1 = mem::size_of::<PhantomData<&i32>>(); //0
println!("phantomData &i32 {:?}", s1);

let closure = || println!("Not capturing anything!"); //0
println!("empty closure: {}", std::mem::size_of_val(&closure));

//---- Trait objects ----

//Nope - Not Zero ... (vtable needed)
let s1 = mem::size_of::<&dyn TNotZero>(); //16
println!("Tero {:?}", s1);

// 1. A trait object reference (fat pointer)
let closure = |x: i32| x + 1;
let trait_obj_ref: &dyn Fn(i32) -> i32 = &closure;
println!("Size of &dyn Fn: {}", mem::size_of_val(&trait_obj_ref)); // 16 bytes

// 2. A boxed trait object 
let boxed_trait: Box<dyn Fn(i32) -> i32> = Box::new(closure);
println!("Size of Box<dyn Fn>: {}", mem::size_of_val(&boxed_trait)); // 16 bytes

//---- Trait objects ----

let s1 = mem::size_of::<i32>(); //4
println!("i32 {:?}", s1);

let s1 = mem::size_of::<&()>(); //8
println!("ref {:?}", s1);

let s1 = mem::size_of::<*const i32>();
let s2 = mem::size_of::<*mut i32>();
println!("raw ptr const {:?} , mut {:?}", s1, s2);//8 8

let s1 = mem::size_of::<Option<&i32>>();
let s2 = mem::size_of::<Option<i32>>();
println!("Option<&i32> {:?} , Option<i32> {:?}", s1, s2);//8 8

let s1 = mem::size_of::<Box<&i32>>();
let s2 = mem::size_of::<Box<i32>>();
println!("box<&i32> {:?} , box<i32> {:?}", s1, s2);//8 8

let s1 = mem::size_of::<Rc<&i32>>();
let s2 = mem::size_of::<Rc<i32>>();
println!("Rc<&i32> {:?} , Rc<i32> {:?}", s1, s2);//8 8

let s1 = mem::size_of::<Arc<&i32>>();
let s2 = mem::size_of::<Arc<i32>>();
println!("Arc<&i32> {:?} , Arc<i32> {:?}", s1, s2);//8 8

let s1 = mem::size_of::<Cell<&i32>>();
let s2 = mem::size_of::<Cell<i32>>();
println!("Cell<&i32> {:?} , Cell<i32> {:?}", s1, s2);//8 4

let s1 = mem::size_of::<RefCell<&i32>>();
let s2 = mem::size_of::<RefCell<i32>>();
println!("RefCell<&i32> {:?} , RefCell<i32> {:?}", s1, s2);//16 16
```

## Reasoning behind outliers Cell , RefCell , Rc ,Arc

### 1. `Cell<&i32>` (8) vs. `Cell<i32>` (4)

#### **Understanding `Cell<T>` Layout**

- `Cell<T>` provides interior mutability by wrapping `T` in a type that allows mutation through shared references.
- It generally stores `T` **directly inside** the `Cell<T>`, rather than using a pointer indirection.

#### **Why the Size Difference?**

- `Cell<i32>`:
  - Since `i32` is a plain integer (size: `4` bytes), `Cell<i32>` **just stores the `i32` directly**.
  - The size of `Cell<i32>` is therefore **4 bytes**, matching the size of an `i32`.

- `Cell<&i32>`:
  - A reference (`&i32`) is a pointer, which takes **8 bytes** on a 64-bit system.
  - `Cell<&i32>` stores the pointer **directly** in the `Cell`, making its size **8 bytes**.

- **Comparison with `Rc` and `Arc`**:
  - `Rc<T>` and `Arc<T>` **do not store the data directly**. Instead, they store **a pointer** to the heap-allocated `T`, so their size is **always 8 bytes**, regardless of `T`.
  - `Cell<T>`, on the other hand, stores `T` **inside itself**, so its size depends on `T`.

### 2. `RefCell<&i32>` (16) vs. `RefCell<i32>` (16)

#### **Understanding `RefCell<T>` Layout**

- `RefCell<T>` enables interior mutability but **adds runtime borrow tracking**.
- It contains:
  1. The actual `T` **stored inline**.
  2. A borrow flag (used for runtime borrow checking).

#### **Why the Size is 16 bytes?**

- The extra space comes from the **borrow flag** (likely an `usize`, 8 bytes on 64-bit systems).
- Let's break it down:
  - `RefCell<i32>`:
    - `i32` is `4` bytes.
    - Borrow flag is `8` bytes.
    - Likely padding added to align memory properly → **total size: 16 bytes**.
  - `RefCell<&i32>`:
    - `&i32` (a pointer) is `8` bytes.
    - Borrow flag is `8` bytes.
    - No padding needed → **total size: 16 bytes**.

#### **Why Doesn't `Rc<T>` or `Arc<T>` Grow Like `RefCell<T>`?**

- `Rc<T>` and `Arc<T>` store a pointer to the heap, and **their reference count is stored in the heap allocation, not in the struct itself**.
- `RefCell<T>` needs to store the borrow flag **inside** the struct, leading to the extra space requirement.

---

### **Summary**

| Type               | `T = i32` | `T = &i32` | Why? |
|-------------------|----------|------------|------|
| `Cell<T>`        | 4        | 8          | Stores `T` inline. A reference is a pointer (8 bytes), but `i32` is just 4 bytes. |
| `Rc<T>` / `Arc<T>` | 8      | 8          | Always a pointer (8 bytes), because the actual value is heap-allocated. |
| `RefCell<T>`      | 16       | 16         | Stores `T` inline + a borrow flag (`usize` = 8 bytes) |

## Trait object size `&dyn TNotZero` (**16 bytes** on a 64-bit system)

**Why is `&dyn TNotZero` 16 bytes?** </br>
A **trait object** like `&dyn TNotZero` consists of **two pointers**, making it **16 bytes** in size on a 64-bit system.

#### **1. What is a Trait Object?**
- A **trait object** allows dynamic dispatch, meaning the actual type implementing the trait is **not known at compile time**.
- To achieve this, Rust needs **both**:
  1. A **data pointer** → points to the actual value (of some type that implements `TNotZero`).
  2. A **vtable pointer** → points to a table of function pointers (used for dynamic dispatch).

#### **2. Memory Layout of `&dyn TNotZero`**
A reference to a trait object (`&dyn TNotZero`) is a **fat pointer** consisting of:
- **Data Pointer (8 bytes)** → Points to the actual concrete type implementing `TNotZero`.
- **VTable Pointer (8 bytes)** → Points to the virtual table (vtable) containing function pointers for `TNotZero`.

Since each pointer is **8 bytes** on a 64-bit system, the total size of `&dyn TNotZero` is:

8 **(data pointer)**  + 8  **(vtable pointer)** = 16  bytes

---

### **Is `&dyn TNotZero` a Trait Object?**
Yes! A **trait object** in Rust is any instance of a **trait used as a type** (e.g., `&dyn TNotZero`, `Box<dyn TNotZero>`, `Rc<dyn TNotZero>`, etc.).  
Trait objects enable **dynamic dispatch**, meaning method calls are resolved at runtime rather than at compile time.

- **Normal references (e.g., `&i32`)** → 8 bytes (single pointer).
- **Trait objects (e.g., `&dyn TNotZero`)** → 16 bytes (two pointers).

---

### **Comparison with Other Types**
| Type                | Size on 64-bit System | Explanation |
|---------------------|----------------------|-------------|
| `&i32`             | 8 bytes               | Single pointer |
| `&[i32]` (slice)   | 16 bytes              | Fat pointer: (ptr + length) |
| `&dyn Trait`       | 16 bytes              | Fat pointer: (ptr + vtable) |
| `Box<dyn Trait>`   | 16 bytes               | Single pointer (heap-allocated, but still stores fat pointer) |
| `Rc<dyn Trait>`    | 16 bytes               | Same as `Box<dyn Trait>` |
| `Arc<dyn Trait>`   | 16 bytes               | Same as `Box<dyn Trait>` |

---

# Function pointers fn | Fn | FnOnce | FnMut

## **1. Function Pointer Types in Rust**
Rust has several ways to represent functions:

### **A. Function Pointers (`fn(T) -> U`)**
- A function pointer is a simple pointer to a function.
- **Size: Always 8 bytes** on a 64-bit system.
- Example:
  ```rust
  let s = std::mem::size_of::<fn(i32) -> i32>();
  println!("fn(i32) -> i32: {}", s); // Output: 8
  ```
- Since a function’s memory location is constant, a function pointer just stores that address, making its size 8 bytes.

---

### **B. Trait Objects for Closures (`dyn Fn`, `dyn FnMut`, `dyn FnOnce`)**
Unlike function pointers, **closures in Rust can capture their environment**. This means their representations vary depending on what they capture.

#### **1. `&dyn Fn(T) -> U` (Immutable Closure Trait Object)**
- **Size: 16 bytes** (fat pointer).
- **Why?**
  - `&dyn Fn` stores:
    1. **Pointer to the closure data (environment)**
    2. **Pointer to the vtable** (needed for dynamic dispatch)
  - This is **the same as a trait object (`&dyn Trait`)**.
- Example:
  ```rust
  let s = std::mem::size_of::<&dyn Fn(i32) -> i32>();
  println!("&dyn Fn: {}", s); // Output: 16
  ```

#### **2. `&dyn FnMut(T) -> U` (Mutable Closure Trait Object)**
- **Size: 16 bytes** (same as `&dyn Fn`).
- **Why?**
  - `FnMut` closures may modify their environment, but the structure of the trait object remains the same.
- Example:
  ```rust
  let s = std::mem::size_of::<&dyn FnMut(i32) -> i32>();
  println!("&dyn FnMut: {}", s); // Output: 16
  ```

#### **3. `&dyn FnOnce(T) -> U` (One-Time Callable Closure Trait Object)**
- **Size: 16 bytes** (same as `&dyn Fn` and `&dyn FnMut`).
- **Why?**
  - `FnOnce` means the closure consumes its environment, but the fat pointer representation remains unchanged.
- Example:
  ```rust
  let s = std::mem::size_of::<&dyn FnOnce(i32) -> i32>();
  println!("&dyn FnOnce: {}", s); // Output: 16
  ```

---

### **C. Boxed Function Trait Objects (`Box<dyn Fn>` etc.)**

For unsized types (like trait objects), the Box holds a fat pointer.

#### **1. `Box<dyn Fn(T) -> U>`**
- Example:
  ```rust
  let s = std::mem::size_of::<Box<dyn Fn(i32) -> i32>>();
  println!("Box<dyn Fn>: {}", s); // Output: 16
  ```

#### **2. `Box<dyn FnMut(T) -> U>`**
- Example:
  ```rust
  let s = std::mem::size_of::<Box<dyn FnMut(i32) -> i32>>();
  println!("Box<dyn FnMut>: {}", s); // Output: 16
  ```

#### **3. `Box<dyn FnOnce(T) -> U>`**
- Example:
  ```rust
  let s = std::mem::size_of::<Box<dyn FnOnce(i32) -> i32>>();
  println!("Box<dyn FnOnce>: {}", s); // Output: 16
  ```

---

### **D. Capturing Closures (Heap Allocated)**
If a closure captures **variables from its environment**, its size depends on those captured variables.

- Example:
  ```rust
      let x: i32 = 42;
    let closure = || println!("{}", x);
    println!("Size of closure: {}", std::mem::size_of_val(&closure));// 8 bytes
    let (x,y,z)= (42,43,44);
    let closure = || println!("{} {} {}", x, y, z);
    println!("Size of closure: {}", std::mem::size_of_val(&closure));// 24 bytes
  ```
  - If `x` is an `i32`, then the closure will **store `x` inside itself**, increasing its size.
  - For instance, if the closure captures an `i32`, it might be **4 bytes (i32) + alignment padding**.

---

## **Comparison Table**
| Type                 | Size on 64-bit System | Why? |
|----------------------|----------------------|------|
| `fn(i32) -> i32`    | 8 bytes               | Function pointer (just an address). |
| `&dyn Fn(i32) -> i32` | 16 bytes              | Fat pointer (closure env + vtable). |
| `&dyn FnMut(i32) -> i32` | 16 bytes          | Same as `&dyn Fn` (fat pointer). |
| `&dyn FnOnce(i32) -> i32` | 16 bytes         | Same as `&dyn Fn` (fat pointer). |
| `Box<dyn Fn(i32) -> i32>` | 16 bytes          | Single pointer to heap. |
| `Box<dyn FnMut(i32) -> i32>` | 16 bytes       | Same as `Box<dyn Fn>`. |
| `Box<dyn FnOnce(i32) -> i32>` | 16 bytes      | Same as `Box<dyn Fn>`. |
| **Closure capturing nothing** | 0 bytes     | Empty closures have zero size. |
| **Closure capturing an `i32`** | 8 bytes     | Stores captured variable inline. |

---

## **Summary**
1. **Function Pointers (`fn`) are lightweight** (8 bytes) since they store only the function address.
2. **Trait Objects (`dyn Fn`, `dyn FnMut`, `dyn FnOnce`) use fat pointers** (16 bytes) to support dynamic dispatch.
3. **Boxed closures reduce size to 8 bytes** (single pointer to heap allocation).
4. **Capturing closures** increase in size based on captured variables.

---

Rust now also provides async fn pointers and has some more neat tricks on wrapping them in `Rc<impl Fn>` to avoid unnecessary **clone** (which can be large due to the size of the captured scope).</br>

Example

```rust
fn make_input_clone(f: impl Fn() + Clone + 'static) -> Input {
    let f = Rc::new(f);
    Input {
        on_submit: f.clone(),
        on_key: Rc::new(move |key_code: u32| {
            if key_code == 13 {
                f();
            }
        }),
    }
}
```

[Understand Rust closures and function pointers](https://www.youtube.com/watch?v=N3rXqmfzeSs)

---

## When would you use each fn pointer or Fn trait object?

> **Note:**  

> - **FnOnce** is a bit special because its call method takes ownership (by value). (In most cases, you’ll use Fn or FnMut as trait objects because FnOnce isn’t object safe for calling.)  

---

Examples

## A. Plain Function Pointer (`fn`)

### **When to use:**  
Use a function pointer when you have a plain, stateless function. They are very lightweight and have a fixed size.

### **Example:**
```rust
// A regular function that adds one.
fn add_one(x: i32) -> i32 {
    x + 1
}

fn main() {
    // Create a function pointer to `add_one`
    let func: fn(i32) -> i32 = add_one;
    println!("Function pointer result: {}", func(5)); // prints 6
}
```

---

## B. Immutable Closure Trait Object (`&dyn Fn`)

### **When to use:**  
Use `&dyn Fn` when you have a closure that does not mutate its environment. It enables dynamic dispatch (the exact closure need not be known at compile time).

### **Example:**
```rust
fn main() {
    // A simple closure that adds one without capturing any environment.
    let closure = |x: i32| x + 1;
    // Create a trait object reference.
    let func: &dyn Fn(i32) -> i32 = &closure;
    println!("&dyn Fn result: {}", func(5)); // prints 6
}
```

---

## C. Mutable Closure Trait Object (`&mut dyn FnMut`)

### **When to use:**  
Use `&mut dyn FnMut` when your closure needs to modify (or “mutate”) its captured environment. The mutable trait object allows the closure to update its internal state.

### **Example:**
```rust
fn main() {
    let mut count = 0;
    // A closure that mutates its environment (updates `count`).
    let mut closure = |x: i32| {
        count += 1;
        x + count
    };
    // Borrow the closure as a mutable trait object.
    let func: &mut dyn FnMut(i32) -> i32 = &mut closure;
    println!("First call (&mut FnMut): {}", func(5)); // count becomes 1, prints 6
    println!("Second call (&mut FnMut): {}", func(5)); // count becomes 2, prints 7
}
```

---

## D. FnOnce Trait Object (`&dyn FnOnce`)

### **When to use:**  
FnOnce closures “consume” captured values (for example, by moving them) and are callable only once. In practice, **FnOnce is not object safe** (because its call method consumes `self`), so you rarely use a trait object of type `&dyn FnOnce`. We show a demonstration only for completeness.

### **Example:**
```rust
fn main() {
    let x = String::from("Hello");
    // A closure that moves `x`, making it a FnOnce closure.
    let closure = move |s: &str| {
        // This closure can only be called once because it takes ownership of `x`
        println!("{} {}", x, s);
    };

    // Creating a reference to a FnOnce trait object is only for demonstration.
    // (You cannot actually call `f.call_once(...)` via a &dyn FnOnce.)
    let _f: &dyn FnOnce(&str) = &closure;
    println!("Created a &dyn FnOnce trait object (not callable via trait object)!");
}
```

> **Tip:** In practice, if you need to call a closure more than once or store it in a trait object, design it so that it implements Fn or FnMut rather than FnOnce.

---

## E. Boxed Trait Objects for Closures

Boxing is useful when the closure’s size isn’t known at compile time or you want to return a closure from a function.

### **1. Boxed Immutable Closure (`Box<dyn Fn>`)**

#### **When to use:**  
When you want to store an immutable closure on the heap.

#### **Example:**
```rust
fn main() {
    let closure = |x: i32| x * 2;
    let func: Box<dyn Fn(i32) -> i32> = Box::new(closure);
    println!("Box<dyn Fn> result: {}", func(5)); // prints 10
}
```

### **2. Boxed Mutable Closure (`Box<dyn FnMut>`)**

#### **When to use:**  
When you need a heap-allocated closure that can modify its internal state.

#### **Example:**
```rust
fn main() {
    let mut multiplier = 1;
    let mut closure = move |x: i32| {
        multiplier *= 2;
        x * multiplier
    };
    let mut func: Box<dyn FnMut(i32) -> i32> = Box::new(closure);
    println!("Box<dyn FnMut> result: {}", func(5));
}
```

### **3. Boxed FnOnce Closure (`Box<dyn FnOnce>`)**

#### **When to use:**  
Boxed FnOnce closures are rare because FnOnce isn’t object safe for calling.
They might be used when you need to return a closure that consumes captured variables and you’re okay with calling it only once (using workarounds).

#### **Example:**
```rust
fn main() {
    let x = 10;
    let closure = move |y: i32| x + y;
    // Box the closure.
    // Note: Although we can box it, calling it via the trait object is not straightforward
    // because FnOnce’s call method takes ownership.
    let _func: Box<dyn FnOnce(i32) -> i32> = Box::new(closure);
    println!("Created Box<dyn FnOnce> trait object (not directly callable)!");
}
```

---

## F. Capturing Closures

Closures that capture environment variables have sizes that depend on what they capture.

### **1. Closure Capturing Nothing**

#### **When to use:**  
When no external variables are needed. Such closures are very lightweight and can sometimes even be converted to function pointers.

#### **Example:**
```rust
fn main() {
    let closure = || println!("Hello, world!");
    closure();
    println!("Size of empty closure: {}", std::mem::size_of_val(&closure));
}
```

### **2. Closure Capturing an `i32`**

#### **When to use:**  
When you need to capture a value from the surrounding scope. The closure’s size will include the captured value(s).

#### **Example:**
```rust
fn main() {
    let x = 42;
    let closure = || println!("Captured x = {}", x);
    closure();
    println!("Size of closure capturing an i32: {}", std::mem::size_of_val(&closure));
}
```

---

## Comparison Table

| Type                              | Size (64‑bit) | When/Why to Use It                                                                                                                                                           |
|-----------------------------------|---------------|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| **`fn(i32) -> i32`**              | 8 bytes       | Plain function pointer for a static function that does not capture environment.                                                                                             |
| **`&dyn Fn(i32) -> i32`**         | 16 bytes      | Trait object for an immutable closure; use when you need dynamic dispatch for read‑only operations.                                                                          |
| **`&mut dyn FnMut(i32) -> i32`**    | 16 bytes      | Trait object for a mutable closure; use when the closure modifies its environment.                                                                                         |
| **`&dyn FnOnce(&str)`**           | 16 bytes*     | Trait object for a FnOnce closure (not normally callable via trait object because FnOnce isn’t object safe). Demonstration only.                                              |
| **`Box<dyn Fn(i32) -> i32>`**      | 16 bytes       | Heap‑allocated immutable closure; useful for returning closures or when size is unknown at compile time.                                                                     |
| **`Box<dyn FnMut(i32) -> i32>`**   | 16 bytes       | Heap‑allocated mutable closure; similar to Box<dyn Fn> but allows internal mutation.                                                                                        |
| **`Box<dyn FnOnce(i32) -> i32>`**  | 16 bytes*      | Heap‑allocated FnOnce closure; rarely used because FnOnce is not object safe. (*Trait objects for FnOnce are generally for demonstration only.)                              |
| **Empty capturing closure**       | ~0 bytes      | Closure that captures nothing—minimal overhead and sometimes convertible to a function pointer.                                                                           |
| **Closure capturing an `i32`**    | ≥4 bytes      | Closure that captures an `i32` from the environment; size includes the captured value plus any alignment/padding.                                                             |

*The FnOnce trait objects are shown for completeness.</br> In practice, you typically work with Fn or FnMut trait objects because `FnOnce's` call signature **(taking self by value) prevents it from being used as a trait object** in most cases.

---

# Reasoning about `&dyn Fn` and `Box<dyn Fn>` pointer sizes

Let's break down what’s happening under the hood and how boxing changes the story.

---

## 1. Fat Pointers for Trait Objects

When you write a reference to a trait object like `&dyn Fn(i32) -> i32`, Rust doesn’t just store one pointer. Instead, it creates a **fat pointer** that contains two pieces of information:

1. **Data Pointer (8 bytes on 64‑bit systems):**  
   Points to the actual closure or function data in memory.

2. **Vtable Pointer (8 bytes on 64‑bit systems):**  
   Points to a *vtable*—a table of function pointers that the compiler has generated. This table tells Rust which concrete method implementations to call when you invoke methods on the trait object.

So a reference like `&dyn Fn(i32) -> i32` is 16 bytes in total (8 bytes for the data pointer + 8 bytes for the vtable pointer).

---

## 2. Where Is the Vtable Stored?

- **Vtable Location:**  
  The vtable is created by the compiler for each concrete type that implements the trait. It is stored in the binary’s read-only data section (often called the “rodata” section).  
- **One per Concrete Type:**  
  Every type that appears as a trait object gets its own vtable, but note that it’s not duplicated every time you have a trait object reference. The same vtable is reused for all references (or boxes) pointing to that type.
- **Cost:**  
  The vtable is just a static table of pointers (typically a few dozen bytes total for a trait with several methods). Even though each fat pointer holds an extra 8-byte pointer, that 8 bytes per trait object isn’t “duplicating” the whole vtable—it’s simply carrying a pointer that tells you where to look.

---

## 3. Boxing and Its Impact on Pointer Size

### How Box<dyn Trait> Differs from &dyn Trait

- **`&dyn Trait` (Fat Pointer):**  
  When you have a reference (or a function parameter) of type `&dyn Fn(i32) -> i32`, it’s a fat pointer (16 bytes) because it directly contains both the data pointer and the vtable pointer.

- **`Box<dyn Trait>` (Owned Pointer):**  
  A `Box<T>` is a smart pointer that owns its data on the heap. 

---

## 4. Why Does the Vtable Pointer Add 8 Bytes?

Every time you have a trait object reference (like `&dyn Fn`), the pointer must carry along the address of the vtable. The vtable pointer is necessary because:
- **Dynamic Dispatch:**  
  It tells Rust how to call the right method implementation for the concrete type behind the trait object.
- **Runtime Flexibility:**  
  Since trait objects enable polymorphism (the actual type isn’t known until runtime), the vtable pointer is crucial for determining behavior on the fly.

Even though an 8-byte pointer might seem “heavy” compared to a single 8-byte pointer, remember:
- The **vtable itself is static**; it exists in one location per concrete type.
- The **fat pointer duplicates only the pointer to the vtable**, not the entire table. That extra pointer (8 bytes) is the cost for enabling dynamic dispatch without losing type safety.

---

## Summary

- **&dyn Fn:**  
  - **Size:** 16 bytes (8 bytes data pointer + 8 bytes vtable pointer).  
  - **Use Case:** When you need a trait object reference for dynamic dispatch.

- **Box<dyn Fn>:**  
  - **Size:** 8 bytes on the stack.  
  - **Use Case:** When you want to allocate the closure or function on the heap (e.g., to hide its concrete type, for returning from functions, or when its size isn’t known at compile time).  
  - **How It Works:** The Box stores only a thin pointer on the stack; the vtable pointer is stored in the heap allocation (or known by the compiler based on the trait object’s type), so you don’t need to carry it around on the stack every time.

Boxing lets you avoid repeatedly storing the 8-byte vtable pointer on the stack. </br>
Instead, you store it once (or rely on the compiler’s type information) and only pass around an 8-byte pointer to the heap—improving the size of the pointer in your local scope without sacrificing the dynamic dispatch capabilities.

</br>
A combination of my own examples translated into MD format and synthetic reasoning on top of them.

---

# Comparing **Trait objects** and static dispatch

1. **`&dyn Fn`**  
2. **`Box<dyn Fn>`**  
3. **`impl Fn()`**

## Summary

- **`&dyn Fn`** is ideal for **borrowing** a callable; you pay the cost of a fat pointer (16 bytes) and dynamic dispatch on every call.  
- **`Box<dyn Fn>`** is great when you need **ownership** and flexibility (like storing or returning callables) while keeping your stack usage minimal.
- **`impl Fn`** hides the concrete type but allows **static dispatch**, which can lead to better performance and potentially zero runtime overhead if the closure is small or captures nothing.

</br>

by `Dominik Polzer`