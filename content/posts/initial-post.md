---
title: "Design by contract in Rust"
date: 2023-07-23T15:58:19+01:00
draft: true
---

No, no blockchains here.

There's a [contracts crate](https://docs.rs/contracts/latest/contracts/) in Rust, as in [Design by Contract](https://en.wikipedia.org/wiki/Design_by_contract), but I figured I'd try to write my own, but I'm going on a tangent first.

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
fn robust_f(thing: i32) -> i32 {
    // Some checks here 
    unsafe { reified_transmogrify(thing) }
}

unsafe fn basic_f(p: i32) -> i32 {
    // your implementation here 
} 
```

The big cousin has a clear reason to exist: it prevent unnecessary recompilation of the full `reified_transmogrify` function - though `transmogrify` will still be recompiled.

Why the smaller cousin though? Well, the naming and the rest of the post might have given it away.

Suppose you have an unsafe performance critical function, and you want to make sure it's called with valid arguments. The contracts crate essentially would turn this:

```rust
#[requires(thing > 0)]
fn robust_f(thing: i32) -> i32 {
    // Some checks here 
    unsafe { /* implementation here */ }
}
```

Into this:

```rust
fn robust_f(thing: i32) -> i32 {
    if !(thing > 0) {
        panic!("Precondition failed: thing > 0");
    }
    unsafe { /* implementation here */ }
}
```

That is marvelous from a usability perspective, but I think our pattern has a place here. We might turn the original function into this:

```rust
#[inline(always)]
fn robust_f(thing: i32) -> i32 {
   // Some assert! here 
    let ret = unsafe { simplified_f(thing) }
    // Some assert_unchecked here
    ret
}
unsafe fn simplified_f(p: i32) -> i32 {
        // Some assert_unchecked here, from the assert-unchecked crate
        // your implementation here 
        // some assert! here (not unchecked)
}   
```

That way, all `#[requires]` and `#[ensures]` are turned into `assert!` calls at the _call site_.

That way, we get the best of both worlds:

- The `robust_f` function can be called from safe code, and the checks will be optimized away.
- The `robust_f` function can be called from unsafe code, and the checks will still be performed.
- The `robust_f` function can be called from unsafe code, and the checks might be optimized away, if the compiler can prove they are redundant. This is, admittedly, a bit of a stretch.

Is it any use? I have no idea. I guess I'll write the crate as a drop-in replacement for the contracts crate and see if it's any good in crates that use it.