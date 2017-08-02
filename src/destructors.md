# Destructors

What the language *does* provide is full-blown automatic destructors through the
`Drop` trait, which provides the following method:

```rust,ignore
fn drop(&mut self);
```

This method gives the type time to somehow finish what it was doing.

**After `drop` is run, Rust will recursively try to drop all of the fields
of `self`.**

This is a convenience feature so that you don't have to write "destructor
boilerplate" to drop children. If a struct has no special logic for being
dropped other than dropping its children, then it means `Drop` doesn't need to
be implemented at all!

If this behaviour is unacceptable, it can be supressed by placing each field
you don't want to drop in a `union`. The standard library provides the
[`mem::ManuallyDrop`][ManuallyDrop] wrapper type as a convience for doing this.



Consider a custom implementation of `Box`, which might write `Drop` like this:

```rust
#![feature(unique, allocator_api)]

use std::heap::{Heap, Alloc, Layout};
use std::mem;
use std::ptr::drop_in_place;

struct Box<T>{ ptr: *mut T }

impl<T> Drop for Box<T> {
    fn drop(&mut self) {
        unsafe {
            drop_in_place(self.ptr);
            Heap.dealloc(self.ptr as *mut u8, Layout::new::<T>())
        }
    }
}
# fn main() {}
```

This works fine because when Rust goes to drop the `ptr` field it just sees
a `*mut T` that has no actual `Drop` implementation. Similarly nothing can
use-after-free the `ptr` because when drop exits, it becomes inaccessible.

However this wouldn't work:

```rust
#![feature(allocator_api, unique)]

use std::heap::{Heap, Alloc, Layout};
use std::ptr::drop_in_place;
use std::mem;

struct Box<T>{ ptr: *mut T }

impl<T> Drop for Box<T> {
    fn drop(&mut self) {
        unsafe {
            drop_in_place(self.ptr);
            Heap.dealloc(self.ptr as *mut u8, Layout::new::<T>());
        }
    }
}

struct SuperBox<T> { my_box: Box<T> }

impl<T> Drop for SuperBox<T> {
    fn drop(&mut self) {
        unsafe {
            // """Hyper-optimized""": deallocate the box's contents for it
            // without `drop`ing the contents
            Heap.dealloc(self.my_box.ptr as *mut u8, Layout::new::<T>());
        }
    }
}
# fn main() {}
```

After we deallocate `my_box`'s ptr in SuperBox's destructor, Rust will
happily proceed to tell `my_box` to Drop itself and everything will blow up with
use-after-frees and double-frees.

Note that the recursive drop behavior applies to all structs and enums
regardless of whether they implement Drop. Therefore something like

```rust
struct Boxy<T> {
    data1: Box<T>,
    data2: Box<T>,
    info: u32,
}
```

will have its `data1` and `data2` fields' destructors run whenever it "would" be
dropped, even though it itself doesn't implement Drop. We say that such a type
*needs Drop*, even though it is not itself Drop. This property can be checked
for with the [`mem::needs_drop()`][needs_drop] function.

Similarly,

```rust
enum Link {
    Next(Box<Link>),
    None,
}
```

will have its inner Box field dropped if and only if an instance stores the
Next variant.

In general this works really nicely because you don't need to worry about
adding/removing drops when you refactor your data layout. But there's
certainly many valid usecases for needing to do trickier things with
destructors.

The classic safe solution to preventing recursive drop and allowing moving out
of Self during `drop` is to use an Option:

```rust
#![feature(allocator_api, unique)]

use std::heap::{Alloc, Heap, Layout};
use std::ptr::drop_in_place;
use std::mem;

struct Box<T>{ ptr: *mut T }

impl<T> Drop for Box<T> {
    fn drop(&mut self) {
        unsafe {
            drop_in_place(self.ptr);
            Heap.dealloc(self.ptr as *mut u8, Layout::new::<T>());
        }
    }
}

struct SuperBox<T> { my_box: Option<Box<T>> }

impl<T> Drop for SuperBox<T> {
    fn drop(&mut self) {
        unsafe {
            // Hyper-optimized: deallocate the box's contents for it
            // without `drop`ing the contents. Need to set the `box`
            // field as `None` to prevent Rust from trying to Drop it.
            let my_box = self.my_box.take().unwrap();
            Heap.dealloc(my_box.ptr as *mut u8, Layout::new::<T>());
            mem::forget(my_box);
        }
    }
}
# fn main() {}
```

However this has fairly odd semantics: you're saying that a field that *should*
always be Some *may* be None, just because that happens in the destructor. Of
course this conversely makes a lot of sense: you can call arbitrary methods on
self during the destructor, and this should prevent you from ever doing so after
deinitializing the field. Not that it will prevent you from producing any other
arbitrarily invalid state in there.

On balance this is an ok choice. Certainly what you should reach for by default.

Should using Option be unacceptable, [`ManuallyDrop`][ManuallyDrop] is always
available.


[ManuallyDrop]: https://doc.rust-lang.org/std/mem/union.ManuallyDrop.html
[needs_drop]: https://doc.rust-lang.org/nightly/std/mem/fn.needs_drop.html
