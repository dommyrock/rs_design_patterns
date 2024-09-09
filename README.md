# Rust Design patterns 

This document focuses on the common design patterns around **Traits**,**Static** and **Dynamic** dispatch.

### Initial Notes: 
> Since Rust doesn't allow inheritance.<br>
> You have to use composition instead of inheritance.<br>
> And when you need polymorphism, you use a trait. <br>

Implementing a trait on a type is similar to implementing regular methods.
```rust
pub trait Summary {
    fn summarize(&self) -> String;
}

pub struct Tweet {
    pub username: String,
    pub content: String,
}

impl Summary for Tweet {
    fn summarize(&self) -> String {
        format!("{}: {}", self.username, self.content)
    }
}
```
Sometimes it’s useful to have default behavior for some or all of the methods in a trait instead of requiring implementations for all methods on every type. 
```rust
pub trait Summary {
    fn summarize_author(&self) -> String;

    fn summarize(&self) -> String {
        format!("(Read more from {}...)", self.summarize_author())
    }
}
```
To use this version of Summary, we only need to define summarize_author when we implement the trait on a type:
```rust
impl Summary for Tweet {
    fn summarize_author(&self) -> String {
        format!("@{}", self.username)
    }
}
```

### Traits as Parameters
This parameter accepts any type that implements the specified trait.
```rust
pub fn notify(item: &impl Summary) {
    println!("Breaking news! {}", item.summarize());
}
```

### Trait Bound Syntax

The impl Trait syntax works for straightforward cases but is **actually syntax sugar** for a longer form known as a **trait bound**; it looks like this:
```rust
pub fn notify<T: Summary>(item: &T) {
    println!("Breaking news! {}", item.summarize());
}
```
This longer form is equivalent to the example in the previous section but is more verbose.
The **impl Trait** syntax is convenient and makes for **more concise code in simple cases**.<br>
While the fuller **trait bound syntax** can express more complexity in other cases. 

For example, we can have two parameters that implement Summary. Doing so with the impl Trait syntax looks like this:
```rust,ignore
pub fn notify(item1: &impl Summary, item2: &impl Summary) { ...

//Using impl Trait is appropriate if we want this function to allow item1 and item2 to have different types (as long as both types implement Summary).
//If we want to force both parameters to have the same type, however, we must use a trait bound, like this:
pub fn notify<T: Summary>(item1: &T, item2: &T) { ...
```

### Specifying Multiple Trait Bounds with the + Syntax
```rust,ignore
pub fn notify(item: &(impl Summary + Display)) { ...

pub fn notify(item: &(impl Summary + Display)) { ...
//With the two trait bounds specified, the body of notify can call summarize and use {} to format item.
```

### Clearer Trait Bounds with where Clauses
Cleaner syntax when you have a lot of trait bounds.<br>
Rust has alternate syntax for specifying trait bounds inside a where clause after the function signature. So, instead of writing this:
```rust,ignore
fn some_function<T: Display + Clone, U: Clone + Debug>(t: &T, u: &U) -> i32 { ...

//we can use a where clause, like this:
fn some_function<T, U>(t: &T, u: &U) -> i32
where
    T: Display + Clone,
    U: Clone + Debug,
{
```

Returning Types That Implement Traits
```rust
//We can also use the impl Trait syntax in the return position to return a value of some type that implements a trait, as shown here:
fn returns_summarizable() -> impl Summary {
    Tweet {
        username: String::from("horse_ebooks"),
        content: String::from(
            "of course, as you probably already know, people",
        ),
    }
}
//returns_summarizable function returns some type that implements the Summary trait without naming the concrete type.
```
However, you can only use impl Trait if you’re **returning a single type**.<br>
Returning either a NewsArticle or a Tweet isn’t allowed due to restrictions around how the impl Trait syntax is implemented in the compiler.<br>
See : [Using Trait Objects That Allow for Values of Different Types](https://doc.rust-lang.org/book/ch17-02-trait-objects.html#using-trait-objects-that-allow-for-values-of-different-types)

### Using Trait Bounds to Conditionally Implement Methods
By using a trait bound with an impl block that uses generic type parameters, we can implement methods conditionally for types that implement the specified traits.

```rust
//in the next impl block, Pair<T> only implements the cmp_display method if its inner type T
//1 implements the PartialOrd trait that enables comparison
//2 and the Display trait that enables printing.
struct Pair<T> {
    x: T,
    y: T,
}

impl<T> Pair<T> {
    fn new(x: T, y: T) -> Self {
        Self { x, y }
    }
}

impl<T: std::fmt::Display + PartialOrd> Pair<T> {
    fn cmp_display(&self) {
        if self.x >= self.y {
            println!("The largest member is x = {}", self.x);
        } else {
            println!("The largest member is y = {}", self.y);
        }
    }
}
```
We can also conditionally implement a trait for any type that implements another trait. 
```rust,ignore
impl<T: Display> ToString for T {
    // --snip--
}
//Because the standard library has this blanket implementation, we can call the to_string method
//defined by the ToString trait on any type that implements the Display trait.

//For example, we can turn integers into their corresponding String values like this because integers implement Display
let s = 3.to_string();
```
Blanket implementations appear in the documentation for the trait in the “Implementors” section.


### Fn Traits | [Book](https://doc.rust-lang.org/book/ch13-01-closures.html#moving-captured-values-out-of-closures-and-the-fn-traits)
```rust
//FnOnce - All closures implement at least this trait
//A closure that moves captured values out of its body will only implement FnOnce and none of the other Fn traits,
//because it can only be called once.
impl<T> Option<T> {
    pub fn unwrap_or_else<F>(self, f: F) -> T
    where
        F: FnOnce() -> T
    {
        match self {
            Some(x) => x,
            None => f(),
        }
    }
}
//The trait bound specified on the generic type F is FnOnce() -> T,
//which means F must be able to be called once, take no arguments, and return a T.
```
Using $${\color{orange}FnOnce}$$ in the trait bound expresses the constraint that unwrap_or_else is only going to call f at most one time.
Because all closures implement $${\color{orange}FnOnce}$$, unwrap_or_else accepts all three kinds of closures and is as flexible as it can be.

### Trait objects - Dynamic & Static dispatch | [Book](https://doc.rust-lang.org/book/ch17-02-trait-objects.html#using-trait-objects-that-allow-for-values-of-different-types)
<br/>

$${\color{orange}Monomorphization}$$ process performed by the compiler when we use trait bounds on generics:<br>
the compiler generates nongeneric implementations of functions and methods for each concrete type that we use in place of a generic type parameter.<br>
The code that results from monomorphization is doing **static dispatch**, which is when the compiler knows what method you’re calling at compile time.<br>
This is opposed to **dynamic dispatch**, which is when the compiler can’t tell at compile time which method you’re calling.<br>
In dynamic dispatch cases, the compiler emits code that at runtime will figure out which method to call.
<br/>
When we use trait objects, Rust must use dynamic dispatch.




---
### References 
##### Book
- [Advanced Traits](https://doc.rust-lang.org/book/ch19-03-advanced-traits.html)
- [Trait objects](https://doc.rust-lang.org/book/ch17-02-trait-objects.html#using-trait-objects-that-allow-for-values-of-different-types)
- [Functional features](https://doc.rust-lang.org/book/ch13-00-functional-features.html)
- ['Deref' Trait](https://doc.rust-lang.org/book/ch15-02-deref.html#treating-smart-pointers-like-regular-references-with-the-deref-trait)

##### Posts
- [Polymorphism in rust - Matt Oswalt](https://oswalt.dev/2021/06/polymorphism-in-rust/)
- [Catalog of more Rust design patterns](https://rust-unofficial.github.io/patterns/intro.html)
- [Typestate Pattern in Rust](https://cliffle.com/blog/rust-typestate/)
- [Extension Traits in Rust](http://xion.io/post/code/rust-extension-traits.html)

##### Videos: 
- [Dyn vs Static Dispatch - Ryan Levick](https://www.youtube.com/watch?v=tM2r9HD4ivQ)
- [Dispatch & Fat pointers - Jon Gjengset](https://www.youtube.com/watch?v=xcygqF5LVmM)
- [Zero cost abstractions intro - Oliver Jumpertz](https://www.youtube.com/watch?v=MRXi19zQgSo)
- 


---

$${\color{orange}By| Dominik&#8202;Polzer}$$
