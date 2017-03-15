# Lifetimes

Rust enforces these rules through *lifetimes*. Lifetimes are effectively
just names for scopes somewhere in the program. Each reference,
and anything that contains a reference, is tagged with a lifetime specifying
the scope it's valid for.

Within a function body, Rust generally doesn't let you explicitly name the
lifetimes involved. This is because it's generally not really necessary
to talk about lifetimes in a local context; Rust has all the information and
can work out everything as optimally as possible. Many anonymous scopes and
temporaries that you would otherwise have to write are often introduced to
make your code Just Work.

However once you cross the function boundary, you need to start talking about
lifetimes. Lifetimes are denoted with an apostrophe: `'a`, `'static`. To dip
our toes with lifetimes, we're going to pretend that we're actually allowed
to label scopes with lifetimes, and desugar the examples from the start of
this chapter.

Originally, our examples made use of *aggressive* sugar -- high fructose corn
syrup even -- around scopes and lifetimes, because writing everything out
explicitly is *extremely noisy*. All Rust code relies on aggressive inference
and elision of "obvious" things.

One particularly interesting piece of sugar is that each `let` statement implicitly
introduces a scope. For the most part, this doesn't really matter. However it
does matter for variables that refer to each other. As a simple example, let's
completely desugar this simple piece of Rust code:

```rust
let x = 0;
let y = &x;
let z = &y;
```

The borrow checker always tries to minimize the extent of a lifetime, so it will
likely desugar to the following:

```rust,ignore
// NOTE: `'a: {` and `&'b x` is not valid syntax!
'a: {
    let x: i32 = 0;
    'b: {
        // lifetime used is 'b because that's good enough.
        let y: &'b i32 = &'b x;
        'c: {
            // ditto on 'c
            let z: &'c &'b i32 = &'c y;
        }
    }
}
```

Wow. That's... awful. Let's all take a moment to thank Rust for making this easier.

Actually passing references to outer scopes will cause Rust to infer
a larger lifetime:

```rust
let x = 0;
let z;
let y = &x;
z = y;
```

```rust,ignore
'a: {
    let x: i32 = 0;
    'b: {
        let z: &'b i32;
        'c: {
            // Must use 'b here because this reference is
            // being passed to that scope.
            let y: &'b i32 = &'b x;
            z = y;
        }
    }
}
```



# Example: references that outlive referents

Alright, let's look at some of those examples from before:

```rust,ignore
fn as_str(data: &u32) -> &str {
    let s = format!("{}", data);
    &s
}
```

desugars to:

```rust,ignore
fn as_str<'a>(data: &'a u32) -> &'a str {
    'b: {
        let s = format!("{}", data);
        return &'a s;
    }
}
```

This signature of `as_str` takes a reference to a u32 with *some* lifetime, and
promises that it can produce a reference to a str that can live *just as long*.
Already we can see why this signature might be trouble. That basically implies
that we're going to find a str somewhere in the scope the reference
to the u32 originated in, or somewhere *even earlier*. That's a bit of a tall
order.

We then proceed to compute the string `s`, and return a reference to it. Since
the contract of our function says the reference must outlive `'a`, that's the
lifetime we infer for the reference. Unfortunately, `s` was defined in the
scope `'b`, so the only way this is sound is if `'b` contains `'a` -- which is
clearly false since `'a` must contain the function call itself. We have therefore
created a reference whose lifetime outlives its referent, which is *literally*
the first thing we said that references can't do. The compiler rightfully blows
up in our face.

To make this more clear, we can expand the example:

```rust,ignore
fn as_str<'a>(data: &'a u32) -> &'a str {
    'b: {
        let s = format!("{}", data);
        return &'a s
    }
}

fn main() {
    'c: {
        let x: u32 = 0;
        'd: {
            // An anonymous scope is introduced because the borrow does not
            // need to last for the whole scope x is valid for. The return
            // of as_str must find a str somewhere before this function
            // call. Obviously not happening.
            println!("{}", as_str::<'d>(&'d x));
        }
    }
}
```

Shoot!

Of course, the right way to write this function is as follows:

```rust
fn to_string(data: &u32) -> String {
    format!("{}", data)
}
```

We must produce an owned value inside the function to return it! The only way
we could have returned an `&'a str` would have been if it was in a field of the
`&'a u32`, which is obviously not the case.

(Actually we could have also just returned a string literal, which as a global
can be considered to reside at the bottom of the stack; though this limits
our implementation *just a bit*.)





# Example: aliasing a mutable reference

How about the other example:

```rust,ignore
let mut data = vec![1, 2, 3];
let x = &data[0];
data.push(4);
println!("{}", x);
```

```rust,ignore
'a: {
    let mut data: Vec<i32> = vec![1, 2, 3];
    'b: {
        // 'b is as big as we need this borrow to be
        // (just need to get to `println!`)
        let x: &'b i32 = Index::index::<'b>(&'b data, 0);
        'c: {
            // Temporary scope because we don't need the
            // &mut to last any longer.
            Vec::push(&'c mut data, 4);
        }
        println!("{}", x);
    }
}
```

The problem here is a bit more subtle and interesting. We want Rust to
reject this program for the following reason: We have a live shared reference `x`
to a descendant of `data` when we try to take a mutable reference to `data`
to `push`. This would create an aliased mutable reference, which would
violate the *second* rule of references.

However this is *not at all* how Rust reasons that this program is bad. Rust
doesn't understand that `x` is a reference to a subpath of `data`. It doesn't
understand Vec at all. What it *does* see is that `x` has to live for `'b` to
be printed. The signature of `Index::index` subsequently demands that the
reference we take to `data` has to survive for `'b`. When we try to call `push`,
it then sees us try to make an `&'c mut data`. Rust knows that `'c` is contained
within `'b`, and rejects our program because the `&'b data` must still be live!

Here we see that the lifetime system is much more coarse than the reference
semantics we're actually interested in preserving. For the most part, *that's
totally ok*, because it keeps us from spending all day explaining our program
to the compiler. However it does mean that several programs that are totally
correct with respect to Rust's *true* semantics are rejected because lifetimes
are too dumb.

# Unifying Lifetimes

XXX: is unification a good term to be using?

Often the Rust compiler must prove that two references with different lifetimes
are compatible. We call this *unification* of lifetimes.

Consider the following program:

```rust,ignore
fn main() {
    let s1 = String::from("short");
    {
        let s2 = String::from("a long long long string");
        println!("{}", shortest(&s1, &s2));
    }
}

fn shortest<'a>(x: &'a str, y: &'a str) -> &'a str {
    if x.len() < y.len() {
        return x;
    } else {
        return y;
    }
}
```

The idea is that `shortest()` returns a reference to the shorter of the two
strings referenced by its arguments, but *without* allocating a new string.

The two references passed at the call-site of `shortest()` have different lifetimes
since their referents are defined in different scopes. The
string value `s1` is live longer than the string value `s2`, yet the signature
of `shortest()` requires that these two references have the same lifetime.
Furthermore, the returned string reference must also share this same lifetime too.
So how does the compiler make this so?

Essentially, the signature of `shortest()` asks the compiler to find a single
lifetime in the *caller* under which the three references annotated `'a` remain
valid. Rust will try to *convert* each of these three reference
lifetimes in `main()` into *one* *unified* lifetime which is shorter than, or equally as
long as, each of the references in isolation. A reference `&'o T` can be
converted to to `&'p T` if (and only if) it can be proven that `'o` lives as
long as (or longer than) `'p`. In our example the references `'&s1`, `&s2` and
the returned reference can all be shown to be valid as long as the implicit
scope created by the `let s2` binding (see above for information on implicit
scopes introduced by `let` bindings). So in this case, we can prove that the
lifetimes of `&s1`, `&s2` and the returned reference can be unified, and as a
result he compiler accepts the program.

If, on the other hand, the compiler cannot find such a lifetime, then the
lifetime constraints described by the program are inconsistent, and the
compiler will reject the program. For example:

```rust,ignore
XXX
```
