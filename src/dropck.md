# Drop Check

When looking at the *outlives* relationship in previous sections, we never
considered the case where two values have the *exact* same lifetime. We made
this clear by desugarring each let statement into its own scope:

```rust,ignore
let x;
let y;
```

```rust,ignore
{
    let x;
    {
        let y;
    }
}
```

But what if we write the following let statement?

```rust,ignore
let (x, y) = (vec![], vec![]);
```

Does either value strictly outlive the other? The answer is in fact *no*,
neither value strictly outlives the other. At least, as far as the type system
is concerned.

In actual execution, Rust guarantees that `x` will be dropped before `y`.
This is because they are stored in a composite value (a tuple), and composite
values have their fields destroyed [in declaration order][drop-order].

So why do we care if the compiler considers `x` and `y` to live for the same
amount of time? Well, there's a special trick the compiler can do with equal
lifetimes: it can let us hold onto dangling pointers during destruction! But
we must be careful where we allow this, because any mistake can lead to a
use-after-free.

Consider the following simple program:

```rust
struct Inspector<'a>(&'a u8);

fn main() {
    let (days, inspector);
    days = Box::new(1);
    inspector = Inspector(&days);
}
```

This program is perfectly sound, and even compiles today! The fact that `days`
is dropped, and therefore freed, while `inspector` holds a pointer into it doesn't
matter because `inspector` will *also* be destroyed before any code gets a chance
to dereference that dangling pointer.

Just to make it clear that something special is happening here, this code
(which should behave identically at runtime) *doesn't* compile:

```rust,ignore
struct Inspector<'a>(&'a u8);

fn main() {
    let inspector;
    let days;
    days = Box::new(1);
    inspector = Inspector(&days);
}
```

```text
error: `days` does not live long enough
 --> src/main.rs:8:1
  |
7 |     inspector = Inspector(&days);
  |                            ---- borrow occurs here
8 | }
  | ^ `days` dropped here while still borrowed
  |
  = note: values in a scope are dropped in the opposite order they are created
```

The fact that `inspector` and `days` are stored in the same composite is letting
the compiler apply this special trick.

Now the *really* interesting part is that if we add a destructor to Inspector,
the program will *also* stop compiling!

```rust,ignore
struct Inspector<'a>(&'a u8);

impl<'a> Drop for Inspector<'a> {
    fn drop(&mut self) {
        println!("I was only {} days from retirement!", self.0);
    }
}

fn main() {
    let (days, inspector);
    days = Box::new(1);
    inspector = Inspector(&days);
    // When Inspector is dropped here, it will try to read free'd memory!
}
```

```text
error: `days` does not live long enough
  --> <anon>:15:1
   |
12 |     inspector = Inspector(&days);
   |                            ---- borrow occurs here
...
15 | }
   | ^ `days` dropped here while still borrowed
   |
   = note: values in a scope are dropped in the opposite order they are created

error: aborting due to previous error
```

Implementing Drop lets the Inspector execute arbitrary code during its
death, which means it can dereference any dangling pointer that it contains.
If we allowed this program to compile, it would perform a use-after-free.

This is the *sound generic drop* issue. We call it that because it only applies
to destructors of generic types. That is, if `Inspector` weren't generic, it
couldn't store any lifetime other than `'static', and dangling pointers would
never be a concern. The enforcement of sound generic drop is handled by the
*drop check*, which is more commonly known as *dropck*.

It turns out that getting dropck's design exactly right has been very difficult.
This is because, as we'll see, we don't want it to give up completely on generic
destructors. In particular: what if we knew `Inspector` *didn't* or even *couldn't*
dereference the pointer in its destructor? Wouldn't it be nice if that meant our
code compiled again?

Since Rust 1.0, and as of Rust 1.18, there have been two changes to how dropck
works due to soundness issues. At least one more is planned, as the latest
version was intended to be a temporary hack.

* [non-parametric dropck](https://github.com/rust-lang/rfcs/blob/master/text/1238-nonparametric-dropck.md)
* [dropck eyepatch](https://github.com/rust-lang/rfcs/blob/master/text/1327-dropck-param-eyepatch.md)

Old versions of this document were based on the original 1.0 design, which
was unsafe by default, and therefore expected unsafe Rust programmers to manually
opt out of it.

The good news is that, unlike the 1.0 design, the 1.18 design is *safe by default*:
you can completely ignore that dropck exists, and nothing bad can be done with
your types. But sometimes you will be able to leverage your knowledge of dropck
to make transiently dangling references work, and that's nice.

So here's all you should need to know about dropck these days:

* If a type isn't generic, then it cannot contain borrows that expire, and
  is therefore uninteresting.
* If a type is generic, and doesn't have a destructor, its generic arguments
  must must live *at least* as long as it.
* If a type is generic, and *does* have a destructor, its generic arguments
  must live *strictly* longer than it.

Or to put it another way: if you want to be able to store references to things
that live **exactly** as long as yourself, you can't have a destructor.




[drop-order]: https://github.com/rust-lang/rfcs/blob/master/text/1857-stabilize-drop-order.md
