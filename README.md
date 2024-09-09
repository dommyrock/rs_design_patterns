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
//we specify that the returns_summarizable function returns some type that implements the Summary trait without naming the concrete type
```
However, you can only use impl Trait if you’re **returning a single type**.<br>
Returning either a NewsArticle or a Tweet isn’t allowed due to restrictions around how the impl Trait syntax is implemented in the compiler.<br>
See : [Using Trait Objects That Allow for Values of Different Types](https://doc.rust-lang.org/book/ch17-02-trait-objects.html#using-trait-objects-that-allow-for-values-of-different-types)



---
# TODO:
### Static dispatch

### Dynamic dispatch 


---
### References 
- [Advanced Traits (Book)](https://doc.rust-lang.org/book/ch19-03-advanced-traits.html)
- [Catalog of more Rust design patterns](https://rust-unofficial.github.io/patterns/intro.html)
- ['Deref' Trait](https://doc.rust-lang.org/book/ch15-02-deref.html#treating-smart-pointers-like-regular-references-with-the-deref-trait)
  <br>
##### Other
- [Typestate Pattern in Rust](https://cliffle.com/blog/rust-typestate/)
- [Extension Traits in Rust](http://xion.io/post/code/rust-extension-traits.html)

$${\color{orange}By| Dominik Polzer}$$
