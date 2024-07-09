# 수명

러스트는 이런 규칙들을 *수명*을 통해서 강제합니다. 수명은 레퍼런스가 유효해야 하는, 이름이 지어진 코드의 지역입니다. 이 지역들은 꽤나 복잡할 수 있는데, 프로그램의 실행 분기에 대응하기 때문입니다. 
또한 이런 실행 분기에는 구멍까지도 있을 수 있는데, 레퍼런스가 다시 쓰이기 전에 재초기화된다면, 이 레퍼런스를 무효화할 수 있기 때문입니다. 
레퍼런스를 포함하는 (또는 포함하는 척하는) 타입도 수명을 붙여서 러스트가 그것을 무효화하는 것을 막게 할 수 있습니다. 

우리의 예제 대부분, 수명은 코드 구역과 대응할 것입니다. 이것은 우리의 예제가 간단하기 때문입니다. 그렇게 대응되지 않는 복잡한 경우는 밑에 서술하겠습니다.

함수 본문 안에서 러스트는 보통 관련된 수명을 명시적으로 쓰게 하지 않습니다. 이것은 지역적인 문맥에서 수명에 대해 말하는 것은 대부분 별로 필요없기 때문입니다: 러스트는 모든 정보를 가지고 있고, 
모든 것들이 가능한 한 최적으로 동작하도록 할 수 있습니다. 컴파일러가 아니라면 일일히 작성해야 하는 많은 무명 구역과 경계들을 컴파일러가 대신해 주어서 당신의 코드가 "그냥 작동하게" 해 줍니다.

그러나 일단 함수 경계를 건너고 나면 수명에 대해서 이야기해야만 합니다. 수명은 보통 작은 따옴표로 표시됩니다: `'a`, `'static` 처럼요. 수명이라는 주제에 발가락을 담그기 위해서, 우리는 코드 구역을 수명으로 수식할 수 있는 척을 하며, 이 챕터의 시작부터 있는 예제들의 문법적 설탕을 해독해 보겠습니다.

원래 우리의 예제들은 코드 구역과 수명에 대해서 *공격적인* 문법적 설탕-- 고당 옥수수콘 시럽 같은 --을 이용했는데, 모든 것들을 명시적으로 적는 것은 *굉장히 요란하기* 때문입니다. 
모든 러스트 코드는 이런 식의 공격적인 추론과 "뻔한" 것들을 생략하는 것에 의존합니다. 

문법적 설탕 중 하나의 특별한 조각은 모든 `let` 문장이 암시적으로 코드 구역을 시작한다는 점입니다. 대부분의 경우에는 이것이 문제가 되지는 않습니다. 그러나 서로 참조하는 변수들에게는 문제가 됩니다. 
간단한 예제로, 이런 간단한 러스트 코드 조각을 해독해 봅시다:

```rust
let x = 0;
let y = &x;
let z = &y;
```

대여 검사기는 항상 수명의 길이를 최소화하려고 하기 때문에, 아마 이런 식으로 해독할 것입니다:

<!-- ignore: desugared code -->
```rust,ignore
// 주의: `'a: {` 나 `&'b x` 는 유효한 문법이 아닙니다!
'a: {
    let x: i32 = 0;
    'b: {
        let y: &'b i32 = &'b x;
        'c: {
            let z: &'c &'b i32 = &'c y; // "i32의 레퍼런스의 레퍼런스" (수명이 표시되어 있음)
        }
    }
}
```

와. 이건... 별로네요. 러스트가 이런 것들을 간단하게 만들어 준다는 것을 잠시 감사하는 시간을 가집시다.

...

자, 계속하자면, 외부 범위에 레퍼런스를 넘기면 러스트는 더 긴 수명을 추론하게 됩니다:

```rust
let x = 0;
let z;
let y = &x;
z = y;
```

<!-- ignore: desugared code -->
```rust,ignore
'a: {
    let x: i32 = 0;
    'b: {
        let z: &'b i32;
        'c: {
            // x의 레퍼런스가 'b 구역으로 넘겨지기 때문에 
            // 'b 가 됩니다.
            let y: &'b i32 = &'b x;
            z = y;
        }
    }
}
```

## 예제: 본체보다 오래 사는 레퍼런스들

좋습니다, 예전의 예제 중 몇 가지를 살펴봅시다:

```rust,compile_fail
fn as_str(data: &u32) -> &str {
    let s = format!("{}", data);
    &s
}
```

이렇게 해독됩니다:

<!-- ignore: desugared code -->
```rust,ignore
fn as_str<'a>(data: &'a u32) -> &'a str {
    'b: {
        let s = format!("{}", data);
        return &'a s;
    }
}
```

`as_str`의 시그니처는 *어떤* 수명을 가지고 있는, u32의 레퍼런스를 받고 *딱 그만큼 사는*, str의 레퍼런스를 만들 수 있다고 약속하고 있습니다. 이미 여기서 우리는 왜 이 시그니처가 문제가 있을 수도 있는지 알 수 있습니다. 이것은 우리가 str을 u32의 레퍼런스가 있던 코드 구역에서, 혹은 *더 이전의 구역에서* 찾을 것을 내포하고 있습니다. 이것은 좀 무리한 요구입니다.

그 다음 우리는 String `s`를 계산하고, 그것을 가리키는 레퍼런스를 반환합니다. 우리의 함수가 체결한 계약은 레퍼런스가 `'a`보다 더 살아야 한다고 되어 있으므로, 이것이 바로 우리가 이 레퍼런스에 할당해야 하는 수명입니다. 불행하게도 `s`는 `'b` 구역에서 정의되었으므로, 이 코드가 건전할 수 있는 유일한 방법은 `'b`가 `'a`를 포함하는 것입니다 -- `'a`는 함수 호출 자체를 포함해야 하므로, 이는 명백하게 성립하지 않습니다. 그러므로 우리는 그 수명이 본체를 능가하는 레퍼런스를 만들었는데, 이것은 *말 그대로* 레퍼런스가 할 수 없다고 우리가 말했던 가장 첫번째 것입니다. 컴파일러는 당연히 우리 눈앞에서 터질 겁니다.

좀더 확실하게 하기 위해 이 예제를 확장해 볼 수 있습니다:

<!-- ignore: desugared code -->
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
            // x가 유효한 범위 전체 동안 빌림이 유지될 필요가 
            // 없기 때문에 새로운 구역을 집어넣었습니다. as_str의 반환값은 
            // 이 함수 호출 이전에 존재하는 str을 찾아야만 합니다.
            // 당연히 일어나지 않을 일이죠.
            println!("{}", as_str::<'d>(&'d x));
        }
    }
}
```

젠장!

이 함수를 올바르게 작성하는 방법은 물론 다음과 같습니다:

```rust
fn to_string(data: &u32) -> String {
    format!("{}", data)
}
```

우리는 함수 안에서 소유한 값을 생성해서 반환해야 합니다! 우리가 이렇게 `&'a str`을 반환하려면 이것이 `&'a u32`의 필드 안에 있어야만 하는데, 당연히 이 경우에는 말이 안됩니다.

(사실 우리는 문자열 상수값을 반환할 수도 있었습니다. 이 상수값은 스택의 맨 밑바닥에 있다고 생각할 수 있습니다. 이 구현이 우리가 원하는 것을 *조금* 제한하기는 하지만요.)

## 예제: 가변 레퍼런스의 복제

다른 예제를 볼까요:

```rust,compile_fail
let mut data = vec![1, 2, 3];
let x = &data[0];
data.push(4);
println!("{}", x);
```

<!-- ignore: desugared code -->
```rust,ignore
'a: {
    let mut data: Vec<i32> = vec![1, 2, 3];
    'b: {
        // 'b 는 우리가 이 레퍼런스가 필요한 동안 유지됩니다
        // (`println!`까지 가야 하죠)
        let x: &'b i32 = Index::index::<'b>(&'b data, 0);
        'c: {
            // &mut가 더 오래 살아남을 필요가 없기 때문에
            // 임시 구역을 추가합니다
            Vec::push(&'c mut data, 4);
        }
        println!("{}", x);
    }
}
```

여기서 문제는 조금 더 감추어져 있고 흥미롭습니다. 우리는 다음의 이유로 러스트가 이 프로그램을 거부하기를 원합니다: 우리는 `data`의 하위 변수를 참조하는, 살아있는 불변 레퍼런스 `x`를 가지고 있는데, 
이 동안 `data`에 `push` 함수를 호출하여 가변 레퍼런스를 취하려고 합니다. 

This would create an aliased mutable reference, which would
violate the *second* rule of references.

However this is *not at all* how Rust reasons that this program is bad. Rust
doesn't understand that `x` is a reference to a subpath of `data`. It doesn't
understand `Vec` at all. What it *does* see is that `x` has to live for `'b` in
order to be printed. The signature of `Index::index` subsequently demands that
the reference we take to `data` has to survive for `'b`. When we try to call
`push`, it then sees us try to make an `&'c mut data`. Rust knows that `'c` is
contained within `'b`, and rejects our program because the `&'b data` must still
be alive!

Here we see that the lifetime system is much more coarse than the reference
semantics we're actually interested in preserving. For the most part, *that's
totally ok*, because it keeps us from spending all day explaining our program
to the compiler. However it does mean that several programs that are totally
correct with respect to Rust's *true* semantics are rejected because lifetimes
are too dumb.

## The area covered by a lifetime

A reference (sometimes called a *borrow*) is *alive* from the place it is
created to its last use. The borrowed value needs to outlive only borrows that
are alive. This looks simple, but there are a few subtleties.

The following snippet compiles, because after printing `x`, it is no longer
needed, so it doesn't matter if it is dangling or aliased (even though the
variable `x` *technically* exists to the very end of the scope).

```rust
let mut data = vec![1, 2, 3];
let x = &data[0];
println!("{}", x);
// This is OK, x is no longer needed
data.push(4);
```

However, if the value has a destructor, the destructor is run at the end of the
scope. And running the destructor is considered a use ‒ obviously the last one.
So, this will *not* compile.

```rust,compile_fail
#[derive(Debug)]
struct X<'a>(&'a i32);

impl Drop for X<'_> {
    fn drop(&mut self) {}
}

let mut data = vec![1, 2, 3];
let x = X(&data[0]);
println!("{:?}", x);
data.push(4);
// Here, the destructor is run and therefore this'll fail to compile.
```

One way to convince the compiler that `x` is no longer valid is by using `drop(x)` before `data.push(4)`.

Furthermore, there might be multiple possible last uses of the borrow, for
example in each branch of a condition.

```rust
# fn some_condition() -> bool { true }
let mut data = vec![1, 2, 3];
let x = &data[0];

if some_condition() {
    println!("{}", x); // This is the last use of `x` in this branch
    data.push(4);      // So we can push here
} else {
    // There's no use of `x` in here, so effectively the last use is the
    // creation of x at the top of the example.
    data.push(5);
}
```

And a lifetime can have a pause in it. Or you might look at it as two distinct
borrows just being tied to the same local variable. This often happens around
loops (writing a new value of a variable at the end of the loop and using it for
the last time at the top of the next iteration).

```rust
let mut data = vec![1, 2, 3];
// This mut allows us to change where the reference points to
let mut x = &data[0];

println!("{}", x); // Last use of this borrow
data.push(4);
x = &data[3]; // We start a new borrow here
println!("{}", x);
```

Historically, Rust kept the borrow alive until the end of scope, so these
examples might fail to compile with older compilers. Also, there are still some
corner cases where Rust fails to properly shorten the live part of the borrow
and fails to compile even when it looks like it should. These'll be solved over
time.
