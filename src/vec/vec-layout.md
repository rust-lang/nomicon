# Layout

First off, we need to come up with the struct layout. A Vec has three parts:
a pointer to the allocation, the size of the allocation, and the number of
elements that have been initialized.

Naively, this means we just want this design:

<!-- ignore: simplified code -->
```rust,ignore
pub struct Vec<T> {
    ptr: *mut T,
    cap: usize,
    len: usize,
}
```

And indeed this would compile. Unfortunately, it would be incorrect. First, the
compiler will give us too strict variance. So a `&Vec<&'static str>`
couldn't be used where an `&Vec<&'a str>` was expected. More importantly, it
will give incorrect ownership information to the drop checker, as it will
conservatively assume we don't own any values of type `T`. See [the chapter
on ownership and lifetimes][ownership] for all the details on variance and
drop check.

As we saw in the ownership chapter, the standard library uses `Unique<T>` in place of
`*mut T` when it has a raw pointer to an allocation that it owns. Unique is unstable,
so we'd like to not use it if possible, though.

As a recap, Unique is a wrapper around a raw pointer that declares that:

* We are covariant over `T`
* We may own a value of type `T` (for drop check)
* We are Send/Sync if `T` is Send/Sync
* Our pointer is never null (so `Option<Vec<T>>` is null-pointer-optimized)

We can implement all of the above requirements in stable Rust. To do this, instead
of using `Unique<T>` we will use [`NonNull<T>`][NonNull], another wrapper around a
raw pointer, which gives us two of the above properties, namely it is covariant
over `T` and is declared to never be null. By adding a `PhantomData<T>` (for drop
check) and implementing Send/Sync if `T` is, we get the same results as using
`Unique<T>`:

```rust
use std::ptr::NonNull;
use std::marker::PhantomData;

pub struct Vec<T> {
    // Invariant 1: Either cap is 0, or ptr is a Layout::array of cap T's
    // Invariant 2: from ptr onwards is `len` amount of initialized T's
    ptr: NonNull<T>,
    cap: usize,
    len: usize,
    _marker: PhantomData<T>,
}

// Safety: Semantically we own T's, so Send if T: Send, Sync if T: Sync is correct
unsafe impl<T: Send> Send for Vec<T> {}
unsafe impl<T: Sync> Sync for Vec<T> {}
# fn main() {}
```

## Safety Comments and Invariants

Before going on, it's worth explaining the comment above the impls of
`Send`/`Sync` for `Vec<T>`.

These are known as safety comments, and are good practice to get into doing if
you are writing any unsafe code. They help any reader understand why the
`unsafe` blocks are sound.

Additionally, making yourself write them encourages you to think harder about
the safety of your code, possibly leading you to discover bugs.

There is [a clippy lint][undocumented_unsafe_blocks] to warn when you forget to
do this, which you may wish to enable.

The rest of the section on implementing `Vec` will use safety comments to
explain why its use of `unsafe` is correct.

[ownership]: ../ownership.html
[NonNull]: ../../std/ptr/struct.NonNull.html
[undocumented_unsafe_blocks]: https://rust-lang.github.io/rust-clippy/master/index.html#undocumented_unsafe_blocks
