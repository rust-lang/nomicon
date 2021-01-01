# Layout

Let's start by making the layout for our implementation of `Arc`.

We'll need at least two things: a pointer to our data and a shared atomic
reference count. Since we're *building* `Arc`, we can't store the reference
count inside another `Arc`. Instead, we can get the best of both worlds and
store our data and the reference count in one structure and get a pointer to
that. This also prevents having to dereference two separate pointers inside our
code, which may also improve performance. (Yay!)

Naively, it'd looks something like this:
```rust,ignore
use std::sync::atomic;

pub struct Arc<T> {
    ptr: *mut ArcInner<T>,
}

pub struct ArcInner<T> {
    rc: atomic::AtomicUsize,
    data: T
}
```

This would compile, however it would be incorrect. First of all, the compiler
will give us too strict variance. For example, an `Arc<&'static str>` couldn't
be used where an `Arc<&'a str>` was expected. More importantly, it will give
incorrect ownership information to the drop checker, as it will assume we don't
own any values of type `T`. As this is a structure providing shared ownership of
a value, at some point there will be an instance of this structure that entirely
owns its data. See [the chapter on ownership and lifetimes](ownership.md) for
all the details on variance and drop check.

To fix the first problem, we can use `NonNull<T>`. Note that `NonNull<T>` is a
wrapper around a raw pointer that declares that:
* We are variant over `T`
* Our pointer is never null

To fix the second problem, we can include a `PhantomData` marker containing an
`ArcInner<T>`. This will tell the drop checker that we have some notion of
ownership of a value of `ArcInner<T>` (which itself contains some `T`).

With these changes, our new structure will look like this:
```rust,ignore
use std::marker::PhantomData;
use std::ptr::NonNull;
use std::sync::atomic::AtomicUsize;

pub struct Arc<T> {
    ptr: NonNull<ArcInner<T>>,
    phantom: PhantomData<ArcInner<T>>
}

pub struct ArcInner<T> {
    rc: AtomicUsize,
    data: T
}
```
