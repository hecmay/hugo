---
title: "Learning Rust"
date: 2023-05-05T15:05:55-04:00
draft: false
tags: ["rust"]
---

### Basics
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