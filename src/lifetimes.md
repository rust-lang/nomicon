# Lifetimes

Rust ensures memory safety through *lifetimes*. Lifetimes are effectively
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
our toes into lifetimes, we're going to pretend that we're actually allowed
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



# Example: References that Outlive Referents

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





# Example: Aliasing a Mutable Reference

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

# Inter-procedural Borrow Checking

Earlier we discussed lifetime constraints within a single function. Now let's
talk about the constraints *between* functions. Consider the following program:

```rust
fn main() {
    let s1 = String::from("short");
    let s2 = String::from("a long long long string");
    print_shortest(&s1, &s2);
}

fn print_shortest<'k>(x: &'k str, y: &'k str) {
    if x.len() < y.len() {
        println!("{}", x);
    } else {
        println!("{}", y);
    }
}
```

`print_shortest` prints the shorter of its two pass-by-reference
string arguments. Let's first de-sugar to make the reference lifetimes of
`main` explicit:

```rust,ignore
fn main() {
    's1 {
        let s1 = String::from("short");
        's2 {
            let s2 = String::from("a long long long string");
            print_shortest(&'s1 s1, &'s2 s2);
        }
    }
}
```

(for brevity, we don't show the third implicit scope that would be introduced
to limit to lifetimes of the borrows in the call to `print_shortest`)

Now we see that the references passed as arguments to `print_shortest`
actually have different lifetimes (and thus a different type!) since the values
they refer to were introduced in different scopes. At the call site of
`print_shortest` the compiler must now check that the lifetimes in the
*caller* (`main`) are consistent with the lifetimes in the signature of the
*callee* (`print_shortest`).

The signature of `print_shortest` requires that both of it's arguments
have the same lifetime (because both arguments are marked with the same
lifetime identifier in the signature). If in `main` we had done:

```rust,ignore
print_shortest(&s1, &s1);
```

Then consistency is trivially proven, since both arguments would have
the same lifetime `&'s1` at the call-site. However, for our example, the
arguments have different lifetimes. We don't want Rust to reject the program
because it actually is safe. Instead the compiler uses some rules for
converting between lifetimes whilst retaining referential safety. The first
such rule is as follows:

> A reference can always be shrunk to one of a shorter lifetime. In other
> words, `&'a T` can be implicitly converted to `&'b T` as long as `'a`
outlives `'b.`

At our call site, the type of the arguments are `&'s1 str` and `&'s2 str`, and
we know that a `&'s1 str` outlives an `&'s2 str`, so we can shrink `&'s1
s1` to `&'s2 s1`. After this both arguments are of lifetime `&'s2` and the
call-site is consistent with the signature of `print_shortest`.

## Inter-procedural Borrow Checking of Function Return Values

Now consider a slight variation of the previous example:

```rust,ignore
fn main() {
    let s1 = String::from("short");
    let res;
    let s2 = String::from("a long long long string");
    res = shortest(&s1, &s2);
    println!("{}", res);
}

fn shortest<'k>(x: &'k str, y: &'k str) -> &'k str {
    if x.len() < y.len() {
        return x;
    } else {
        return y;
    }
}
```

`print_shortest` has been renamed to `shortest`, which instead of printing,
now returns the shorter of the two strings. It does this using only references
for efficiency, avoiding the need to allocate a new string to pass back to
`main`. The responsibility of printing the result has been shifted to `main`.

Let's again de-sugar `main` by adding explicit scopes and lifetimes:

```rust,ignore
fn main() {
    's1 {
        let s1 = String::from("short");
        'res {
            let res: &'res str;
            's2 {
                let s2 = String::from("a long long long string");
                res: &'res: str = shortest(&'s1 s1, &'s2 s2);
                println!("{}", res);
            }
        }
    }
}
```

Again, at the call-site of `shortest` the compiler needs to check the consistency
of the arguments in the caller with the signature of the callee. The signature
of shortest says that all three references must have the same lifetime `'k`, so
we have to find a lifetime `'k` such that:

 * `&s1`: `&'s1 str` can be converted to the first argument `&'k str`
 * `&s2`: `&'s2 str` can be converted to the second argument `&'k str`
 * The return value `&'k str` can be converted to res: `&'res str`

This leads to three requirements:

 * `'s1` must outlive `'k`
 * `'s2` must outlive `'k`
 * `'k` must outlive `'res`

So by transitivity (`'s2` outlives `'k` outlives `'res`), we also require `'s2`
to outlive `'res`, which is not the case. The borrow checker rightfully
rejects our program because we are making a reference (`res`) which outlives
one of the values it may refer to (`s2`).

The program is fixed by swapping the definition order of `res` and `s2`. Then
`res` lives longer than both `s1` and `s2`.
