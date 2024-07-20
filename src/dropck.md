# 해제 검사

우리는 수명이 간단한 규칙으로 우리가 달랑거리는 레퍼런스를 절대 읽지 않도록 보장하는 것을 봤습니다. 하지만 이 동안 우리는 한 수명이 다른 수명보다 *오래 산다고* 할 때, 두 수명이 같은 경우도 포함한 경우만 말했습니다. 즉, 우리가 `'a: 'b`라고 할 때, `'a`가 *딱* `'b`만큼만 살아도 괜찮았다는 말입니다. 처음 보면, 이것은 의미없는 구분 같습니다. 같은 시점에 동시에 해제되는 값은 없잖아요, 그렇죠? 그래서 우리는 이런 `let` 문장들을:

<!-- ignore: simplified code -->
```rust,ignore
let x;
let y;
```

이렇게 해독할 수 있었죠:

<!-- ignore: desugared code -->
```rust,ignore
{
    let x;
    {
        let y;
    }
}
```

이렇게 코드 구역을 이용해서 해독할 수 없는, 좀더 복잡한 상황들도 있지만, 순서는 여전히 정의되어 있습니다 - 변수들은 그 정의 순서의 반대로 해제되고, 구조체와 튜플의 필드들은 그 정의 순서대로 해제됩니다. [RFC 1857][rfc1857]에 해제 순서에 대한 좀더 자세한 내용이 있습니다.

이렇게 해 봅시다:

<!-- ignore: simplified code -->
```rust,ignore
let tuple = (vec![], vec![]);
```

왼쪽 벡터가 먼저 해제됩니다. 하지만 이것이 대여 검사기의 눈에 오른쪽 벡터가 왼쪽 벡터보다 엄밀하게 더 오래 산다는 것을 뜻할까요? 이 질문의 답은 *아니라는 겁니다*. 대여 검사기는 튜플의 필드들을 분리해서 추적할 수 있지만, 벡터 원소들에 대해서는 무엇이 무엇보다 오래 사는지 결정할 수 없을 겁니다, 왜냐하면 그 원소들은 대여 검사기가 이해하지 못하는, 순수한 라이브러리 코드로 해제되기 때문입니다.

그래서 우리는 이것을 왜 신경쓸까요? 우리가 신경쓰는 이유는 만약 타입 시스템이 주의하지 않으면, 우발적으로 달랑거리는 포인터를 만들 수도 있기 때문입니다. 다음의 간단한 프로그램을 생각해 보세요:

```rust
struct Inspector<'a>(&'a u8);

struct World<'a> {
    inspector: Option<Inspector<'a>>,
    days: Box<u8>,
}

fn main() {
    let mut world = World {
        inspector: None,
        days: Box::new(1),
    };
    world.inspector = Some(Inspector(&world.days));
}
```

이 프로그램은 완벽히 건전하고 오늘날 컴파일됩니다. `days`가 `inspector`보다 엄밀하게 더 오래 살지는 않는다는 사실이 중요하지 않습니다. `inspector`가 살아있는 동안, `days`도 그럴 테니까요.

그러나 우리가 소멸자를 추가하면, 이 프로그램은 더 이상 컴파일되지 않습니다!

```rust,compile_fail
struct Inspector<'a>(&'a u8);

impl<'a> Drop for Inspector<'a> {
    fn drop(&mut self) {
        println!("I was only {} days from retirement!", self.0);
    }
}

struct World<'a> {
    inspector: Option<Inspector<'a>>,
    days: Box<u8>,
}

fn main() {
    let mut world = World {
        inspector: None,
        days: Box::new(1),
    };
    world.inspector = Some(Inspector(&world.days));
    // `days`가 먼저 해제되게 되었다고 가정해 봅시다.
    // 그럼 `Inspector`가 해제될 때, 이미 해제된 메모리를 읽으려고 할 겁니다!
}
```

```text
error[E0597]: `world.days` does not live long enough
  --> src/main.rs:19:38
   |
19 |     world.inspector = Some(Inspector(&world.days));
   |                                      ^^^^^^^^^^^ borrowed value does not live long enough
...
22 | }
   | -
   | |
   | `world.days` dropped here while still borrowed
   | borrow might be used here, when `world` is dropped and runs the destructor for type `World<'_>`
```

필드의 순서를 바꾸거나 구조체 대신 튜플을 쓴다 해도, 컴파일되지는 않을 겁니다.

`Drop`을 구현하는 것은 `Inspector`가 죽는 동안 어떤 임의의 코드를 실행하게 해 줍니다. 이것이 의미하는 것은 `Inspector`가 사는 동안 살기로 되어있는 타입들이 실제로는 먼저 해제되었는지 관찰할 수도 있다는 것입니다.

흥미롭게도, 제네릭 타입들만 이런 것에 대해서 걱정해야 합니다. 제네릭이 아니라면, 그들이 사용할 수 있는 수명은 `'static`밖에 없는데, 이것은 정말로 *영원히* 살 것이기 때문입니다. 이것이 바로 이런 문제가 *건전한 제네릭 해제*라고 불리는 이유입니다. 건전한 제네릭 해제는 *해제 검사기*에 의해 강제됩니다. 이 글을 쓰는 시점에서, 해제 검사기(dropck라고도 부릅니다)가 어떻게 타입을 검증하는지에 대한 자세한 내용은 전혀 정해져 있지 않습니다. 하지만 큰 틀에서의 규칙은 우리가 이 섹션 내내 집중해온 엄밀함입니다:

**제네릭 타입이 건전하게 `Drop`을 구현하려면, 그 제네릭 매개변수들이 엄밀하게 더 오래 살아야 합니다.**

이 규칙을 지키는 것은 (보통은) 대여 검사기를 만족시키기 위해서 필수적입니다; 이 규칙을 지키는 것은 건전하기에는 충분하지만 필수적이지는 않습니다. 즉, 당신의 타입이 이 규칙을 지킨다면 해제되기에 확실히 건전하다는 말입니다.

The reason that it is not always necessary to satisfy the above rule
is that some Drop implementations will not access borrowed data even
though their type gives them the capability for such access, or because we know
the specific drop order and the borrowed data is still fine even if the borrow
checker doesn't know that.

For example, this variant of the above `Inspector` example will never
access borrowed data:

```rust,compile_fail
struct Inspector<'a>(&'a u8, &'static str);

impl<'a> Drop for Inspector<'a> {
    fn drop(&mut self) {
        println!("Inspector(_, {}) knows when *not* to inspect.", self.1);
    }
}

struct World<'a> {
    inspector: Option<Inspector<'a>>,
    days: Box<u8>,
}

fn main() {
    let mut world = World {
        inspector: None,
        days: Box::new(1),
    };
    world.inspector = Some(Inspector(&world.days, "gadget"));
    // Let's say `days` happens to get dropped first.
    // Even when Inspector is dropped, its destructor will not access the
    // borrowed `days`.
}
```

Likewise, this variant will also never access borrowed data:

```rust,compile_fail
struct Inspector<T>(T, &'static str);

impl<T> Drop for Inspector<T> {
    fn drop(&mut self) {
        println!("Inspector(_, {}) knows when *not* to inspect.", self.1);
    }
}

struct World<T> {
    inspector: Option<Inspector<T>>,
    days: Box<u8>,
}

fn main() {
    let mut world = World {
        inspector: None,
        days: Box::new(1),
    };
    world.inspector = Some(Inspector(&world.days, "gadget"));
    // Let's say `days` happens to get dropped first.
    // Even when Inspector is dropped, its destructor will not access the
    // borrowed `days`.
}
```

However, _both_ of the above variants are rejected by the borrow
checker during the analysis of `fn main`, saying that `days` does not
live long enough.

The reason is that the borrow checking analysis of `main` does not
know about the internals of each `Inspector`'s `Drop` implementation. As
far as the borrow checker knows while it is analyzing `main`, the body
of an inspector's destructor might access that borrowed data.

Therefore, the drop checker forces all borrowed data in a value to
strictly outlive that value.

## An Escape Hatch

The precise rules that govern drop checking may be less restrictive in
the future.

The current analysis is deliberately conservative and trivial; it forces all
borrowed data in a value to outlive that value, which is certainly sound.

Future versions of the language may make the analysis more precise, to
reduce the number of cases where sound code is rejected as unsafe.
This would help address cases such as the two `Inspector`s above that
know not to inspect during destruction.

In the meantime, there is an unstable attribute that one can use to
assert (unsafely) that a generic type's destructor is _guaranteed_ to
not access any expired data, even if its type gives it the capability
to do so.

That attribute is called `may_dangle` and was introduced in [RFC 1327][rfc1327].
To deploy it on the `Inspector` from above, we would write:

```rust
#![feature(dropck_eyepatch)]

struct Inspector<'a>(&'a u8, &'static str);

unsafe impl<#[may_dangle] 'a> Drop for Inspector<'a> {
    fn drop(&mut self) {
        println!("Inspector(_, {}) knows when *not* to inspect.", self.1);
    }
}

struct World<'a> {
    days: Box<u8>,
    inspector: Option<Inspector<'a>>,
}

fn main() {
    let mut world = World {
        inspector: None,
        days: Box::new(1),
    };
    world.inspector = Some(Inspector(&world.days, "gadget"));
}
```

Use of this attribute requires the `Drop` impl to be marked `unsafe` because the
compiler is not checking the implicit assertion that no potentially expired data
(e.g. `self.0` above) is accessed.

The attribute can be applied to any number of lifetime and type parameters. In
the following example, we assert that we access no data behind a reference of
lifetime `'b` and that the only uses of `T` will be moves or drops, but omit
the attribute from `'a` and `U`, because we do access data with that lifetime
and that type:

```rust
#![feature(dropck_eyepatch)]
use std::fmt::Display;

struct Inspector<'a, 'b, T, U: Display>(&'a u8, &'b u8, T, U);

unsafe impl<'a, #[may_dangle] 'b, #[may_dangle] T, U: Display> Drop for Inspector<'a, 'b, T, U> {
    fn drop(&mut self) {
        println!("Inspector({}, _, _, {})", self.0, self.3);
    }
}
```

It is sometimes obvious that no such access can occur, like the case above.
However, when dealing with a generic type parameter, such access can
occur indirectly. Examples of such indirect access are:

- invoking a callback,
- via a trait method call.

(Future changes to the language, such as impl specialization, may add
other avenues for such indirect access.)

Here is an example of invoking a callback:

```rust
struct Inspector<T>(T, &'static str, Box<for <'r> fn(&'r T) -> String>);

impl<T> Drop for Inspector<T> {
    fn drop(&mut self) {
        // The `self.2` call could access a borrow e.g. if `T` is `&'a _`.
        println!("Inspector({}, {}) unwittingly inspects expired data.",
                 (self.2)(&self.0), self.1);
    }
}
```

Here is an example of a trait method call:

```rust
use std::fmt;

struct Inspector<T: fmt::Display>(T, &'static str);

impl<T: fmt::Display> Drop for Inspector<T> {
    fn drop(&mut self) {
        // There is a hidden call to `<T as Display>::fmt` below, which
        // could access a borrow e.g. if `T` is `&'a _`
        println!("Inspector({}, {}) unwittingly inspects expired data.",
                 self.0, self.1);
    }
}
```

And of course, all of these accesses could be further hidden within
some other method invoked by the destructor, rather than being written
directly within it.

In all of the above cases where the `&'a u8` is accessed in the
destructor, adding the `#[may_dangle]`
attribute makes the type vulnerable to misuse that the borrow
checker will not catch, inviting havoc. It is better to avoid adding
the attribute.

## A related side note about drop order

While the drop order of fields inside a struct is defined, relying on it is
fragile and subtle. When the order matters, it is better to use the
[`ManuallyDrop`] wrapper.

## Is that all about drop checker?

It turns out that when writing unsafe code, we generally don't need to
worry at all about doing the right thing for the drop checker. However there
is one special case that you need to worry about, which we will look at in
the next section.

[rfc1327]: https://github.com/rust-lang/rfcs/blob/master/text/1327-dropck-param-eyepatch.md
[rfc1857]: https://github.com/rust-lang/rfcs/blob/master/text/1857-stabilize-drop-order.md
[`manuallydrop`]: ../std/mem/struct.ManuallyDrop.html
