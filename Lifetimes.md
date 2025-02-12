# Lifetimes are kinda like types

```rust
fn main(){
    let my_str :String = "ananas is blue".to_string();
    let mut vec:Vec<&str> = Vec::new();
    vec.push(&my_str);
    drop(my_str);
    println!("{:?}",vec); //Vec still contains reference to non existent object (use after free c++..)
}
```

Error we get for this in Rust is ...

```txt
>> cannot move out of `my_str` because it is borrowed
         let mut vec = Vec::<&str>::new();
9  |     vec.push(&my_str);
   |              ------- borrow of `my_str` occurs here
10 |     //push_vec(&mut vec, a);
11 |     drop(my_str);
   |          ^^^^^^ move out of `my_str` occurs here
12 |     println!("{:?}",vec);
   |                     --- borrow later used here
```

## Usingn our own `my_push` fn instead

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

We Get compiler error complaining about lifetimes of our `fn references`.</br>
Even if we cleared our main we would still get the compiler erorr.

```rust
fn push_vec(vec: &mut Vec<&str>, item: &str){
    vec.push(item);
}

fn main(){
}
```

```txt
>> `lifetime may not live long enough`
fn push_vec(vec: &mut Vec<&str>, item: &str){
  |                       -            - let's call the lifetime of this reference `'1`
  |                       |
  |                       let's call the lifetime of this reference `'2`
3 |     vec.push(item);
  |     ^^^^^^^^^^^^^^ argument requires that `'1` must outlive `'2`
```

- Meaning that compiler mostly looks at the `fn` signatures to determine potential lifetime issues.

</br>

If we never pushed our reference to the Vec, than error would be gone.

- Even if I pass a reference `my_str` to `my_push`, but the function doesn't actually use it, the compiler is smart enough to see that and won't give me an error when it's dropped later.

```rust
fn push_vec(vec: &mut Vec<&str>, item: &str){
    //vec.push(item); uncommenting this will throw 'lifetime' error again
}
fn main(){
    let my_str :String = "ananas is blue".to_string();
    let mut vec = Vec::<&str>::new();
    push_vec(&mut vec, &my_str);
    drop(my_str);
    println!("{:?}",vec);
}
```

</br>

All this is saying that Rust doesn't see the `Vex<&str>` items and item:`&str` as the same types...</br>
Hinting to use that **lifetimes are actually the part of the TYPE SYSTEM of Rust**. </br>

If we signal to Rust compiler that these types are actually the same type by replacing `&str` with generic `<T>`.</br>

The Compiler is happy! No more errors.

```rust
fn push_vec<T>(vec: &mut Vec<T>, item: T){
    vec.push(item);
}

fn main(){
    let my_str :String = "ananas is blue".to_string();
    let mut vec = Vec::<&str>::new();
    push_vec(&mut vec, &my_str);
    //drop(my_str);
    println!("{:?}",vec);
}
```

**Hint** if you checked `HIR code` outputed by the compiler at this point you would see this.</br>
Showing that compiler did indeed only elide (assume) lifetime of `Vec` itself. It noticed that inner Vec type and `element` param are of the same Type `<T>`.

> Compiler emmited HIR code

```txt
fn push_vec<T, '_>(vec: &'_ mut Vec<T>, item: T) { vec.push(item); }
```

Onto more experiments...

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

### Final Verion

We don't have any requirements on lifetime 'c , so we can just delete it an let compiler emit this for us.</br>
We only needed to name one lifetime `'a` param.

```rust
fn my_push<'a>(vector: & mut Vec<&'a str>, element: &'a str) {
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

- This will thow : `cannot borrow 'my_vec' as immutable because it is also borrowed as mutable`
- Because. This breaks NO MUTABLE ALIASING RULE.

</br>

---

### Lowering Rust code to HIR / MIR

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

[Rust book](https://doc.rust-lang.org/book/ch10-03-lifetime-syntax.html)

[Rustonomicon - Lifetimes](https://doc.rust-lang.org/nomicon/lifetimes.html)

[MIT - lifetimes in structs, impl blocks, 'static](https://web.mit.edu/rust-lang_v1.25/arch/amd64_ubuntu1404/share/doc/rust/html/book/first-edition/lifetimes.html)

[Rust by example](https://doc.rust-lang.org/rust-by-example/scope/lifetime.html)

</br>

by `Dominik Polzer`
