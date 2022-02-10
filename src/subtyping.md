# Subtyping and Variance

Rust uses lifetimes to track the relationships between borrows and ownership.
However, a naive implementation of lifetimes would be either too restrictive,
or permit undefined behavior.

In order to allow flexible usage of lifetimes
while also preventing their misuse, Rust uses a combination of **Subtyping** and **Variance**.

## Subtyping

Subtyping is the idea that one type can be a *subtype* of another.

Let's define that `Sub` is a subtype of `Super` (we'll be using the notation `Sub: Super` throughout this chapter)

What this is suggesting to us is that the set of *requirements* that `Super` defines
are completely satisfied by `Sub`. `Sub` may then have more requirements.

An example of simple subtyping that exists in the language are [supertraits](https://doc.rust-lang.org/stable/book/ch19-03-advanced-traits.html?highlight=supertraits#using-supertraits-to-require-one-traits-functionality-within-another-trait)

```rust
use std::fmt;

trait OutlinePrint: fmt::Display {
    fn outline_print(&self) {
        todo!()
    }
}
```

Here, we have that `OutlinePrint: fmt::Display` (`OutlinePrint` is a *subtype* of `Display`),
because it has all the requirements of `fmt::Display`, plus the `outline_print` function.

However, subtyping in traits is not that interesting in the case of Rust.
Here in the nomicon, we're going to focus more with how subtyping interacts with **lifetimes**

Take this example

```rust
fn debug<T: std::fmt::Debug>(a: T, b: T) {
    println!("a = {:?} b = {:?}", a, b);
}

fn main() {
    let hello: &'static str = "hello";
    {
        let world = String::from("world");
        let world = &world; // 'b has a shorter lifetime than 'static
        debug(hello, world);
    }
}
```

In an overly restrictive implementation of lifetimes, since `a` and `b` have differeing lifetimes,
we might see the following error:

```text
error[E0308]: mismatched types
 --> src/main.rs:10:16
   |
10 |         debug(hello, world);
   |                      ^
   |                      |
   |                      expected `&'static str`, found struct `&'b str`
```

This is over-restrictive. In this case, what we want is to accept any type that lives *at least as long* as `'b`.
Let's try using subtyping with our lifetimes.

Let's define a lifetime to have the a simple set of requirements: `'a` defines a region of code in which a value will be alive.
Now that we have a defined set of requirements for lifetimes, we can define how they relate to each other.
`'a: 'b` if and only if `'a` defines a region of code that **completely contains** `'b`.

`'a` may define a region larger than `'b`, but that still fits our definition.
Going back to our example above, we can say that `'static: 'b`.

For now, let's accept the idea that subtypes of lifetimes can be passed through references (more on this in [Variance](#variance)),
eg. `&'static str` is a subtype of `&'b str`, then we can let them coerce, and then the example above will compile

```rust
fn debug<T: std::fmt::Debug>(a: T, b: T) {
    println!("a = {:?} b = {:?}", a, b);
}

fn main() {
    let hello: &'static str = "hello";
    {
        let world = String::from("world");
        let world = &world; // 'b has a shorter lifetime than 'static
        debug(hello, world); // a silently converts from `&'static str` into `&'b str`
    }
}
```

## Variance

Above, we glossed over the fact that `'static: 'b` implied that `&'static T: &'b T`. This uses a property known as variance.
It's not always as simple as this example though, to understand that let's try extend this example a bit

```rust,compile_fail
fn assign<T>(input: &mut T, val: T) {
    *input = val;
}

fn main() {
    let mut hello: &'static str = "hello";
    {
        let world = String::from("world");
        assign(&mut hello, &world);
    }
}
```

If this were to compile, this would have a memory bug.

If we were to expand this out, we'd see that we're trying to assign a `&'b str` into a `&'static str`,
but the problem is that as soon as `b` goes out of scope, `a` is now invalid, even though it's supposed to have a `'static` lifetime.

However, the implementation of `assign` is valid.
Therefore, this must mean that `&mut &'static str` should **not** a *subtype* of `&mut &'b str`,
even if `'static` is a subtype of `'b`.

Variance is the way that Rust defines the relationships of subtypes through their *type constructor*.
A type constructor in Rust is any generic type with unbound arguments.
For instance `Vec` is a type constructor that takes a type `T` and returns
`Vec<T>`. `&` and `&mut` are type constructors that take two inputs: a
lifetime, and a type to point to.

> NOTE: For convenience we will often refer to `F<T>` as a type constructor just so
> that we can easily talk about `T`. Hopefully this is clear in context.

A type constructor F's *variance* is how the subtyping of its inputs affects the
subtyping of its outputs. There are three kinds of variance in Rust. Given two
types `Sub` and `Super`, where `Sub` is a subtype of `Super`:

* F is **covariant** if `F<Sub>` is a subtype of `F<Super>` (the subtype property is passed through)
* F is **contravariant** if `F<Super>` is a subtype of `F<Sub>` (the subtype property is "inverted")
* F is **invariant** otherwise (no subtyping relationship exists)

If we remember from the above examples,
it was ok for us to treat `&'a T` as a subtype of `&'b T` if `'a: 'b`,
therefore we can say that `&'a T` is *covariant* over `'a`.

Also, we saw that it was not ok for us to treat `&mut &'a T` as a subtype of `&mut &'b T`,
therefore we can say that `&mut T` is *invariant* over `T`

Here is a table of some other type constructors and their variances:

|   |                 |     'a    |         T         |     U     |
|---|-----------------|:---------:|:-----------------:|:---------:|
|   | `&'a T `        | covariant | covariant         |           |
|   | `&'a mut T`     | covariant | invariant         |           |
|   | `Box<T>`        |           | covariant         |           |
|   | `Vec<T>`        |           | covariant         |           |
|   | `UnsafeCell<T>` |           | invariant         |           |
|   | `Cell<T>`       |           | invariant         |           |
|   | `fn(T) -> U`    |           | **contra**variant | covariant |
|   | `*const T`      |           | covariant         |           |
|   | `*mut T`        |           | invariant         |           |

Some of these can be explained simply in relation to the others:

* `Vec<T>` and all other owning pointers and collections follow the same logic as `Box<T>`
* `Cell<T>` and all other interior mutability types follow the same logic as `UnsafeCell<T>`
* `UnsafeCell<T>` having interior mutability gives it the same variance properties as `&mut T`
* `*const T` follows the logic of `&T`
* `*mut T` follows the logic of `&mut T` (or `UnsafeCell<T>`)

For more types, see the ["Variance" section][variance-table] on the reference.

[variance-table]: ../reference/subtyping.html#variance

> NOTE: the *only* source of contravariance in the language is the arguments to
> a function, which is why it really doesn't come up much in practice. Invoking
> contravariance involves higher-order programming with function pointers that
> take references with specific lifetimes (as opposed to the usual "any lifetime",
> which gets into higher rank lifetimes, which work independently of subtyping).

Now that we have some more formal understanding of variance,
let's go through some more examples in more detail.

```rust,compile_fail
fn assign<T>(input: &mut T, val: T) {
    *input = val;
}

fn main() {
    let mut hello: &'static str = "hello";
    {
        let world = String::from("world");
        assign(&mut hello, &world);
    }
}
```

And what do we get when we run this?

```text
error[E0597]: `world` does not live long enough
  --> src/main.rs:9:28
   |
6  |     let mut hello: &'static str = "hello";
   |                    ------------ type annotation requires that `world` is borrowed for `'static`
...
9  |         assign(&mut hello, &world);
   |                            ^^^^^^ borrowed value does not live long enough
10 |     }
   |     - `world` dropped here while still borrowed
```

Good, it doesn't compile! Let's break down what's happening here in detail.

First let's look at the `assign` function:

```rust
fn assign<T>(input: &mut T, val: T) {
    *input = val;
}
```

All it does is take a mutable reference and a value and overwrite the referent with it.
What's important about this function is that it creates a type equality constraint. It
clearly says in its signature the referent and the value must be the *exact same* type.

Meanwhile, in the caller we pass in `&mut &'static str` and `&'spike_str str`.

Because `&mut T` is invariant over `T`, the compiler concludes it can't apply any subtyping
to the first argument, and so `T` must be exactly `&'static str`.

This is counter to the `&T` case

```rust
fn debug<T: std::fmt::Debug>(a: T, b: T) {
    println!("a = {:?} b = {:?}", a, b);
}
```

Where similarly `a` and `b` must have the same type `T`.
But since `&'a T` *is* covariant over `'a`, we are allowed to perform subtyping.
So the compiler decides that `&'static str` can become `&'b str` if and only if
`&'static str` is a subtype of `&'b str`, which will hold if `'static: 'b`.
This is true, so the compiler is happy to continue compiling this code.

`Box<T>` is also *covariant* over `T`. This would make sense, since it's supposed to be
usable the same as `&T`. If you try to mutate the box, you'll need a `&mut Box<T>` and the
invariance of `&mut` will kick in here.

Only one thing left to explain: function pointers.

To see why `fn(T) -> U` should be covariant over `U`, consider the following signature:

<!-- ignore: simplified code -->
```rust,ignore
fn get_str() -> &'a str;
```

This function claims to produce a `str` bound by some liftime `'a`. As such, it is perfectly valid to
provide a function with the following signature instead:

<!-- ignore: simplified code -->
```rust,ignore
fn get_static() -> &'static str;
```

So when the function is called, all it's expecting is a `&str` which lives at least the lifetime of `'a`,
it doesn't matter if the value actually lives longer.

However, the same logic does not apply to *arguments*. Consider trying to satisfy:

<!-- ignore: simplified code -->
```rust,ignore
fn store_ref(&'a str);
```

with:

<!-- ignore: simplified code -->
```rust,ignore
fn store_static(&'static str);
```

The first function can accept any string reference as long as it lives at least for `'a`,
but the second cannot accept a string reference that lives for any duration less than `'static`,
which would cause a conflict.
Covariance doesn't work here. But if we flip it around, it actually *does*
work! If we need a function that can handle `&'static str`, a function that can handle *any* reference lifetime
will surely work fine.

And that's why function types, unlike anything else in the language, are
**contra**variant over their arguments.

Now, this is all well and good for the types the standard library provides, but
how is variance determined for types that *you* define? A struct, informally
speaking, inherits the variance of its fields. If a struct `MyType`
has a generic argument `A` that is used in a field `a`, then MyType's variance
over `A` is exactly `a`'s variance over `A`.

However if `A` is used in multiple fields:

* If all uses of `A` are covariant, then MyType is covariant over `A`
* If all uses of `A` are contravariant, then MyType is contravariant over `A`
* Otherwise, MyType is invariant over `A`

```rust
use std::cell::Cell;

struct MyType<'a, 'b, A: 'a, B: 'b, C, D, E, F, G, H, In, Out, Mixed> {
    a: &'a A,     // covariant over 'a and A
    b: &'b mut B, // covariant over 'b and invariant over B

    c: *const C,  // covariant over C
    d: *mut D,    // invariant over D

    e: E,         // covariant over E
    f: Vec<F>,    // covariant over F
    g: Cell<G>,   // invariant over G

    h1: H,        // would also be covariant over H except...
    h2: Cell<H>,  // invariant over H, because invariance wins all conflicts

    i: fn(In) -> Out,       // contravariant over In, covariant over Out

    k1: fn(Mixed) -> usize, // would be contravariant over Mixed except..
    k2: Mixed,              // invariant over Mixed, because invariance wins all conflicts
}
```
