# Subtyping and Variance

Subtyping is a relationship between types that allows statically typed
languages to be a bit more flexible and permissive.

The most common and easy to understand example of this can be found in
languages with inheritance. Consider an Animal type which has an `eat()`
method, and a Cat type which extends Animal, adding a `meow()` method.
Without subtyping, if someone were to write a `feed(Animal)` function, they
wouldn't be able to pass a Cat to this function, because a Cat isn't *exactly*
an Animal. But being able to pass a Cat where an Animal is expected seems
fairly reasonable. After all, a Cat is just an Animal *and more*. Something
having extra features that can be ignored shouldn't be any impediment to
using it!

This is exactly what subtyping lets us do. Because a Cat is an Animal *and more*
we say that Cat is a *subtype* of Animal. We then say that anywhere a value of
a certain type is expected, a value with a subtype can also be supplied. Ok
actually it's a lot more complicated and subtle than that, but that's the
basic intuition that gets you by in 99% of the cases. We'll cover why it's
*only* 99% later in this section.

Although Rust doesn't have any notion of structural inheritance, it *does*
include subtyping. In Rust, subtyping derives entirely from lifetimes. Since
lifetimes are regions of code, we can partially order them based on the
*contains* (outlives) relationship.

Subtyping on lifetimes is in terms of that relationship: if `'big: 'small`
("big contains small" or "big outlives small"), then `'big` is a subtype
of `'small`. This is a large source of confusion, because it seems backwards
to many: the bigger region is a *subtype* of the smaller region. But it makes
sense if you consider our Animal example: *Cat* is an Animal *and more*,
just as `'big` is `'small` *and more*.

Put another way, if someone wants a reference that lives for `'small`,
usually what they actually mean is that they want a reference that lives
for *at least* `'small`. They don't actually care if the lifetimes match
exactly. For this reason `'static`, the forever lifetime, is a subtype
of every lifetime.

Higher-ranked lifetimes are also subtypes of every concrete lifetime. This is
because taking an arbitrary lifetime is strictly more general than taking a
specific one.

(The typed-ness of lifetimes is a fairly arbitrary construct that some
disagree with. However it simplifies our analysis to treat lifetimes
and types uniformly.)

However you can't write a function that takes a value of type `'a`! Lifetimes
are always just part of another type, so we need a way of handling that.
To handle it, we need to talk about *variance*.





# Variance

Variance is where things get a bit complicated.

Variance is a property that *type constructors* have with respect to their
arguments. A type constructor in Rust is a generic type with unbound arguments.
For instance `Vec` is a type constructor that takes a `T` and returns a
`Vec<T>`. `&` and `&mut` are type constructors that take two inputs: a
lifetime, and a type to point to.

A type constructor's *variance* is how the subtyping of its inputs affects the
subtyping of its outputs. There are three kinds of variance in Rust:

* F is *covariant* over `T` if `T` being a subtype of `U` implies
  `F<T>` is a subtype of `F<U>` (subtyping "passes through")
* F is *contravariant* over `T` if `T` being a subtype of `U` implies
  `F<U>` is a subtype of `F<U>` (subtyping is "inverted")
* F is *invariant* over `T` otherwise (no subtyping relation can be derived)

It should be noted that covariance is *far* more common and important than
contravariance in Rust. The existence of contravariance in Rust can mostly
be ignored.

Some important variances (which we will explain in detail below):

* `&'a T` is covariant over `'a` and `T` (as is `*const T` by metaphor)
* `&'a mut T` is covariant over `'a` but invariant over `T`
* `fn(T) -> U` is **contra**variant over `T`, but covariant over `U`
* `Box`, `Vec`, and all other collections are covariant over the types of
  their contents
* `UnsafeCell<T>`, `Cell<T>`, `RefCell<T>`, `Mutex<T>` and all other
  interior mutability types are invariant over T (as is `*mut T` by metaphor)

To understand why these variances are correct and desirable, we will consider
several examples.

We have already covered why `&'a T` should be covariant over `'a` when
introducing subtyping: it's desirable to be able to pass longer-lived things
where shorter-lived things are needed.

Similar reasoning applies to why it should be covariant over T: it's reasonable
to be able to pass `&&'static str` where an `&&'a str` is expected. The
additional level of indirection doesn't change the desire to be able to pass
longer lived things where shorter lived things are expected.

However this logic doesn't apply to `&mut`. To see why `&mut` should
be invariant over T, consider the following code:

```rust,ignore
fn overwrite<T: Copy>(input: &mut T, new: &mut T) {
    *input = *new;
}

fn main() {
    let mut forever_str: &'static str = "hello";
    {
        let string = String::from("world");
        overwrite(&mut forever_str, &mut &*string);
    }
    // Oops, printing free'd memory
    println!("{}", forever_str);
}
```

The signature of `overwrite` is clearly valid: it takes mutable references to
two values of the same type, and overwrites one with the other.

But, if `&mut T` was covariant over T, then `&mut &'static str` would be a
subtype of `&mut &'a str`, since `&'static str` is a subtype of `&'a str`.
Therefore the lifetime of `forever_str` would successfully be "shrunk" down
to the shorter lifetime of `string`, and `overwrite` would be called successfully.
`string` would subsequently be dropped, and `forever_str` would point to
freed memory when we print it! Therefore `&mut` should be invariant.

This is the general theme of variance vs invariance: if variance would allow you
to store a short-lived value in a longer-lived slot, then invariance must be used.

More generally, the soundness of subtyping and variance is based on the idea that its ok to
forget details, but with mutable references there's always someone (the original
value being referenced) that remembers the forgotten details and will assume
that those details haven't changed. If we do something to invalidate those details,
the original location can behave unsoundly.

However it *is* sound for `&'a mut T` to be covariant over `'a`. The key difference
between `'a` and T is that `'a` is a property of the reference itself,
while T is something the reference is borrowing. If you change T's type, then
the source still remembers the original type. However if you change the
lifetime's type, no one but the reference knows this information, so it's fine.
Put another way: `&'a mut T` owns `'a`, but only *borrows* T.

`Box` and `Vec` are interesting cases because they're covariant, but you can
definitely store values in them! This is where Rust's typesystem allows it to
be a bit more clever than others. To understand why it's sound for owning
containers to be covariant over their contents, we must consider
the two ways in which a mutation may occur: by-value or by-reference.

If mutation is by-value, then the old location that remembers extra details is
moved out of, meaning it can't use the value anymore. So we simply don't need to
worry about anyone remembering dangerous details. Put another way, applying
subtyping when passing by-value *destroys details forever*. For example, this
compiles and is fine:

```rust
fn get_box<'a>(str: &'a str) -> Box<&'a str> {
    // String literals are `&'static str`s, but it's fine for us to
    // "forget" this and let the caller think the string won't live that long.
    Box::new("hello")
}
```

If mutation is by-reference, then our container is passed as `&mut Vec<T>`. But
`&mut` is invariant over its value, so `&mut Vec<T>` is actually invariant over `T`.
So the fact that `Vec<T>` is covariant over `T` doesn't matter at all when
mutating by-reference.

But being covariant still allows `Box` and `Vec` to be weakened when shared
immutably. So you can pass a `&Vec<&'static str>` where a `&Vec<&'a str>` is
expected.

The invariance of the cell types can be seen as follows: `&` is like an `&mut`
for a cell, because you can still store values in them through an `&`. Therefore
cells must be invariant to avoid lifetime smuggling.

`fn` is the most subtle case because they have mixed variance, and in fact are
the only source of **contra**variance. To see why `fn(T) -> U` should be contravariant
over T, consider the following function signature:

```rust,ignore
// 'a is derived from some parent scope
fn foo(&'a str) -> usize;
```

This signature claims that it can handle any `&str` that lives at least as
long as `'a`. Now if this signature was **co**variant over `&'a str`, that
would mean

```rust,ignore
fn foo(&'static str) -> usize;
```

could be provided in its place, as it would be a subtype. However this function
has a stronger requirement: it says that it can only handle `&'static str`s,
and nothing else. Giving `&'a str`s to it would be unsound, as it's free to
assume that what it's given lives forever. Therefore functions definitely shouldn't
be **co**variant over their arguments.

However if we flip it around and use **contra**variance, it *does* work! If
something expects a function which can handle strings that live forever,
it makes perfect sense to instead provide a function that can handle
strings that live for *less* than forever. So

```rust,ignore
fn foo(&'a str) -> usize;
```

can be passed where

```rust,ignore
fn foo(&'static str) -> usize;
```

is expected.

To see why `fn(T) -> U` should be **co**variant over U, consider the following
function signature:

```rust,ignore
// 'a is derived from some parent scope
fn foo(usize) -> &'a str;
```

This signature claims that it will return something that outlives `'a`. It is
therefore completely reasonable to provide

```rust,ignore
fn foo(usize) -> &'static str;
```

in its place, as it does indeed return things that outlive `'a`. Therefore
functions are covariant over their return type.

`*const` has the exact same semantics as `&`, so variance follows. `*mut` on the
other hand can dereference to an `&mut` whether shared or not, so it is marked
as invariant just like cells.

This is all well and good for the types the standard library provides, but
how is variance determined for type that *you* define? A struct, informally
speaking, inherits the variance of its fields. If a struct `Foo`
has a generic argument `A` that is used in a field `a`, then Foo's variance
over `A` is exactly `a`'s variance. However if `A` is used in multiple fields:

* If all uses of A are covariant, then Foo is covariant over A
* If all uses of A are contravariant, then Foo is contravariant over A
* Otherwise, Foo is invariant over A

```rust
use std::cell::Cell;

struct Foo<'a, 'b, A: 'a, B: 'b, C, D, E, F, G, H, In, Out, Mixed> {
    a: &'a A,     // covariant over 'a and A
    b: &'b mut B, // covariant over 'b and invariant over B

    c: *const C,  // covariant over C
    d: *mut D,    // invariant over D

    e: E,         // covariant over E
    f: Vec<F>,    // covariant over F
    g: Cell<G>,   // invariant over G

    h1: H,        // would also be variant over H except...
    h2: Cell<H>,  // invariant over H, because invariance wins all conflicts

    i: fn(In) -> Out,       // contravariant over In, covariant over Out

    k1: fn(Mixed) -> usize, // would be contravariant over Mixed except..
    k2: Mixed,              // invariant over Mixed, because invariance wins all conflicts
}
```
