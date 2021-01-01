# Deref

Alright. We now have a way to make, clone, and destroy `Arc`s, but how do we get
to the data inside?

What we need now is an implementation of `Deref`.

We'll need to import the trait:
```rust,ignore
use std::ops::Deref;
```

And here's the implementation:
```rust,ignore
impl<T> Deref for Arc<T> {
    type Target = T;

    fn deref(&self) -> &T {
        &self.inner().data
    }
}
```

Pretty simple, eh? This simply dereferences the `NonNull` pointer to the
`ArcInner<T>`, then gets a reference to the data inside.
