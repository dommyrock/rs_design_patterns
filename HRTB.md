# HRTB - `Higher Rank Trait Bounds`

A combination of `Traits`, `Trait Bounds` and `Lifetime Annotations`

This concept Combines `Trait bounds` and `Lifetimes` so that **Trait bounds are HIGHER RANKED over Lifetimes**.

It specifies that a `Trait Bound` **should hold true for all possible lifetimes**.

```rust
for‹'a > T: Trait‹'a>
```

> Usefull when we're expressing complex lifetime relationships. </br>

## Some examples : ) (not using HRTB yet)

Lets first start by describing why would we (or the compiler) opt in for HRTB...

```rust
//we'll create a Formatter trait that takes in value of T :std::fmt::Display 
//meaning we allow any type that implements a 'Display' Trait

trait Formatter {
   fn format<T: Display>(&self, value: T) -> String;
}

struct SimpleFormatter;

//Implement our trait on struct
impl Formatter for SimpleFormatter {
   fn format<T: Display>(&self, value: T) -> String {
      format! ("Value: {}", value)
   }
}

//fn that takes in 'F' of type 'Formatter' and 'returns a Closure' by using that Formatter
fn apply_format<F>(formatter: F) -> impl Fn(&str) -> String where
   F: Formatter,
{
   move formatter.format(s)
}
```

- [std::fmt::Display Trait](https://doc.rust-lang.org/std/fmt/trait.Display.html)

</br>

Core problems is that the `Lifetime` here is **to Restrictive**.

If we were to write it like this instead. (adding a lifetime 'a to generic fn)

```rust
fn apply_format<'a, F>(formatter: F) -> impl Fn(&'a str) -> String
   where F: Formatter,
{
   move |s| formatter.format(s)
}
```

Meaning that the `reference lifetime` that is passed to our closure is **tied** to the `lifetime of the closure`.

---

### Using HRTB

What we really want is ...
Instead of defining a lifetime on the function we will use `higher rank trait bounds`.

```rust
fn apply_format<F>(formatter: F) -> impl for<'a> Fn(&'a str) -> String
//...
```

This means that closure can work with ANY lifetimes as long as it covers the lifetime of the closure execution.

</br>

In simple cases like this compiler is smart enought to infer the `HRTB` for us. So we can just write.

```rust
fn apply_format<F>(formatter: F) -> impl Fn(&'a str) -> String
where F: Formatter,
{
   move s| formatter.format(s)
}

fn main() {
   let formatter = SimpleFormatter;
   let s1 = "Hello";
   let s2 = String:: from ("World");
   let format_fn = apply_format(formatter);
   {
      let s3: String = String::from("World");
      printin!("{}", format_fn(&s3));
   }
}
```

#### One more example

In case when lifetime of the reference is shorter than any possible lifetime parameter on the function.

```rust
fn call_on_ref_zero<F>(f: F) where for<'a> F: Fn(&'a i32) {
    let zero = 0;
    f(&zero);
}
```

HRTB can also be written like this .

```rust
fn call_on_ref_zero<F>(f: F) where F: for<'a> Fn(&'a i32) {
    let zero = 0;
    f(&zero);
}
```

Higher-ranked lifetimes may also be specified just before the trait: the only difference is the [scope](https://doc.rust-lang.org/reference/names/scopes.html#higher-ranked-trait-bound-scopes) of the lifetime parameter, which extends only to the end of the following trait instead of the whole bound.

---

#### Refs

- [HRTB Intro - Video](https://www.youtube.com/watch?v=6fwDwJodJrg)
- [Rustonomicon](https://doc.rust-lang.org/nomicon/hrtb.html)
- [Hrtb Rust Reference](https://doc.rust-lang.org/reference/trait-bounds.html#higher-ranked-trait-bounds)