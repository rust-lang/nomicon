
# RawVec

We've actually reached an interesting situation here: we've duplicated the logic
for specifying a buffer and freeing its memory in Vec and IntoIter. Now that
we've implemented it and identified *actual* logic duplication, this is a good
time to perform some logic compression.

We're going to abstract out the `(ptr, cap)` pair and give them the logic for
allocating, growing, and freeing:

```rust,ignore
struct RawVec<T> {
    ptr: Unique<T>,
    cap: usize,
}

impl<T> RawVec<T> {
    fn new() -> Self {
        assert!(mem::size_of::<T>() != 0, "TODO: implement ZST support");
        Self {
            ptr: Unique::empty(),
            cap: 0,
        }
    }

    // unchanged from Vec
    fn grow(&mut self) {
        // this is all pretty delicate, so let's say it's all unsafe
        unsafe {
            let layout = alloc::Layout::new::<T>();

            let (new_cap, ptr) = if self.cap == 0 {
                let ptr = alloc::alloc(layout);
                (1, ptr)
            } else {
                let new_cap = self.cap * 2;
                let ptr = alloc::realloc(
                    self.ptr.as_ptr() as *mut u8,
                    layout,
                    new_cap * mem::size_of::<T>(),
                );
                (new_cap, ptr)
            };

            // If allocate or reallocate fail, we'll get `null` back
            if ptr.is_null() {
                alloc::handle_alloc_error(layout);
            }

            self.ptr = Unique::new_unchecked(ptr as *mut T);
            self.cap = new_cap;
        }
    }
}

impl<T> Drop for RawVec<T> {
    fn drop(&mut self) {
        if self.cap != 0 {
            unsafe {
                let align = mem::align_of::<T>();
                let elem_size = mem::size_of::<T>();
                let num_bytes = elem_size * self.cap;

                alloc::dealloc(
                    self.buf.as_ptr() as *mut u8,
                    alloc::Layout::from_size_align_unchecked(num_bytes, align),
                );
            }
        }
    }
}
```

And change Vec as follows:

```rust,ignore
pub struct Vec<T> {
    buf: RawVec<T>,
    len: usize,
}

impl<T> Vec<T> {
    fn ptr(&self) -> *mut T { self.buf.ptr.as_ptr() }

    fn cap(&self) -> usize { self.buf.cap }

    pub fn new() -> Self {
        Vec { buf: RawVec::new(), len: 0 }
    }

    // push/pop/insert/remove largely unchanged:
    // * `self.ptr -> self.ptr()`
    // * `self.cap -> self.cap()`
    // * `self.grow -> self.buf.grow()`
}

impl<T> Drop for Vec<T> {
    fn drop(&mut self) {
        while let Some(_) = self.pop() {}
        // deallocation is handled by RawVec
    }
}
```

And finally we can really simplify IntoIter:

```rust,ignore
struct IntoIter<T> {
    _buf: RawVec<T>, // we don't actually care about this. Just need it to live.
    start: *const T,
    end: *const T,
}

// next and next_back literally unchanged since they never referred to the buf

impl<T> Drop for IntoIter<T> {
    fn drop(&mut self) {
        // only need to ensure all our elements are read;
        // buffer will clean itself up afterwards.
        for _ in &mut *self {}
    }
}

impl<T> Vec<T> {
    pub fn into_iter(self) -> IntoIter<T> {
        unsafe {
            // need to use `ptr::read` to unsafely move the buf out since it's
            // not Copy, and Vec implements Drop (so we can't destructure it).
            let buf = ptr::read(&self.buf);
            let len = self.len;
            mem::forget(self);

            IntoIter {
                start: buf.ptr.as_ptr(),
                end: if buf.cap == 0 {
                    // can't offset off this pointer, it's not allocated!
                    buf.ptr.as_ptr()
                } else {
                    buf.ptr.as_ptr().offset(len as isize)
                },
                _buf: buf,
            }
        }
    }
}
```

Much better.
