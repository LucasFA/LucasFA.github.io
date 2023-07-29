---
title: "Design by contract in Rust"
date: 2023-07-23T15:58:19+01:00
draft: true
---

No, no blockchains here.

There's a contracts crate in Rust, as in [Design by Contract](https://en.wikipedia.org/wiki/Design_by_contract), but 
One day I asked on reddit what this pattern I remembered seeing was called.

```rust
fn transmogrify<T: Into<P>>(thing: T) -> i32 {
    return reified_transmogrify(thing.into());
}

fn reified_transmogrify(p: P) -> i32 {
    // your implementation here 
} 
```

I probably found it in [this post from Matklad](https://matklad.github.io/2021/07/09/inline-in-rust.html).

It also has a smaller cousin I just made up: 

```rust
#[inline(always)]
fn robust_transmogrify(thing: i32) -> i32 {
    // Some assertions, maybe
    return reified_transmogrify(thing);
}

fn basic_transmogrify(p: i32) -> i32 {
    // your implementation here 
} 
```

The big cousin has a clear reason to exist: it prevent unnecessary recompilation of the full `reified_transmogrify` function - though `transmogrify` will still be recompiled.

Why the smaller cousin though? Well, I wrote a crate to see if it would be useful in contract programming.