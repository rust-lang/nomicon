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

And indeed this would compile. Unfortunately, it would be too strict. The
compiler will give us too strict variance. So a `&Vec<&'static str>`
couldn't be used where a `&Vec<&'a str>` was expected. See [the chapter
on ownership and lifetimes][ownership] for all the details on variance.

As we saw in the ownership chapter, when the standard library has a raw pointer
to an allocation that it owns, it wants that pointer to declare more than a bare
`*mut T` does:

* We are covariant over `T`
* We may own a value of type `T` (this is not relevant for our example here, but see 
  [the chapter on PhantomData][phantom-data] on why the real `std::vec::Vec<T>` needs this)
* We are Send/Sync if `T` is Send/Sync
* Our pointer is never null (so `Option<Vec<T>>` is null-pointer-optimized)

We can implement all of the above requirements in stable Rust. To do this, we will
use [`NonNull<T>`][NonNull], a wrapper around a raw pointer which gives us two of
the above properties, namely it is covariant over `T` and is declared to never be
null. By implementing Send/Sync if `T` is, we get the rest of what we need:

```rust
use std::ptr::NonNull;

pub struct Vec<T> {
    ptr: NonNull<T>,
    cap: usize,
    len: usize,
}

unsafe impl<T: Send> Send for Vec<T> {}
unsafe impl<T: Sync> Sync for Vec<T> {}
# fn main() {}
```

[ownership]: ../ownership.html
[phantom-data]: ../phantom-data.md
[NonNull]: ../../std/ptr/struct.NonNull.html
