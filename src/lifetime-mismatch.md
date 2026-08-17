# Limits of Lifetimes

Given the following code:

```rust,compile_fail
#[derive(Debug)]
struct Foo;

impl Foo {
    fn mutate_and_share(&mut self) -> &Self { &*self }
    fn share(&self) {}
}

fn main() {
    let mut foo = Foo;
    let loan = foo.mutate_and_share();
    foo.share();
    println!("{:?}", loan);
}
```

One might expect it to compile. We call `mutate_and_share`, which mutably
borrows `foo` temporarily, but then returns only a shared reference. Therefore
we would expect `foo.share()` to succeed as `foo` shouldn't be mutably borrowed.

However when we try to compile it:

```text
error[E0502]: cannot borrow `foo` as immutable because it is also borrowed as mutable
  --> src/main.rs:12:5
   |
11 |     let loan = foo.mutate_and_share();
   |                --- mutable borrow occurs here
12 |     foo.share();
   |     ^^^ immutable borrow occurs here
13 |     println!("{:?}", loan);
```

What happened? Well, we got the exact same reasoning as we did for
[Example 2 in the previous section][ex2]. We desugar the program and we get
the following:

<!-- ignore: desugared code -->
```rust,ignore
struct Foo;

impl Foo {
    fn mutate_and_share<'a>(&'a mut self) -> &'a Self { &'a *self }
    fn share<'a>(&'a self) {}
}

fn main() {
    'b: {
        let mut foo: Foo = Foo;
        'c: {
            let loan: &'c Foo = Foo::mutate_and_share::<'c>(&'c mut foo);
            'd: {
                Foo::share::<'d>(&'d foo);
            }
            println!("{:?}", loan);
        }
    }
}
```

The lifetime system is forced to extend the `&mut foo` to have lifetime `'c`,
due to the lifetime of `loan` and `mutate_and_share`'s signature. Then when we
try to call `share`, it sees we're trying to alias that `&'c mut foo` and
blows up in our face!

This program is clearly correct according to the reference semantics we actually
care about, but the lifetime system is too coarse-grained to handle that.

[ex2]: lifetimes.html#example-aliasing-a-mutable-reference
