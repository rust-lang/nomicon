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

세상에, 어떻게 우리가 `F`의 트레잇 제한에서 수명을 이야기할 수 있을까요? 이곳에 어떤 수명을 제공해야 하지만, 우리가 신경쓰는 수명은 우리가 `call`의 본문에 들어가기 전에는 이름을 붙일 수가 없습니다! 또한, 이 수명은 어떤 정해진 수명이 아닙니다; `call`은 그 시점에 `&self`가 가지고 있게 되는 *아무* 수명과 작업이 가능합니다.

이 작업은 상계 트레잇 제한(Higher-Rank Trait Bounds, HRTBs)의 마법이 필요합니다. 우리가 일전의 코드를 다시 해독하면 이와 같습니다:

<!-- ignore: simplified code -->
```rust,ignore
where for<'a> F: Fn(&'a (u8, u16)) -> &'a u8,
```

또는 이렇게요:

<!-- ignore: simplified code -->
```rust,ignore
where F: for<'a> Fn(&'a (u8, u16)) -> &'a u8,
```

(여기서 `Fn(a, b, c) -> d` 자체는 불안정한 *실제* `Fn` 트레잇을 위한 문법 설탕입니다)



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
