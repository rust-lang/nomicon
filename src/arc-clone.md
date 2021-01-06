# Cloning

Now that we've got some basic code set up, we'll need a way to clone the `Arc`.

Basically, we need to:
1. Increment the atomic reference count
2. Construct a new instance of the `Arc` from the inner pointer

First, we need to get access to the `ArcInner`:
```rust,ignore
let inner = unsafe { self.ptr.as_ref() };
```

We can update the atomic reference count as follows:
```rust,ignore
let old_rc = inner.rc.fetch_add(1, Ordering::Relaxed);
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

However, we have one problem with this implementation right now. What if someone
decides to `mem::forget` a bunch of Arcs? The code we have written so far (and
will write) assumes that the reference count accurately portrays how many Arcs
are in memory, but with `mem::forget` this is false. Thus, when more and more
Arcs are cloned from this one without them being `Drop`ped and the reference
count being decremented, we can overflow! This will cause use-after-free which
is **INCREDIBLY BAD!**

To handle this, we need to check that the reference count does not go over some
arbitrary value (below `usize::MAX`, as we're storing the reference count as an
`AtomicUsize`), and do *something*.

The standard library's implementation decides to just abort the program (as it
is an incredibly unlikely case in normal code and if it happens, the program is
probably incredibly degenerate) if the reference count reaches `isize::MAX`
(about half of `usize::MAX`) on any thread, on the assumption that there are
probably not about 2 billion threads (or about **9 quintillion** on some 64-bit
machines) incrementing the reference count at once. This is what we'll do.

It's pretty simple to implement this behaviour:
```rust,ignore
if old_rc >= isize::MAX {
    std::process::abort();
}
```

Then, we need to return a new instance of the `Arc`:
```rust,ignore
Self {
    ptr: self.ptr,
    phantom: PhantomData
}
```

Now, let's wrap this all up inside the `Clone` implementation:
```rust,ignore
use std::sync::atomic::Ordering;

impl<T> Clone for Arc<T> {
    fn clone(&self) -> Arc<T> {
        let inner = unsafe { self.ptr.as_ref() };
        // Using a relaxed ordering is alright here as knowledge of the original
        // reference prevents other threads from wrongly deleting the object.
        inner.rc.fetch_add(1, Ordering::Relaxed);

        if old_rc >= isize::MAX {
            std::process::abort();
        }

        Self {
            ptr: self.ptr,
            phantom: PhantomData,
        }
    }
}
```
