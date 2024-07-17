# 부분타입 다형성과 변성(變性, Variance)

러스트는 빌림과 소유권 사이의 관계를 추적하기 위해 수명을 사용합니다. 하지만 수명의 순진한 구현은 너무 제한적이거나, 아니면 미정의 동작을 허용하게 됩니다.

수명을 유연하게 사용하면서도 수명의 오용을 방지하기 위해서, 러스트는 **부분타입 다형성** 과 **변성(變性, Variance)** 을 사용합니다.

예제와 함께 시작해 보죠.

```rust
// 주의: debug는 수명이 *같은* 두 개의 매개변수를 기대합니다.
fn debug<'a>(a: &'a str, b: &'a str) {
    println!("a = {a:?} b = {b:?}");
}

fn main() {
    let hello: &'static str = "hello";
    {
        let world = String::from("world");
        let world = &world; // 'world 는 'static 보다 짧은 수명입니다
        debug(hello, world);
    }
}
```

보수적인 수명의 구현에서는 `hello`와 `world`는 다른 수명을 가지고 있으므로, 우리는 다음과 같은 오류를 볼지도 모릅니다:

```text
error[E0308]: mismatched types
 --> src/main.rs:10:16
   |
10 |         debug(hello, world);
   |                      ^
   |                      |
   |                      expected `&'static str`, found struct `&'world str`
```

이것은 뭔가 부적절할 것입니다. 이 경우에 우리가 원하는 것은 *최소한* `'world`만큼만 사는 타입은 모두 받는 것입니다. 우리의 수명들에 부분타입 다형성을 이용해 봅시다.

## 부분타입 다형성

부분타입 다형성은 한 타입이 다른 타입 대신에 쓰일 수 있다는 개념입니다.

`Sub`이라는 타입이 `Super`라는 타입의 부분타입이라고 해 봅시다 (우리는 이 단원에서 이것을 `Sub <: Super`라고 표현하는 표기법을 사용하겠습니다).

이것이 우리에게 나타내는 것은 `Super`가 정의하는 *요구사항들*의 집합을 `Sub`이 완벽하게 충족한다는 것입니다. 그 다음 `Sub`은 더 많은 요구사항을 가질 수 있겠죠.

이제, 부분타입 다형성을 수명에 쓰기 위해, 우리는 수명의 요구사항을 정의해야 합니다:

> `'a`는 코드 구역을 정의한다.

이제 수명을 위한 요구사항을 만들었으니, 우리는 수명들이 서로 어떻게 관련이 있는지를 정의할 수 있습니다:

> `'long`이 정의하는 코드 구역이 `'short`가 정의하는 구역을 **완전히 포함할 때**, 그리고 오직 그 경우에만 `'long <: 'short`이다.

`'long`은 `'short`가 정의한 구역보다 더 넓은 코드 구역을 정의할 수 있지만, 그래도 우리의 정의에 어긋나지 않습니다.

우리가 이 단원의 나머지를 통해서 보겠지만, 부분타입 다형성은 이것보다는 훨씬 복잡하고 세밀하지만, 이 간단한 규칙은 직관상 99%로 아주 좋습니다.
그리고 만약 불안전한 코드를 작성하지 않는다면, 컴파일러가 당신을 위해 온갖 특수한 경우를 다 처리해 줄 겁니다. 하지만 이것은 러스토노미콘이죠. 우리는 불안전한 코드를 작성할 것이니,
우리는 이것이 실제로 어떻게 동작하는지, 그리고 우리가 이것을 어떻게 가지고 놀 수 있을지를 이해해야 합니다.

위의 예제로 돌아오면, 우리는 `'static <: 'world`라고 말할 수 있습니다. 지금으로써는, 수명의 부분타입 관계가 레퍼런스에도 그대로 전달된다는 것을 일단은 받아들입시다 (더 자세한 건 [변성](#)에서 다룹니다).
예를 들어, `&'static str`은 `&'world str`의 부분타입이므로, 우리는 `&'static str`을 `&'world str`로 "격하시킬" 수 있습니다. 이렇게 하면, 위의 예제는 컴파일될 겁니다:

```rust
fn debug<'a>(a: &'a str, b: &'a str) {
    println!("a = {a:?} b = {b:?}");
}

fn main() {
    let hello: &'static str = "hello";
    {
        let world = String::from("world");
        let world = &world; // 'world 는 'static 보다 짧은 수명입니다.
        debug(hello, world); // hello 는 조용히 `&'static str`을 `&'world str`로 격하시킵니다.
    }
}
```

## 변성(變性, Variance)

위에서 우리는 `'static <: 'b`가 `&'static T <: &'b T`를 함의한다는 것을 대충 넘어갔었습니다. 이것은 *변성*이라고 알려진 속성을 사용한 것인데요. 이 예제처럼 간단하지만은 않습니다. 이것을 이해하기 위해, 
이 예제를 조금 확장해 보죠:

```rust,compile_fail,E0597
fn assign<T>(input: &mut T, val: T) {
    *input = val;
}

fn main() {
    let mut hello: &'static str = "hello";
    {
        let world = String::from("world");
        assign(&mut hello, &world);
    }
    println!("{hello}"); // 해제 후 사용 😿
}
```

`assign`에서 우리는 `hello` 레퍼런스를 `world`를 향해 가리키도록 합니다. 하지만 그 다음 `world`는, 나중에 `hello`가 `println!`에서 사용되기 전에, 구역 밖으로 벗어나고 맙니다.

이것은 전형적인 "해제 후 사용" 버그입니다!

우리의 본능은 먼저 `assign`의 구현을 나무랄 수도 있겠지만, 여기에는 잘못된 것이 없습니다. 우리가 `T` 타입의 값을 `T` 타입에 할당하는 것이 그렇게 무리는 아닐 겁니다.

문제는 우리가 `&mut &'static str`과 `&mut &'b str`이 서로 호환되는지를 짐작할 수 없다는 점입니다. 이것이 의미하는 것은 `&mut &'static str`이 `&mut &'b str`의 부분타입이 될 수 **없다는** 말입니다, 
비록 `'static`이 `'b`의 부분타입이라고 해도요.

변성은 제네릭 매개변수를 통한 부분타입들간의 관계를 정의하기 위해 러스트가 빌린 개념입니다.

> 주의: 편의를 위해 우리는 제네릭 타입을 `F<T>`로 정의하여 `T`에 대해 쉽게 말할 것입니다. 이것이 문맥에서 잘 드러나길 바랍니다.

타입 `F`의 *변성* 그 입력들의 부분타입 다형성이 출력들의 부분타입 다형성에 어떻게 영향을 주느냐 하는 것입니다. 러스트에서는 세 가지 종류의 변성이 있습니다. 두 타입 `Sub`과 `Super`가 있고, `Sub`이 `Super`의 부분타입일 때:

* `F<Sub>`이 `F<Super>`의 부분타입일 경우 `F`는 **공변(共變)합니다** (부분타입 특성이 전달됩니다)
* `F<Super>`가 `F<Sub>`의 부분타입일 경우 `F`는 **반변(反變)합니다** (부분타입 특성이 "뒤집힙니다")
* 그 외에는 `F`는 **무변(無變)합니다** (부분타입 관계가 존재하지 않습니다)

우리가 위의 예제에서 기억한다면, `'a <: 'b`일 경우 `&'a T`를 `&'b T`의 부분타입으로 다뤄도 되었으니, `&'a T`는 `'a`에 대해서 *공변하는* 것이군요.

또한, 우리는 `&mut &'a U`를 `&mut &'b U`의 부분타입으로 다루면 안된다는 것을 보았으니, `&mut T`는 `T`에 대해서 *무변하다고* 말할 수 있겠습니다.

여기 다른 제네릭 타입들과 그들의 변성에 대한 표입니다:

|                 |     'a    |         T         |     U     |
|-----------------|:---------:|:-----------------:|:---------:|
| `&'a T `        | 공변       | 공변               |           |
| `&'a mut T`     | 공변       | 무변               |           |
| `Box<T>`        |           | 공변               |           |
| `Vec<T>`        |           | 공변               |           |
| `UnsafeCell<T>` |           | 무변               |           |
| `Cell<T>`       |           | 무변               |           |
| `fn(T) -> U`    |           | **반**변           | 공변       |
| `*const T`      |           | 공변               |           |
| `*mut T`        |           | 무변               |           |

이 중의 몇 가지는 다른 것들과의 관계로 설명할 수 있습니다:

* `Vec<T>`와 다른 모든 소유하는 포인터들과 컬렉션들은 `Box<T>`와 같은 논리를 따릅니다
* `Cell<T>`와 다른 모든 내부 가변성이 있는 타입들은 `UnsafeCell<T>`와 같은 논리를 따릅니다
* `UnsafeCell<T>`는 내부 가변성이 있으므로 `&mut T`와 같은 변성을 가지게 됩니다
* `*const T`는 `&T`와 같은 논리를 따릅니다
* `*mut T`는 `&mut T`(또는 `UnsafeCell<T>`)와 같은 논리를 따릅니다

더 많은 타입에 대해서는 참조서의 ["Variance" 섹션을][variance-table] 보세요.

[variance-table]: ../reference/subtyping.html#variance

> NOTE: the *only* source of contravariance in the language is the arguments to
> a function, which is why it really doesn't come up much in practice. Invoking
> contravariance involves higher-order programming with function pointers that
> take references with specific lifetimes (as opposed to the usual "any lifetime",
> which gets into higher rank lifetimes, which work independently of subtyping).

Now that we have some more formal understanding of variance,
let's go through some more examples in more detail.

```rust,compile_fail,E0597
fn assign<T>(input: &mut T, val: T) {
    *input = val;
}

fn main() {
    let mut hello: &'static str = "hello";
    {
        let world = String::from("world");
        assign(&mut hello, &world);
    }
    println!("{hello}");
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

Meanwhile, in the caller we pass in `&mut &'static str` and `&'world str`.

Because `&mut T` is invariant over `T`, the compiler concludes it can't apply any subtyping
to the first argument, and so `T` must be exactly `&'static str`.

This is counter to the `&T` case:

```rust
fn debug<T: std::fmt::Debug>(a: T, b: T) {
    println!("a = {a:?} b = {b:?}");
}
```

where similarly `a` and `b` must have the same type `T`.
But since `&'a T` *is* covariant over `'a`, we are allowed to perform subtyping.
So the compiler decides that `&'static str` can become `&'b str` if and only if
`&'static str` is a subtype of `&'b str`, which will hold if `'static <: 'b`.
This is true, so the compiler is happy to continue compiling this code.

As it turns out, the argument for why it's ok for Box (and Vec, HashMap, etc.) to be covariant is pretty similar to the argument for why it's ok for lifetimes to be covariant: as soon as you try to stuff them in something like a mutable reference, they inherit invariance and you're prevented from doing anything bad.

However Box makes it easier to focus on the by-value aspect of references that we partially glossed over.

Unlike a lot of languages which allow values to be freely aliased at all times, Rust has a very strict rule: if you're allowed to mutate or move a value, you are guaranteed to be the only one with access to it.

Consider the following code:

```rust,ignore
let hello: Box<&'static str> = Box::new("hello");

let mut world: Box<&'b str>;
world = hello;
```

There is no problem at all with the fact that we have forgotten that `hello` was alive for `'static`,
because as soon as we moved `hello` to a variable that only knew it was alive for `'b`,
**we destroyed the only thing in the universe that remembered it lived for longer**!

Only one thing left to explain: function pointers.

To see why `fn(T) -> U` should be covariant over `U`, consider the following signature:

<!-- ignore: simplified code -->
```rust,ignore
fn get_str() -> &'a str;
```

This function claims to produce a `str` bound by some lifetime `'a`. As such, it is perfectly valid to
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

Let's see this in practice

```rust,compile_fail
# use std::cell::RefCell;
thread_local! {
    pub static StaticVecs: RefCell<Vec<&'static str>> = RefCell::new(Vec::new());
}

/// saves the input given into a thread local `Vec<&'static str>`
fn store(input: &'static str) {
    StaticVecs.with_borrow_mut(|v| v.push(input));
}

/// Calls the function with it's input (must have the same lifetime!)
fn demo<'a>(input: &'a str, f: fn(&'a str)) {
    f(input);
}

fn main() {
    demo("hello", store); // "hello" is 'static. Can call `store` fine

    {
        let smuggle = String::from("smuggle");

        // `&smuggle` is not static. If we were to call `store` with `&smuggle`,
        // we would have pushed an invalid lifetime into the `StaticVecs`.
        // Therefore, `fn(&'static str)` cannot be a subtype of `fn(&'a str)`
        demo(&smuggle, store);
    }

    // use after free 😿
    StaticVecs.with_borrow(|v| println!("{v:?}"));
}
```

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
