# Lifetimes are kinda like types

```rust
fn my_push(vector: &mut Vec<&String>, element: &String) {
   vector.push(element);
}
fn main() {
   let my_string = String: :from("foo");
   let mut my_vec = Vec::<&String>::new();
   my_push(&mut my_vec, &my_string) ;
   // drop(my_string);
   println! ("[:?}", my_vec);
}
```

## Manualy typing out the lifetimes

This would be not working (it's the 'named' version of HIR emmited code with 3x different lifetimes)

```rust
fn my_push<'a, 'b, 'c>(vector: &'c mut Vec<&'b String>, element: & 'a String) {
   vector.push(element);
}
```

Correct version (which signals that 'element' lives at least as long as `Vec<&'a String>` )

```rust
fn my_push<'a,'c>(vector: &'c mut Vec<&'a String>, element: & 'a String) {
   vector.push(element);
}
```

We don't have any requirements on lifetime 'c , so we can just delete it an let compiler emit this for us.</br>
We only needed to name one lifetime `'a` param.

```rust
fn my_push<'a>(vector: & mut Vec<&'a String>, element: & 'a String) {
   vector.push(element);
}
```

### ERROR that we could make

Is to put `&'a` lifetime on the `Vec<T>` itself.

```rust
fn my_push<'a>(vector: &'a mut Vec<&'a String>, element: & 'a String) {
   vector.push(element);
}
//later on ...in the code you move my_vec 
println!("{:?}",my_vec);
```

- This will thow : `cannot borrow 'my_vec' as immutable bexause it is also borrowed as mutable`
- Because. This breaks NO MUTABLE ALIASING RULE.

</br>

---

### Lowering Rsust code to HIR / MIR

Lowering the code to MIR, HIR , HIR,Typed

MIR

```bash
rustc -Z unpretty=mir main.rs
```

HIR
[Godbolt - Emitted HIR and MIR](https://godbolt.org/z/WK9q57WMa)

```bash
rustc -Z unpretty=hir main.rs

### HIR output - You can kinda see lifetime elision here (3x unique lifetimes were assigned to fn)

fn my_push<'_, '_, '_>
   (vector:  &'_ mut Vec<&'_ String>,
    element: &'_ String) {
    // vector .push(element);
}
```

1. **First Lifetime ('_' for vector parameter):** This is the lifetime of the reference to the vector itself (&'_ mut Vec<...>`).
2. **Second Lifetime** (inner `'_` in `&'_` String within the vector):
The vector holds references to Strings. This inner reference gets its own lifetime parameter.
3. **Third Lifetime ('_' for elementparameter):** This is the lifetime of the element parameter (&'_ String`).

- **Separation of Lifetimes**: The fact that `three lifetimes appear` shows that Rust treats the lifetime of the container (vector), the lifetime of the elements inside the container, and the lifetime of the element argument separately. This separation is crucial for ensuring memory safety.

Even though you see two parameters in the function signature, one of them (vector) contains a reference type (&'_ String) inside the vector. This inner reference has its own lifetime parameter. That’s why you see three lifetime parameters in total.

[Lifetime elision - nomicon](https://doc.rust-lang.org/nomicon/lifetime-elision.html)

HIR,Typed

```bash
rustc -Z unpretty=hir,typed main.rs
```

---

#### Other rustc `unpretty` flags

For documentation purposes here are suppoerted CLI commands:

```bash
>> rustc -Z unpretty=mir,typed main.rs

error: argument to `unpretty` must be one of `normal`, `identified`, `expanded`, `expanded,identified`, `expanded,hygiene`, `ast-tree`, `ast-tree,expanded`, `hir`, `hir,identified`, `hir,typed`, `hir-tree`, `thir-tree`, `thir-flat`, `mir`, `stable-mir`, or `mir-cfg`; got mir,typed
```

#### Other rustc `emit` flags

```bash
>>rustc --emit=hir main.rs

error: unknown emission type: `hir` - expected one of: `llvm-bc`, `thin-link-bitcode`, `asm`, `llvm-ir`, `mir`, `obj`, `metadata`,`link`, `dep-info`
```

---

### Refs

[A First look at the lifetimes - Jack OConnor](https://www.youtube.com/watch?v=-gkvOoxgp8E) (Great intro to lifetimes)

