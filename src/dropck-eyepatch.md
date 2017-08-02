# Drop Check: The Escape Patch

In spite of everything stated in the previous section, this code compiles:

```rust
fn main() {
    let (day, inspector);
    day = Box::new(0);
    inspector = vec![&*day];
    println!("{:?}", inspector);
}
```

Here instead of storing a reference in our own custom type, we use a Vec, which
is of course generic and implements Drop. Surprisingly, the compiler thinks it's
fine that the Vec stores a reference that lives exactly as long as it.

What happened?

Well, there's an unsafe escape hatch, and the standard library uses it for its
collections and owning pointer types such as Box, Rc, Vec, and BTreeMap. With
this escape hatch, we can tell the compiler that we *promise* that it's safe
for a given generic argument to dangle.

Before we proceed, I must emphasize that this is an incredibly obscure feature.
As in, many people who work on the Rust standard library and rustc itself don't
even know that this exists. So with all likelihood, no one will notice if you
don't use this feature.

**In other words: please don't use the unstable feature we're about to describe!**

This feature primarily exists because some tricky parts of rustc itself use it.
We document it here primarily for the purposes of maintaining the rust-lang
codebase itself.

So let's say we want to write this modified inspector code:

```rust,ignore
struct Inspector<'a, T: 'a> { data: &'a T }

impl<'a, T> Drop for Inspector<'a, T> {
    fn drop(&mut self) {
        println!("I was only one day from retirement!");
    }
}

fn main() {
    let (data, inspector);
    data = Box::new(0u8);
    inspector = Inspector { data: &*data };
}
```

This Inspector is perfectly safe, because it doesn't actually access its
generic data in its destructor. Sadly, the code still doesn't compile:

```text
error: `*data` does not live long enough
  --> src/main.rs:13:1
   |
12 |     inspector = Inspector { data: &*data };
   |                                    ----- borrow occurs here
13 | }
   | ^ `*data` dropped here while still borrowed
   |
   = note: values in a scope are dropped in the opposite order they are created
```

This is because as far as Rust is concerned you *could have* accessed it, and
Rust refuses to inspect your drop implementation to be sure.

With [the eyepatch RFC][eyepatch], we can *partially blind* dropck, by hiding one of our
generic parameters from it. (...by covering it with a patch. Get it? ...Eyepatch?)

The patch we apply is as follows:

```rust
#![feature(generic_param_attrs, dropck_eyepatch)]

struct Inspector<'a, T: 'a> { data: &'a T }

// Changes here:
unsafe impl<#[may_dangle] 'a, T> Drop for Inspector<'a, T> {
    fn drop(&mut self) {
        println!("I was only one day from retirement!");
    }
}

fn main() {
    let (data, inspector);
    data = Box::new(0u8);
    inspector = Inspector { data: &*data };
}
```

...and it compiles and runs!

There are two changes:

* We add `#[may_dangle]` to one of our type parameters
* We add `unsafe` to the impl block (which is required by may_dangle to emphasize the risks)

Note also that `#[may_dangle]` requires both the `generic_param_attrs`, and the
`dropck_eyepatch` features.

The may_dangle attribute tells the dropck to ignore `'a` in its analysis. Since this was
the only reason the Inspector was considered unsound (T is just a u8), our code compiles.

`#[may_dangle]` may be applied to any type parameter. For instance, if we change
`data` to just `T` (so `T = &'a u8`), then we need to blind dropck from `T`:

```rust
#![feature(generic_param_attrs, dropck_eyepatch)]

struct Inspector<T> { data: T }

// Changes here:
unsafe impl<#[may_dangle] T> Drop for Inspector<T> {
    fn drop(&mut self) {
        println!("I was only one day from retirement!");
    }
}

fn main() {
    let (data, inspector);
    data = Box::new(0u8);
    inspector = Inspector { data: &*data };
}
```




# When The Eyepatch is (Un)Sound

The general rule for when it's safe to apply the dropck eyepatch to a type parameter
`T` is that the destructor must only do things to values of type `T` that could be
done with *all* types. Basically: we can move (or copy) the values around, take
references to them, get their size/align, and drop them. Just to be clear
on why these are fine:

* Moving and Copying is just bitwise, and it's perfectly safe to copy the bits
  representing a dangling pointer.

* Static size/align computation (as with `size_of`) doesn't involve actually
  looking at instances of the type, so dangling doesn't matter.

* Dynamic size/align computation (as with `size_of_val`) is also fine, because
  it only looks at the trait object's vtable. This vtable is statically
  allocated, and can be found without looking at the actual instance's data.

* Dropping a pointer is a noop, so it doesn't matter if they're actually
  dangling.

In theory, a function that's generic over all `T` (like `mem::replace`) must also
follow these rules, but in a world with specialization that isn't necessarily true.
For instance any totally-generic function may specialize on `T: Display` to print
the values when possible (please file 100 bugs if `mem::replace` ever does this).

Also note that the following closure isn't actually generic over all values of
type `T`; its body knows the exact type of `T` and therefore can dereference
any dangling pointers `T` might contain:

```rust,ignore
impl<T, F> where F: Fn(T) { ... }
```

All `Vec<T>` and friends do in their destructors is traverse themselves using their
own structure, drop all of the `T`'s they contain, and free themselves. This is
why it's sound for them to apply the eyepatch to their parameters.






# When The Eyepatch Needs Help

Applying the eyepatch correctly isn't sufficient to get a sound drop checking.
To see why, consider this example:

```rust
#![feature(generic_param_attrs, dropck_eyepatch)]

use std::fmt::Debug;

struct Inspector<T: Debug> { data: T }

// Doesn't use eyepatch, but clearly looks at its payload. This is fine.
// Dropck will correctly require that this strictly outlives its payload.
impl<T: Debug> Drop for Inspector<T> {
    fn drop(&mut self) {
        println!("I was only {:?} days from retirement!", self.data);
    }
}

// Our own custom implementation of Box.
struct MyBox<T> {
    data: *mut T,
}

// This is uninteresting
impl<T> MyBox<T> {
    fn new(t: T) -> MyBox<T> {
        MyBox { data: Box::into_raw(Box::new(t)) }
    }
}

// The stdlib's Box impl uses may_dangle, so it should be fine for us!
// (This is true... almost)
unsafe impl<#[may_dangle] T> Drop for MyBox<T> {
    fn drop(&mut self) {
        unsafe { Box::from_raw(self.data); }
    }
}

fn inspect() {
    let (data, inspector);

    // We store this in a std box to avoid distractions
    data = Box::new(7u8);

    // This time we store an Inspector in our custom Box type
    inspector = MyBox::new(Inspector { data: &*data });

    // !!! If this compiles, the Inspector will read the dangling data here !!!
}
```


**This compiles, and will perform a use-after-free.**

Something has gone wrong. Just to check, let's replace our use of MyBox with
std's Box.

```rust,ignore
fn inspect() {
    let (data, inspector);

    data = Box::new(7u8);

    // This time we use std's Box type
    inspector = Box::new(Inspector { data: &*data });

    // !!! If this compiles, the Inspector will read the dangling data here !!!
}
```

```text
error[E0597]: `*data` does not live long enough
  --> src/main.rs:45:1
   |
42 |   inspector = Box::new(Inspector { data: &*data });
   |                                           ----- borrow occurs here
...
45 | }
   | ^ `*data` dropped here while still borrowed
   |
   = note: values in a scope are dropped in the opposite order they are created
```

Somehow, dropck now notices that we're doing something bad, and catches us.

Here's the problem: dropck doesn't know that MyBox will drop an Inspector, and
it really needs to know that to perform a proper analysis. To understand the
analysis it's trying to perform, let's step back to the "generic but no
destructor" case:


```rust,ignore
// Our own custom implementation of a Box, which doesn't actually box.
struct MyFakeBox<T> {
    data: T,
}

// This is uninteresting
impl<T> MyFakeBox<T> {
    fn new(t: T) -> MyFakeBox<T> {
        MyFakeBox { data: t }
    }
}

fn inspect() {
  let (data, inspector);

  data = Box::new(7u8);

  // This time we store an Inspector in our custom FakeBox type
  inspector = MyFakeBox::new(Inspector { data: &*data });

  // !!! If this compiles, the Inspector will read the dangling data here !!!
}
```

```
error[E0597]: `*data` does not live long enough
  --> src/main.rs:37:1
   |
34 |   inspector = MyFakeBox::new(Inspector { data: &*data });
   |                                                 ----- borrow occurs here
...
37 | }
   | ^ `*data` dropped here while still borrowed
   |
   = note: values in a scope are dropped in the opposite order they are created
```

Ok, without a destructor the compiler also performs the right analysis, even
though it should let MyFakeBox contain strictly equal lifetimes when possible.
How does it know that an Inspector will be dropped when a MyFakeBox will be?

Quite simply: it looks at MyFakeBox's fields.

Without an explicit destructor, the compiler is the one providing the destructor
implementation, and so it knows exactly what will be dropped: the fields.

It turns out that this is also the exact analysis that is applied to our MyBox
type. The *problem* is that MyBox stores a `*mut T`, and the compiler knows
dropping a `*mut T` is a noop. So it decides no Inspectors are involved in the
destruction of a MyBox, and lets this code compile (which would be correct if
that conclusion were true).




# Fixing Your Eyepatches

The solution to this problem is fairly simple: if the compiler is going to check
our fields for what we drop, let's add some more fields!

In particular, we will use the PhantomData type to simulate a stored T.
(We'll discuss PhantomData more in the next section. For now, take it for
granted.)

```rust,ignore
use std::marker::PhantomData;

// Our own custom implementation of Box.
struct MyBox<T> {
    data: *mut T,
    _boo: PhantomData<T>, // Tell the compiler we drop a T!
}

// This is still uninteresting
impl<T> MyBox<T> {
  fn new(t: T) -> MyBox<T> {
    MyBox {
        data: Box::into_raw(Box::new(t)),
        _boo: PhantomData,
    }
  }
}

// Completely unchanged! We got this part right!
unsafe impl<#[may_dangle] T> Drop for MyBox<T> {
    fn drop(&mut self) {
        unsafe { Box::from_raw(self.data); }
    }
}

fn inspect() {
    let (data, inspector);

    data = Box::new(7u8);

    // Back to our MyBox type
    inspector = MyBox::new(Inspector { data: &*data });

    // !!! If this compiles, the Inspector will read the dangling data here !!!
}
```

```text
   Compiling playground v0.0.1 (file:///playground)
error[E0597]: `*data` does not live long enough
  --> src/main.rs:50:1
   |
47 |   inspector = MyBox::new(Inspector { data: &*data });
   |                                             ----- borrow occurs here
...
50 | }
   | ^ `*data` dropped here while still borrowed
   |
   = note: values in a scope are dropped in the opposite order they are created
```

Hurray! It worked!

And just to check that we can still store dangling things when it's sound:

```rust,ignore
fn inspect() {
    let (data, non_inspector);

    data = Box::new(7u8);

    // Our custom box type, but no inspector.
    non_inspector = MyBox::new(&*data);
}
```

Compiles fine! Great! 🎉




# Dropck Eyepatch Summary (TL;DR)

When a generic type provides a destructor, the compiler will conservatively
disallow any of the type parameters living exactly as long as that type.

With the dropck eyepatch, we can tell it to ignore certain type parameters
which the destructor only does "trivial" things with. Which is to say, `MyType<T>`
doesn't do anything that `Vec<T>` wouldn't do with `T`.

However we then also become responsible for telling dropck about all the types
*related* to T that we drop. It knows we will drop anything in our fields, but
things like raw pointers "trick" it, as dropping a raw pointer does nothing.

To solve this, you should include a `PhantomData` field that stores each of the
types related to T that you may Drop.

**Note that this includes any associated items that you may drop in the destructor.**

For instance, if you have a destructor for `MyType<I: IntoIter>` that calls
`into_iter()`, you should probably include `PhantomData<I::IntoIter>`.

Yes, this is a big hassle and easy to get wrong. Please don't use the eyepatch.





[eyepatch]: https://github.com/rust-lang/rfcs/blob/master/text/1327-dropck-param-eyepatch.md
