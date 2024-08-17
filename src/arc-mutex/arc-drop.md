# Dropping

We now need a way to decrease the reference count and drop the data once it is
low enough, otherwise the data will live forever on the heap.

To do this, we can implement `Drop`.

Basically, we need to:

1. Decrement the reference count
2. If there is only one reference remaining to the data, then:
3. Atomically fence the data to prevent reordering of the use and deletion of
   the data
4. Drop the inner data

First, we'll need to get access to the `ArcInner`:

<!-- ignore: simplified code -->
```rust,ignore
let inner = unsafe { self.ptr.as_ref() };
```

Now, we need to decrement the reference count. To streamline our code, we can
also return if the returned value from `fetch_sub` (the value of the reference
count before decrementing it) is not equal to `1` (which happens when we are not
the last reference to the data).

<!-- ignore: simplified code -->
```rust,ignore
if inner.rc.fetch_sub(1, Ordering::Release) != 1 {
    return;
}
```

Why do we use `Release` here? Well, the `Release` ordering ensures
that any writes to the data from other threads happen-before this
decrementing of the reference count.

If we succeed, however, we need further guarantees. We must use an
`Acquire` fence, which ensures that the decrement of the reference
count happens-before our deletion of the data. 

To do this, we do the following:

<!-- ignore: simplified code -->
```rust,ignore
use std::sync::atomic;
atomic::fence(Ordering::Acquire);
```

We could have used `AcqRel` for the `fetch_sub` operation, but that
would give more guarantees than we need when we *don't* succeed. On
some platforms, using an `AcqRel` ordering for *every* `Drop` may have
an impact on performance. While this is a niche optimization, it can't
hurt--also, it helps to further convey the guarantees necessary not
only to the processor but also to readers of the code.

With the combination of these two synchronization points, we ensure
that, to our thread, the following order of events manifests:
- Data used by our/other thread
- Reference count decremented by our thread
- Data deleted by our thread
This way, we ensure that our data is not dropped while it is still
in use.

Finally, we can drop the data itself. We use `Box::from_raw` to drop the boxed
`ArcInner<T>` and its data. This takes a `*mut T` and not a `NonNull<T>`, so we
must convert using `NonNull::as_ptr`.

<!-- ignore: simplified code -->
```rust,ignore
unsafe { Box::from_raw(self.ptr.as_ptr()); }
```

This is safe as we know we have the last pointer to the `ArcInner` and that its
pointer is valid.

Now, let's wrap this all up inside the `Drop` implementation:

<!-- ignore: simplified code -->
```rust,ignore
impl<T> Drop for Arc<T> {
    fn drop(&mut self) {
        let inner = unsafe { self.ptr.as_ref() };
        if inner.rc.fetch_sub(1, Ordering::Release) != 1 {
            return;
        }
        // This fence is needed to prevent reordering of the use and deletion
        // of the data.
        atomic::fence(Ordering::Acquire);
        // This is safe as we know we have the last pointer to the `ArcInner`
        // and that its pointer is valid.
        unsafe { Box::from_raw(self.ptr.as_ptr()); }
    }
}
```
