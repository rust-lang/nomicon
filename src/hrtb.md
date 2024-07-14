# 상계 트레잇 제한 (Higher-Rank Trait Bounds, HRTBs)

러스트의 `Fn` 트레잇은 조금 마법 같습니다. 예를 들어, 다음의 코드를 쓸 수 있겠습니다:

```rust
struct Closure<F> {
    data: (u8, u16),
    func: F,
}

impl<F> Closure<F>
    where F: Fn(&(u8, u16)) -> &u8,
{
    fn call(&self) -> &u8 {
        (self.func)(&self.data)
    }
}

fn do_it(data: &(u8, u16)) -> &u8 { &data.0 }

fn main() {
    let clo = Closure { data: (0, 1), func: do_it };
    println!("{}", clo.call());
}
```

만약 우리가 순진하게 [수명 섹션][lt]에서 했던 대로 이 코드를 해독하려 하면, 좀 문제가 발생합니다:

<!-- ignore: desugared code -->
```rust,ignore
// 주의: `&'b data.0`와 `'x: {`은 올바른 문법이 아닙니다!
struct Closure<F> {
    data: (u8, u16),
    func: F,
}

impl<F> Closure<F>
    // where F: Fn(&'??? (u8, u16)) -> &'??? u8,
{
    fn call<'a>(&'a self) -> &'a u8 {
        (self.func)(&self.data)
    }
}

fn do_it<'b>(data: &'b (u8, u16)) -> &'b u8 { &'b data.0 }

fn main() {
    'x: {
        let clo = Closure { data: (0, 1), func: do_it };
        println!("{}", clo.call());
    }
}
```

세상에, 어떻게 우리가 `F`의 트레잇 제한에서 수명을 이야기할 수 있을까요? 

How on earth are we supposed to express the lifetimes on `F`'s trait bound? We
need to provide some lifetime there, but the lifetime we care about can't be
named until we enter the body of `call`! Also, that isn't some fixed lifetime;
`call` works with *any* lifetime `&self` happens to have at that point.

This job requires The Magic of Higher-Rank Trait Bounds (HRTBs). The way we
desugar this is as follows:

<!-- ignore: simplified code -->
```rust,ignore
where for<'a> F: Fn(&'a (u8, u16)) -> &'a u8,
```

Alternatively:

<!-- ignore: simplified code -->
```rust,ignore
where F: for<'a> Fn(&'a (u8, u16)) -> &'a u8,
```

(Where `Fn(a, b, c) -> d` is itself just sugar for the unstable *real* `Fn`
trait)

`for<'a>` can be read as "for all choices of `'a`", and basically produces an
*infinite list* of trait bounds that F must satisfy. Intense. There aren't many
places outside of the `Fn` traits where we encounter HRTBs, and even for
those we have a nice magic sugar for the common cases.

In summary, we can rewrite the original code more explicitly as:

```rust
struct Closure<F> {
    data: (u8, u16),
    func: F,
}

impl<F> Closure<F>
    where for<'a> F: Fn(&'a (u8, u16)) -> &'a u8,
{
    fn call(&self) -> &u8 {
        (self.func)(&self.data)
    }
}

fn do_it(data: &(u8, u16)) -> &u8 { &data.0 }

fn main() {
    let clo = Closure { data: (0, 1), func: do_it };
    println!("{}", clo.call());
}
```

[lt]: lifetimes.html
