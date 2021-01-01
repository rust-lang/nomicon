# Base Code

Now that we've decided the layout for our implementation of `Arc`, let's create
some basic code.

## Constructing the Arc

We'll first need a way to construct an `Arc<T>`.

This is pretty simple, as we just need to box the `ArcInner<T>` and get a
`NonNull<T>` pointer to it.

We start the reference counter at 1, as that first reference is the current
pointer. As the `Arc` is cloned or dropped, it is updated. It is okay to call
`unwrap()` on the `Option` returned by `NonNull` as `Box::into_raw` guarantees
that the pointer returned is not null.

```rust,ignore
impl<T> Arc<T> {
    pub fn new(data: T) -> Arc<T> {
        // We start the reference count at 1, as that first reference is the
        // current pointer.
        let boxed = Box::new(ArcInner {
            rc: AtomicUsize::new(1),
            data,
        });
        Arc {
            // It is okay to call `.unwrap()` here as we get a pointer from
            // `Box::into_raw` which is guaranteed to not be null.
            ptr: NonNull::new(Box::into_raw(boxed)).unwrap(),
            _marker: PhantomData,
        }
    }
}
```

## Send and Sync

Since we're building a concurrency primitive, we'll need to be able to send it
across threads. Thus, we can implement the `Send` and `Sync` marker traits. For
more information on these, see [the section on `Send` and
`Sync`](send-and-sync.md).

This is okay because:
* You can only get a mutable reference to the value inside an `Arc` if and only
  if it is the only `Arc` referencing that data
* We use atomic counters for reference counting

```rust,ignore
unsafe impl<T: Sync + Send> Send for Arc<T> {}
unsafe impl<T: Sync + Send> Sync for Arc<T> {}
```

We need to have the bound `T: Sync + Send` because if we did not provide those
bounds, it would be possible to share values that are thread-unsafe across a
thread boundary via an `Arc`, which could possibly cause data races or
unsoundness.

## Getting the `ArcInner`

We'll now want to make a private helper function, `inner()`, which just returns
the dereferenced `NonNull` pointer.

To dereference the `NonNull<T>` pointer into a `&T`, we can call
`NonNull::as_ref`. This is unsafe, unlike the typical `as_ref` function, so we
must call it like this:
```rust,ignore
// inside the impl<T> Arc<T> block from before:
fn inner(&self) -> &ArcInner<T> {
    unsafe { self.ptr.as_ref() }
}
```

This unsafety is okay because while this `Arc` is alive, we're guaranteed that
the inner pointer is valid.

Here's all the code from this section:
```rust,ignore
impl<T> Arc<T> {
    pub fn new(data: T) -> Arc<T> {
        // We start the reference count at 1, as that first reference is the
        // current pointer.
        let boxed = Box::new(ArcInner {
            rc: AtomicUsize::new(1),
            data,
        });
        Arc {
            // It is okay to call `.unwrap()` here as we get a pointer from
            // `Box::into_raw` which is guaranteed to not be null.
            ptr: NonNull::new(Box::into_raw(boxed)).unwrap(),
            _marker: PhantomData,
        }
    }

    fn inner(&self) -> &ArcInner<T> {
        // This unsafety is okay because while this Arc is alive, we're
        // guaranteed that the inner pointer is valid.
        unsafe { self.ptr.as_ref() }
    }
}

unsafe impl<T: Sync + Send> Send for Arc<T> {}
unsafe impl<T: Sync + Send> Sync for Arc<T> {}
```
