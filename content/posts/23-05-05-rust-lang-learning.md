---
title: "Learning Rust"
date: 2023-05-05T15:05:55-04:00
draft: false
tags: ["rust"]
---

### Basics

- Package, crate, and module: package is a collection of crates. crate is a binary or library. module is a namespace (to manage scope. mod can be nested).

```rust
pub use crate::sound::instrument; // export to external crate
fn example() {
    //...
}
mod sound {
    pub mod instrument {
        pub fn clarinet() { // public function 
            //...
        }
    }

    fn guitar() { // private function
        super::example(); // call parent module
    }
}

pub fn main() {
    // Absolute path
    crate::sound::instrument::clarinet();

    // Relative path
    sound::instrument::clarinet();
}
```

- cargo v.s. `rustc`: run with `cargo build --release | run`. use `cargo update` to update dependencies.

- Simple mutable reference.

```rust
// Example: const (cannot be mutable)
const MAX_POINTS: u32 = 100_000;

let mut s = String::from("hello");
io::stdin().read_line(&mut s).unwrap();

// type conversion (with shadowing)
loop {
    let s: u32 = match s.trim().parse() {
        Ok(num) => num,
        Err(_) => continue,
    };
}   
```

- Tuple and array: and array is fixed length (use vector if you want to change length).

```rust
// continuously placed on stack
let a: [i32; 5] = [1, 2, 3, 4, 5];
```

- Statements and expressions: 

```rust
let y = {
    let x = 3;
    x + 1 // x+1; makes it a statement
    break x+1; // break and return expression
};
```

- Ownership and scope: to better manage the heap memory by tracing at compile time, reducing the waste. Instead of `move`, you can also use `clone` to copy the heap data (i.e., deep copy).

```rust
{
    // s' actual content allocated on heap, pointer (meta date of s) on stack
    let mut s = String::from("hello"); 
    s.push_str(", world!");  
    
    // move from s to s2: meta data on stack copied, s is invalidated
    // this is to prevent double free error
    let s2 = s; 

} // s2 is out of scope, drop() is called here to free buffer


{
    // Copy trait used in assignment: copied on stack
    let y = 5
    let x = y
    makes_copy(y)
    println!("{}", y) // valid
}
```

- Reference: use value without taking ownership. Borrowing: reference as function parameter (can be mutable or immutable).

```rust
let mut s1 = String::from("hello");
let len = calculate_length(&s1); // &s1: reference to s1 (i.e., pointer to meta data)

fn calculate_length(s: &mut String) -> usize { // s: reference to String
    s.push_str(", world");
    s.len()
} // s goes out of scope, but nothing happens

let mut s = String::from("hello");
let r1 = &mut s;
{
    let r2 = &mut s; // valid. not in the same scope as r1
}
```

- Slice: reference to part of a String (basically substring) or array

```rust
let s = String::from("hello world");
let hello = &s[0..5]; // or &s[..5]
```

### Structures
- Struct: similar to C struct, but can have methods.

```rust
#[derive(Debug)] // to print struct
struct Rectangle {
    width: u32,
    height: u32,
}

impl Rectangle { // can be multiple impl blocks
    fn area(&self) -> u32 { // not taking ownership
        self.width * self.height
    }
}

println!("rect is {:#?}", rect); // print struct

```

- Enum: similar to C enum, but can have data with the enum (i.e., enum variants, like IPv4, which is not a specific string value).

```rust
enum IpAddr {
    V4(u8, u8, u8, u8),
    V6(String),
    Other({ x: u32, y: u32 })
}

let v4 = IpAddr::V4(127, 0, 0, 1);
```

- Option: to avoid null pointer exception. 

```rust
// defined in prelude with custom typed enum
enum Option<T> {
    Some(T),
    None,
}

let some_number = Some(5); // Option<i32>
let some_string = Some("a string"); // Option<&str>
let absent_number: Option<i32> = None;
```
 

 - Single case pattern matching `if let`

```rust
let some_u8_value = Some(0u8);
match some_u8_value {
    Some(3) => println!("three"),
    _ => (),
}

// equivalent to
if let Some(3) = some_u8_value {
    println!("three");
} else {
    ()
}
```

### Heap data structure
- Vector: 

```rust
let mut v: Vec<i32> = Vec::new();
let v = vec![1, 2, 3]; // type inference

// access vector values
let third: &i32 = &v[2]; // panic if out of bound
let third: Option<&i32> = v.get(2); // return None if out of bound

// iterate over vector
for i in &mut v {
    // println!("{}", i);
    *i += 1; 
}
```

- HashMap: the values are either copied or moved (i.e., ownership is transferred), depending on the type of values. Or passing reference to the value also works.

```rust 
use std::collections::HashMap;
let mut scores = HashMap::new();

// insert (overwrite if key exists)
scores.insert(String::from("Blue"), 10);
let v: &mut i32 = scores.entry(String::from("Yellow")).or_insert(50); // insert if key does not exist

// use zip to create hashmap
let scores: HashMap<_, _> = keys.iter().zip(values.iter()).collect();
```

### Exception handling
- `panic!` macro: to abort the program when something goes wrong, or unwinding call stack (i.e., backtrace and clean up the stack). set `[profile.release] panic = 'abort'`

- recoverable error: `Result<T, E>` enum. use `match` to handle the returned result enum or `unwrap_or_else` to panic. Or a simple `unwrap()` over the result to either return or panic.

```rust
let f = File::open("hello.txt");

// same as File::open("hello.txt").unwrap() or expect()
let f = match f {
    Ok(file) => file,
    Err(error) => panic!("Problem opening the file: {:?}", error),
};

// propagate error. stop and return error if any error occurs
let f = File::open("hello.txt")?; // implicitly call `from` to convert error type
Ok(())


// chain methods. the parent function must return Result type
File::open("hello.txt")?.read_to_string(&mut s)?;
```

### Advanced
- Trait: similar to interface in Java. Trait bounds: to restrict the generic type to implement certain traits. 

```rust
pub trait Summary {
    fn summarize(&self) -> String; // method signature
}

// implement trait for a struct
// trait needs to be in scope when being used
impl Summary for NewsArticle {
    fn summarize(&self) -> String {
        format!("{}, by {} ({})", self.headline, self.author, self.location)
    }
}

// syntax sugar for trait bounds
pub fn notify(item: &impl Summary) {
    println!("Breaking news! {}", item.summarize());
}

// trait bounds
pub fn notify<T: Summary>(item: &T) {
    println!("Breaking news! {}", item.summarize());
}

// blanket implementation: implement trait for any type that implements another trait
impl<T: Display> ToString for T {
    // --snip--
}
```

- Life cycle: to prevent dangling reference. Lifetime annotations: to annotate the lifetime of references in function signatures. Lifetime elision: to omit lifetime annotations in common cases.

```rust
// borrow checker: to prevent dangling reference

// lifetime annotations: does not change the lifetime of any value
// only used to annotate the relationship of multiple references
fn longest<'a>(x: &'a str, y: &'a str) -> &'a str {
    if x.len() > y.len() {
        x
    } else {
        y
    }
}
```