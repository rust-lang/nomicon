# Subtyping and Variance

Rust uses lifetimes to track the relationships between borrows and ownership.
However, a naive implementation of lifetimes would be either too restrictive,
or permit undefined behavior.

In order to allow flexible usage of lifetimes
while also preventing mis-use, Rust uses a combination of **Subtyping** and **Variance**.

## Subtyping

Subtyping is the idea that one type can be a *subtype* of another.
Let's define that `A: B` is equivalent to saying '`A` is a subtype of `B`'.
What this is suggesting to us is that the set of *requirements* that `B` defines
are completely satisfied by `A`. `A` may then have more requirements.

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
    let a: &'static str = "hello";
    {
        let b = String::from("world");
        let b = &b; // 'b has a shorter lifetime than 'static
        debug(a, b);
    }
}
```

In an overly restrictive implementation of lifetimes, since `a` and `b` have differeing lifetimes,
we might see the following error:

```text
error[E0308]: mismatched types
 --> src/main.rs:10:16
   |
10 |         debug(a, b);
   |                  ^
   |                  |
   |                  expected `&'static str`, found struct `&'b str`
```

This is over-restrictive. In this case, what we want is to accept any type that lives *at least as long* as `'b`.
Let's try using subtyping with our lifetimes.

Let's define a lifetime to have the a simple set of requirements: `'a` defines a region of code in which a value will be alive.
Now that we have a defined set of requirements for lifetimes, we can define how they relate to each other.
`'a: 'b` if and only if `'a` defines a region of code that **completely contains** `'b`.

`'a` may define a region larger than `'b`, but that still fits our definition.
Going back to our example above, we can say that `'static: 'b`.

For now, let's accept the idea that subtypes of lifetimes can be transitive (more on this in [Variance](#variance>)),
eg. `&'static str` is a subtype of `&'b str`, then we can let them coerce, and then the example above will compile

```rust
fn debug<T: std::fmt::Debug>(a: T, b: T) {
    println!("a = {:?} b = {:?}", a, b);
}

fn main() {
    let a: &'static str = "hello";
    {
        let b = String::from("world");
        let b = &b; // 'b has a shorter lifetime than 'static
        debug(a, b); // a silently converts from `&'static str` into `&'b str`
    }
}
```

## Variance

Above, we glossed over the fact that `'static: 'b` implied that `&'static T: &'b T`. This uses a property known as variance.
It's not always as simple as this example though, to understand that let's try extend this example a bit

```rust,compile_fail
fn debug<T>(a: &mut T, b: T) {
    *a = b;
}

fn main() {
    let mut a: &'static str = "hello";
    {
        let b = String::from("world");
        let b = &b;
        debug(&mut a, b);
    }
}
```

This has a memory bug in it.

If we were to expand this out, we'd see that we're trying to assign a `&'b str` into a `&'static str`,
but the problem is that as soon as `b` goes out of scope, `a` is now invalid, even though it's supposed to have a `'static` lifetime.

However, the implementation of `debug` is valid.
Therefore, this must mean that `&mut &'static str` should **not** a *subtype* of `&mut &'b str`,
even if `'static` is a subtype of `'b`.

Variance is the way that Rust defines the transitivity of subtypes through their *type constructor*.
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
| * | `&'a T `        | covariant | covariant         |           |
| * | `&'a mut T`     | covariant | invariant         |           |
| * | `Box<T>`        |           | covariant         |           |
|   | `Vec<T>`        |           | covariant         |           |
| * | `UnsafeCell<T>` |           | invariant         |           |
|   | `Cell<T>`       |           | invariant         |           |
| * | `fn(T) -> U`    |           | **contra**variant | covariant |
|   | `*const T`      |           | covariant         |           |
|   | `*mut T`        |           | invariant         |           |

The types with \*'s are the ones we will be focusing on, as they are in
some sense "fundamental". All the others can be understood by analogy to the others:

* `Vec<T>` and all other owning pointers and collections follow the same logic as `Box<T>`
* `Cell<T>` and all other interior mutability types follow the same logic as `UnsafeCell<T>`
* `*const T` follows the logic of `&T`
* `*mut T` follows the logic of `&mut T` (or `UnsafeCell<T>`)

For more types, see the ["Variance" section][variance-table] on the reference.

[variance-table]: ../reference/subtyping.html#variance

> NOTE: the *only* source of contravariance in the language is the arguments to
> a function, which is why it really doesn't come up much in practice. Invoking
> contravariance involves higher-order programming with function pointers that
> take references with specific lifetimes (as opposed to the usual "any lifetime",
> which gets into higher rank lifetimes, which work independently of subtyping).

Ok, that's enough type theory! Let's try to apply the concept of variance to Rust
and look at some examples.

First off, let's revisit the meowing dog example:

<!-- ignore: simplified code -->
```rust,ignore
fn evil_feeder(pet: &mut Animal) {
    let spike: Dog = ...;

    // `pet` is an Animal, and Dog is a subtype of Animal,
    // so this should be fine, right..?
    *pet = spike;
}

fn main() {
    let mut mr_snuggles: Cat = ...;
    evil_feeder(&mut mr_snuggles);  // Replaces mr_snuggles with a Dog
    mr_snuggles.meow();             // OH NO, MEOWING DOG!
}
```

If we look at our table of variances, we see that `&mut T` is *invariant* over `T`.
As it turns out, this completely fixes the issue! With invariance, the fact that
Cat is a subtype of Animal doesn't matter; `&mut Cat` still won't be a subtype of
`&mut Animal`. The static type checker will then correctly stop us from passing
a Cat into `evil_feeder`.

The soundness of subtyping is based on the idea that it's ok to forget unnecessary
details. But with references, there's always someone that remembers those details:
the value being referenced. That value expects those details to keep being true,
and may behave incorrectly if its expectations are violated.

The problem with making `&mut T` covariant over `T` is that it gives us the power
to modify the original value *when we don't remember all of its constraints*.
And so, we can make someone have a Dog when they're certain they still have a Cat.

With that established, we can easily see why `&T` being covariant over `T` *is*
sound: it doesn't let you modify the value, only look at it. Without any way to
mutate, there's no way for us to mess with any details. We can also see why
`UnsafeCell` and all the other interior mutability types must be invariant: they
make `&T` work like `&mut T`!

Now what about the lifetime on references? Why is it ok for both kinds of references
to be covariant over their lifetimes? Well, here's a two-pronged argument:

First and foremost, subtyping references based on their lifetimes is *the entire point
of subtyping in Rust*. The only reason we have subtyping is so we can pass
long-lived things where short-lived things are expected. So it better work!

Second, and more seriously, lifetimes are only a part of the reference itself. The
type of the referent is shared knowledge, which is why adjusting that type in only
one place (the reference) can lead to issues. But if you shrink down a reference's
lifetime when you hand it to someone, that lifetime information isn't shared in
any way. There are now two independent references with independent lifetimes.
There's no way to mess with the original reference's lifetime using the other one.

Or rather, the only way to mess with someone's lifetime is to build a meowing dog.
But as soon as you try to build a meowing dog, the lifetime should be wrapped up
in an invariant type, preventing the lifetime from being shrunk. To understand this
better, let's port the meowing dog problem over to real Rust.

In the meowing dog problem we take a subtype (Cat), convert it into a supertype
(Animal), and then use that fact to overwrite the subtype with a value that satisfies
the constraints of the supertype but not the subtype (Dog).

So with lifetimes, we want to take a long-lived thing, convert it into a
short-lived thing, and then use that to write something that doesn't live long
enough into the place expecting something long-lived.

Here it is:

```rust,compile_fail
fn evil_feeder<T>(input: &mut T, val: T) {
    *input = val;
}

fn main() {
    let mut mr_snuggles: &'static str = "meow! :3";  // mr. snuggles forever!!
    {
        let spike = String::from("bark! >:V");
        let spike_str: &str = &spike;                // Only lives for the block
        evil_feeder(&mut mr_snuggles, spike_str);    // EVIL!
    }
    println!("{}", mr_snuggles);                     // Use after free?
}
```

And what do we get when we run this?

```text
error[E0597]: `spike` does not live long enough
  --> src/main.rs:9:31
   |
6  |     let mut mr_snuggles: &'static str = "meow! :3";  // mr. snuggles forever!!
   |                          ------------ type annotation requires that `spike` is borrowed for `'static`
...
9  |         let spike_str: &str = &spike;                // Only lives for the block
   |                               ^^^^^^ borrowed value does not live long enough
10 |         evil_feeder(&mut mr_snuggles, spike_str);    // EVIL!
11 |     }
   |     - `spike` dropped here while still borrowed
```

Good, it doesn't compile! Let's break down what's happening here in detail.

First let's look at the new `evil_feeder` function:

```rust
fn evil_feeder<T>(input: &mut T, val: T) {
    *input = val;
}
```

All it does is take a mutable reference and a value and overwrite the referent with it.
What's important about this function is that it creates a type equality constraint. It
clearly says in its signature the referent and the value must be the *exact same* type.

Meanwhile, in the caller we pass in `&mut &'static str` and `&'spike_str str`.

Because `&mut T` is invariant over `T`, the compiler concludes it can't apply any subtyping
to the first argument, and so `T` must be exactly `&'static str`.

The other argument is only an `&'a str`, which *is* covariant over `'a`. So the compiler
adopts a constraint: `&'spike_str str` must be a subtype of `&'static str` (inclusive),
which in turn implies `'spike_str` must be a subtype of `'static` (inclusive). Which is to say,
`'spike_str` must contain `'static`. But only one thing contains `'static` -- `'static` itself!

This is why we get an error when we try to assign `&spike` to `spike_str`. The
compiler has worked backwards to conclude `spike_str` must live forever, and `&spike`
simply can't live that long.

So even though references are covariant over their lifetimes, they "inherit" invariance
whenever they're put into a context that could do something bad with that. In this case,
we inherited invariance as soon as we put our reference inside an `&mut T`.

As it turns out, the argument for why it's ok for Box (and Vec, Hashmap, etc.) to
be covariant is pretty similar to the argument for why it's ok for
references to be covariant: as soon as you try to stuff them in something like a
mutable reference, they inherit invariance and you're prevented from doing anything
bad.

However, Box makes it easier to focus on the by-value aspect of references that we
partially glossed over.

Unlike a lot of languages which allow values to be freely aliased at all times,
Rust has a very strict rule: if you're allowed to mutate or move a value, you
are guaranteed to be the only one with access to it.

Consider the following code:

<!-- ignore: simplified code -->
```rust,ignore
let mr_snuggles: Box<Cat> = ..;
let spike: Box<Dog> = ..;

let mut pet: Box<Animal>;
pet = mr_snuggles;
pet = spike;
```

There is no problem at all with the fact that we have forgotten that `mr_snuggles` was a Cat,
or that we overwrote him with a Dog, because as soon as we moved mr_snuggles to a variable
that only knew he was an Animal, **we destroyed the only thing in the universe that
remembered he was a Cat**!

In contrast to the argument about immutable references being soundly covariant because they
don't let you change anything, owned values can be covariant because they make you
change *everything*. There is no connection between old locations and new locations.
Applying by-value subtyping is an irreversible act of knowledge destruction, and
without any memory of how things used to be, no one can be tricked into acting on
that old information!

Only one thing left to explain: function pointers.

To see why `fn(T) -> U` should be covariant over `U`, consider the following signature:

<!-- ignore: simplified code -->
```rust,ignore
fn get_animal() -> Animal;
```

This function claims to produce an Animal. As such, it is perfectly valid to
provide a function with the following signature instead:

<!-- ignore: simplified code -->
```rust,ignore
fn get_animal() -> Cat;
```

After all, Cats are Animals, so always producing a Cat is a perfectly valid way
to produce Animals. Or to relate it back to real Rust: if we need a function
that is supposed to produce something that lives for `'short`, it's perfectly
fine for it to produce something that lives for `'long`. We don't care, we can
just forget that fact.

However, the same logic does not apply to *arguments*. Consider trying to satisfy:

<!-- ignore: simplified code -->
```rust,ignore
fn handle_animal(Animal);
```

with:

<!-- ignore: simplified code -->
```rust,ignore
fn handle_animal(Cat);
```

The first function can accept Dogs, but the second function absolutely can't.
Covariance doesn't work here. But if we flip it around, it actually *does*
work! If we need a function that can handle Cats, a function that can handle *any*
Animal will surely work fine. Or to relate it back to real Rust: if we need a
function that can handle anything that lives for at least `'long`, it's perfectly
fine for it to be able to handle anything that lives for at least `'short`.

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
