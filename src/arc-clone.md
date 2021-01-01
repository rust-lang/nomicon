# Cloning

Now that we've got some basic code set up, we'll need a way to clone the `Arc`.

Basically, we need to:
1. Get the `ArcInner` value of the `Arc`
2. Increment the atomic reference count
3. Construct a new instance of the `Arc` from the inner pointer

Next, we can update the atomic reference count as follows:
```rust,ignore
self.inner().rc.fetch_add(1, Ordering::Relaxed);
```

As described in [the standard library's implementation of `Arc` cloning][2]:
> Using a relaxed ordering is alright here, as knowledge of the original
> reference prevents other threads from erroneously deleting the object.
> 
> As explained in the [Boost documentation][1]:
> > Increasing the reference counter can always be done with
> > memory_order_relaxed: New references to an object can only be formed from an
> > existing reference, and passing an existing reference from one thread to
> > another must already provide any required synchronization.
> 
> [1]: https://www.boost.org/doc/libs/1_55_0/doc/html/atomic/usage_examples.html
[2]: https://github.com/rust-lang/rust/blob/e1884a8e3c3e813aada8254edfa120e85bf5ffca/library/alloc/src/sync.rs#L1171-L1181

We'll need to add another import to use `Ordering`:
```rust,ignore
use std::sync::atomic::Ordering;
```

It is possible that in some contrived programs (e.g. using `mem::forget`) that
the reference count could overflow, but it's unreasonable that would happen in
any reasonable program.

Then, we need to return a new instance of the `Arc`:
```rust,ignore
Self {
    ptr: self.ptr,
    _marker: PhantomData
}
```

Now, let's wrap this all up inside the `Clone` implementation:
```rust,ignore
use std::sync::atomic::Ordering;

impl<T> Clone for Arc<T> {
    fn clone(&self) -> Arc<T> {
        // Using a relaxed ordering is alright here as knowledge of the original
        // reference prevents other threads from wrongly deleting the object.
        self.inner().rc.fetch_add(1, Ordering::Relaxed);
        Self {
            ptr: self.ptr,
            _marker: PhantomData,
        }
    }
}
```
