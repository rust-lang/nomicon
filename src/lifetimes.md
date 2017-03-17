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

# Inter-procedural Borrow Checking of Function Arguments

Consider the following program:

```rust,ignore
fn main() {
    let s1 = String::from("short");
    let s2 = String::from("a long long long string");
    print_shortest(&s1, &s2);
}

fn shortest<'k>(x: &'k str, y: &'k str) {
    if x.len() < y.len() {
        println!("{}", x);
    } else {
        println!("{}", y);
    }
}
```

`print_shortest` simply prints the shorter of its two pass-by-reference
string arguments. In Rust, each let binding has its own scope. Let's make the
scopes introduced to `main` explicit:

```rust,ignore
fn main() {
    's1 {
        let s1 = String::from("short");
        's2 {
            let s2 = String::from("a long long long string");
            print_shortest(&s1, &s2);
        }
}
```

And now let's explicitly mark the lifetimes of each reference's referent too:

```rust,ignore
fn main() {
    's1 {
        let s1 = String::from("short");
        's2 {
            let s2 = String::from("a long long long string");
            print_shortest(&'s1 s1, &'s2 s2);
        }
}
```

Now we see that the references passed as arguments to `print_shortest`
actually have different lifetimes (and thus a different type!) since the values
they refer to were introduced in different scopes. At the call site of
`print_shortest` the compiler must now check that the lifetimes in the
*caller* (`main`) are consistent with the lifetimes in the signature of the
*callee* (`print_shortest`).

The signature of `print_shortest` simply requires that both of it's arguments
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

> A function argument of type `&'p T` can be coerced with an argument of type
> `&'q T` if the lifetime of `&'p T` is equal or longer than `&'q T`.

At our call site, the type of the arguments are `&'s1 str` and `&'s2 str`, and
we know that  a `&'s1 str' outlives an `&'s2 str`, so we can substitute `&'s1
s1` with `&'s2 s2`. After this both arguments are of lifetime `&'s2` and the
call-site is consistent with the signature of `print_shortest`.

[More formally, the basis for the above rule is in *type variance*. Under this
model, you would consider a longer lifetime a sub-type of a shorter lifetime,
and for function arguments to be *co-variant*. However, an understanding of
variance isn't strictly required to understand the Rust borrow checker. We've
tried here to instead to explain using intuitive terminlolgy.]

# Inter-procedural Borrow Checking of Function Return Values

Now consider a slight variation of this example:

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
for efficiency, avoiding the need to re-allocate a new string to pass back to
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

Again, at the call-site of `shortest` the comipiler needs to check the
consistency of the arguments in the caller with the signature of the callee.
The signature of `shortest` fisrt says that the two reference arguments have
the same lifetime, which can be prove ok in the same way as before, thus giving
us:

```rust,ignore
res: &'res = shortest(&'s2 s1, &'s2 s2);
```

But we now have the additional reference to check. We must now prove that the
returned reference can have the same lifetime as the arguments of lifetime
'&'s2'. This brings us to a second rule:

> The return value of a function `&'r T` can be converted to an argument `&'s T`
> if the lifetime of `&'r T` is equal or shorter than `&'s T`.

To make our program compile, we would have to subsitute `res: &'res` for `res:
&'s2`, but we can't since `&'res` in fact out-lives `&'s2`. This program is in
fact inconsistent and the compiler rightfully rejects the program because we
try make a reference (`&'res`) which outlives one of the values it refer to
(`&'s2`).

[Formally, function return values are said to be *contravariant*, the opposite
of *covariant*.]

How can we fix this porgram? Well if you were to swap the `let s2 = ...` with
the `res = ...` line, you would have:

```rust,ignore
fn main() {
    let s1 = String::from("short");
    let s2 = String::from("a long long long string");
    let res;
    res = shortest(&s1, &s2);
    println!("{}", res);
}
```

Which de-sugars to:

```rust,ignore
fn main() {
    's1 {
        let s1 = String::from("short");
        's2 {
            let s2 = String::from("a long long long string");
            'res {
                let res: &'res str;
                res: &'res str = shortest(&'s1 s1, &'s2 s2);
                println!("{}", res);
            }
        }
    }
}
```

Then at the call-site of `shortest`:
 * `&'s1 s1` outlives `&'s2 s2`, so we can replace the first argument with `&'s2 s1`.
 * `&'res str` lives shorter than `'&s2`, so the return value lifetime can become `res: &'s2 str`

Leaving us with:

```rust,ignore
res: &'s2 str = shortest(&'s2 s1, &'s2 s2);
```

Which matches the signature of `shortest` and thus this compiles.
Intuitively, the return reference can't point to a freed value as the values
live strictly longer than the return reference.
