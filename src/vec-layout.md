# Layout

First off, we need to come up with the struct layout. A Vec has three parts:
a pointer to the allocation, the size of the allocation, and the number of
elements that have been initialized.

Naively, this means we just want this design:

```rust
pub struct Vec<T> {
    ptr: *mut T,
    cap: usize,
    len: usize,
}
# fn main() {}
```

And indeed this would compile and work correctly. However it comes with a semantic
limitation and a missed optimization opportunity.

In terms of semantics, this implementation of Vec would be [invariant over T][variance].
So a `&Vec<&'static str>` couldn't be used where an `&Vec<&'a str>` was expected.

In terms of optimization, this implementation of Vec wouldn't be eligible for the
*null pointer optimization*, meaning `Option<Vec<T>>` would take up more space
than `Vec<T>`.

These are fairly common problems because the raw pointer types in Rust aren't
very well optimized for this use-case. They're more tuned to make it easier to
express C APIs. This is why the standard library provides a pointer type that
better matches the semantics pure-Rust abstractions want: `Shared<T>`.

Compared to `*mut T`, `Shared<T>` provides three benefits:

* Variant over `T` (dangerous in general, but desirable for collections)
* Null-pointer optimizes (so `Option<Shared<T>>` is pointer-sized)

We could get the variance requirement ourselves using `*const T` and casts, but
the API for expressing a value is non-zero is unstable, and that isn't expected
to change any time soon.

Shared should be stabilized in some form very soon, so we're just going to use
that.


```rust
#![feature(shared)]

use std::ptr::Shared;

pub struct Vec<T> {
    ptr: Shared<T>,
    cap: usize,
    len: usize,
}

# fn main() {}
```

If you don't care about the null-pointer optimization, then you can use `*const T`.
For most code, using `*mut T` would also be perfectly reasonable.
However this chapter is focused on providing an implementation that matches the
quality of the one in the standard library, so we will be designing the rest of
the code around using Shared.

Lastly, it should be noted that `Shared::new` is unsafe to call, because
putting `null` inside of it is Undefined Behavior. Code that doesn't use Shared
has no such concern.

[variance]: variance.html
