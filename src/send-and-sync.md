# Send and Sync

Not everything obeys inherited mutability, though. Some types allow you to
have multiple aliases of a location in memory while mutating it. Unless these types use
synchronization to manage this access, they are absolutely not thread-safe. Rust
captures this through the `Send` and `Sync` traits.

* A type is Send if it is safe to send it to another thread.
* A type is Sync if it is safe to share between threads (T is Sync if and only if `&T` is Send).

Send and Sync are fundamental to Rust's concurrency story. As such, a
substantial amount of special tooling exists to make them work right. First and
foremost, they're [unsafe traits]. This means that they are unsafe to
implement, and other unsafe code can assume that they are correctly
implemented. Since they're *marker traits* (they have no associated items like
methods), correctly implemented simply means that they have the intrinsic
properties an implementor should have. Incorrectly implementing Send or Sync can
cause Undefined Behavior.

Send and Sync are also automatically derived traits. This means that, unlike
every other trait, if a type is composed entirely of Send or Sync types, then it
is Send or Sync. Almost all primitives are Send and Sync, and as a consequence
pretty much all types you'll ever interact with are Send and Sync.

Major exceptions include:

* raw pointers are neither Send nor Sync (because they have no safety guards).
* `UnsafeCell` isn't Sync (and therefore `Cell` and `RefCell` aren't).
* `Rc` isn't Send or Sync (because the refcount is shared and unsynchronized).

`Rc` and `UnsafeCell` are very fundamentally not thread-safe: they enable
unsynchronized shared mutable state. However raw pointers are, strictly
speaking, marked as thread-unsafe as more of a *lint*. Doing anything useful
with a raw pointer requires dereferencing it, which is already unsafe. In that
sense, one could argue that it would be "fine" for them to be marked as thread
safe.

However it's important that they aren't thread-safe to prevent types that
contain them from being automatically marked as thread-safe. These types have
non-trivial untracked ownership, and it's unlikely that their author was
necessarily thinking hard about thread safety. In the case of `Rc`, we have a nice
example of a type that contains a `*mut` that is definitely not thread-safe.

Types that aren't automatically derived can simply implement them if desired:

```rust
struct MyBox(*mut u8);

unsafe impl Send for MyBox {}
unsafe impl Sync for MyBox {}
```

In the *incredibly rare* case that a type is inappropriately automatically
derived to be Send or Sync, then one can also unimplement Send and Sync:

```rust
#![feature(negative_impls)]

// I have some magic semantics for some synchronization primitive!
struct SpecialThreadToken(u8);

impl !Send for SpecialThreadToken {}
impl !Sync for SpecialThreadToken {}
```

Note that *in and of itself* it is impossible to incorrectly derive Send and
Sync. Only types that are ascribed special meaning by other unsafe code can
possibly cause trouble by being incorrectly Send or Sync.

Most uses of raw pointers should be encapsulated behind a sufficient abstraction
that Send and Sync can be derived. For instance all of Rust's standard
collections are Send and Sync (when they contain Send and Sync types) in spite
of their pervasive use of raw pointers to manage allocations and complex ownership.
Similarly, most iterators into these collections are Send and Sync because they
largely behave like an `&` or `&mut` into the collection.

[`Box`][box-doc] is implemented as it's own special intrinsic type by the
compiler for [various reasons][box-is-special], but we can implement something
with similar-ish behaviour ourselves to see an example of when it is sound to
implement Send and Sync. Let's call it a `Carton`.

We start by writing code to take a value allocated on the stack and transfer it
to the heap.

```rust,ignore
use std::mem::size_of;
use std::ptr::NonNull;

struct Carton<T>(NonNull<T>);

impl<T> Carton<T> {
    pub fn new(mut value: T) -> Self {
        // Allocate enough memory on the heap to store one T
        let ptr = unsafe { libc::calloc(1, size_of::<T>()) as *mut T };

        // NonNull is just a wrapper that enforces that the pointer isn't null.
        // Malloc returns null if it can't allocate.
        let mut ptr = NonNull::new(ptr).expect("We assume malloc doesn't fail");

        // Move value from the stack to the location we allocated on the heap
        unsafe {
            // Safety: The pointer returned by calloc is alligned, initialized,
            // and dereferenceable, and we have exclusive access to the pointer.
            *ptr.as_mut() = value;
        }

        Self(ptr)
    }
}
```

This isn't very useful, because once our users give us a value they have no way
to access it. [`Box`][box-doc] implements [`Deref`][deref-doc] and
[`DerefMut`][deref-mut-doc] so that you can access the inner value. Let's do
that.

```rust,ignore
use std::ops::{Deref, DerefMut};

impl<T> Deref for Carton<T> {
    type Target = T;

    fn deref(&self) -> &Self::Target {
        unsafe {
            // Safety: The pointer is aligned, initialized, and dereferenceable
            //   by the logic in [`Self::new`]. We require writers to borrow the
            //   Carton, and the lifetime of the return value is elided to the
            //   lifetime of the input. This means the borrow checker will
            //   enforce that no one can mutate the contents of the Carton until
            //   the reference returned is dropped.
            self.0.as_ref()
        }
    }
}

impl<T> DerefMut for Carton<T> {
    fn deref_mut(&mut self) -> &mut Self::Target {
        unsafe {
            // Safety: The pointer is aligned, initialized, and dereferenceable
            //   by the logic in [`Self::new]. We require writers to mutably
            //   borrow the Carton, and the lifetime of the return value is
            //   elided to the lifetime of the input. This means the borrow
            //   checker will enforce that no one else can access the contents
            //   of the Carton until the mutable reference returned is dropped.
            self.0.as_mut()
        }
    }
}
```

Finally, lets think about whether our `Carton` is Send and Sync. Something can
safely be Send unless it shares mutable state with something else without
enforcing exclusive access to it. Each `Carton` has a unique pointer, so
we're good.

```rust,ignore
// Safety: No one besides us has the raw pointer, so we can safely transfer the
// Carton to another thread.
unsafe impl<T> Send for Carton<T> {}
```

What about Sync? For `Carton` to be Sync we have to enforce that you can't
write to something stored in a `&Carton` while that same something could be read
or written to from another `&Carton`. Since you need an `&mut Carton` to
write to the pointer, and the borrow checker enforces that mutable
references must be exclusive, there are no soundness issues making `Carton`
sync either.

```rust,ignore
// Safety: Our implementation of DerefMut requires writers to mutably borrow the
// Carton, so the borrow checker will only let us have references to the Carton
// on multiple threads if no one has a mutable reference to the Carton.
unsafe impl<T> Sync for Carton<T> {}
```

TODO: better explain what can or can't be Send or Sync. Sufficient to appeal
only to data races?

[unsafe traits]: safe-unsafe-meaning.html

[box-doc]: https://doc.rust-lang.org/std/boxed/struct.Box.html

[box-is-special]: https://manishearth.github.io/blog/2017/01/10/rust-tidbits-box-is-special/

[deref-doc]: https://doc.rust-lang.org/core/ops/trait.Deref.html

[deref-mut-doc]: https://doc.rust-lang.org/core/ops/trait.DerefMut.html
