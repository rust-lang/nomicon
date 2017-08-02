# PhantomData

When working with unsafe code, we can often end up in a situation where
types or lifetimes are logically associated with a struct, but not actually
part of a field. This most commonly occurs with lifetimes. For instance, the
`Iter` for `&'a [T]` is (approximately) defined as follows:

```rust,ignore
struct Iter<'a, T: 'a> {
    ptr: *const T,
    end: *const T,
}
```

However because `'a` is unused within the struct's body, it's *unbounded*.
Because of the troubles this has historically caused, unbounded lifetimes and
types are *forbidden* in struct definitions. Therefore we must somehow refer
to these types in the body. Correctly doing this is necessary to have
correct variance and drop checking.

We do this using `PhantomData`, which is a special marker type. `PhantomData`
consumes no space, but simulates a field of the given type for the purpose of
static analysis. This was deemed to be less error-prone than explicitly telling
the type system the kind of variance that you want, while also being useful
for secondary concerns like deriving Send and Sync.

When using the *drop check eyepatch*, PhantomData also becomes important for
telling the compiler about all types that you drop that it can't see. See the
[the previous section][dropck-eyepatch] for details. This can be ignored if you
don't know what the eyepatch is.

Iter logically contains a bunch of `&'a T`s, so this is exactly what we tell
the PhantomData to simulate:

```
use std::marker;

struct Iter<'a, T: 'a> {
    ptr: *const T,
    end: *const T,
    _marker: marker::PhantomData<&'a T>,
}
```

and that's it. The lifetime will be bounded, and your iterator will be variant
over `'a` and `T`. Everything Just Works.

Here's a more extreme example based on HashMap which stores a single opaque
allocation which is used for multiple arrays of different types:

```
use std::marker;

struct HashMap {
    ptr: *mut u8,
    // The pointer actually stores keys and values
    // (and hashes, but those aren't generic)
    _marker: marker::PhantomData<(K, V)>,
}
```


## Table of `PhantomData` patterns

Here’s a table of all the most common ways `PhantomData` is used:

| Phantom type                | `'a`      | `T`                       |
|-----------------------------|-----------|---------------------------|
| `PhantomData<T>`            | -         | variant (and drop check T)|
| `PhantomData<&'a T>`        | variant   | variant                   |
| `PhantomData<&'a mut T>`    | variant   | invariant                 |
| `PhantomData<*const T>`     | -         | variant                   |
| `PhantomData<*mut T>`       | -         | invariant                 |
| `PhantomData<fn(T)>`        | -         | contravariant             |
| `PhantomData<fn() -> T>`    | -         | variant                   |
| `PhantomData<fn(T) -> T>`   | -         | invariant                 |
| `PhantomData<Cell<&'a ()>>` | invariant | -                         |





[dropck-eyepatch]: dropck-eyepatch.html
