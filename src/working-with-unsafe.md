# 불안전함과 함께 일하는 것

러스트는 일반적으로 우리가 불안전한 러스트와 이야기할 때 정해진 범위 안에서, 이진수의 방식으로 이야기하는 도구만 줍니다. 불행하게도 현실은 그것보다 훨씬 복잡하죠. 예를 들어, 다음의 간단한 함수를 생각해 볼까요:

```rust
fn index(idx: usize, arr: &[u8]) -> Option<u8> {
    if idx < arr.len() {
        unsafe {
            Some(*arr.get_unchecked(idx))
        }
    } else {
        None
    }
}
```

이 함수는 안전하고 올바릅니다. 우리는 인덱스가 범위 안에 있는지 확인하고, 그렇다면 더 이상 확인하지 않고 배열을 바로 인덱싱합니다. 이렇게 잘 구현된 불안전한 함수를 *건전하다* 고 하는데, 
이것은 안전한 코드가 이 코드를 악용해서 미정의 동작을 유발할 수 없다는 뜻입니다 (이건 바로 안전한 러스트의 유일한 근본적 특성이었죠).

하지만 이런 흔한 함수 안에서도, 불안전한 코드 블럭의 범위는 모호합니다. 여기서 `<` 를 `<=` 로 바꾸는 경우를 생각해 보세요:

```rust
fn index(idx: usize, arr: &[u8]) -> Option<u8> {
    if idx <= arr.len() {
        unsafe {
            Some(*arr.get_unchecked(idx))
        }
    } else {
        None
    }
}
```

이 프로그램은 이제 *불건전하고*, 안전한 러스트는 미정의 동작을 유발할 수 있지만, *우리는 안전한 코드만 수정했을 뿐입니다*. 이것이 바로 안전함의 근본적인 문제입니다: 지역에 한정되어 있지 않죠. 
이 불안전한 연산의 건전성은 어쩔 수 없이 다른 "안전한" 연산들이 만든 상태에 의존하게 됩니다.

불안전함을 도입하는 것이 임의의 다른 종류의 유해함을 고려하지 않아도 되는 점에서, 안전성은 경계가 있습니다. 예를 들어, 슬라이스에 확인되지 않은 인덱싱을 하는 것은 갑자기 슬라이스가 널이 되거나 초기화되지 않은 메모리를 포함하게 되는 경우를 걱정해야 하는 것은 아닙니다. 근본적으로 변하는 것은 없습니다. 하지만 프로그램이 본질적으로 상태를 가지게 되는 것과 당신의 불안전한 작업들이 임의의 다른 상태에 의존할 수 있게 된다는 점에서, 안전성은 분리되어 있지는 않습니다.

이런 지역에 한정되지 않는 특성은 우리가 실제로 지속되는 상태를 포함하게 되면 더욱 심해집니다. 다음의 간단한 `Vec` 구현을 생각해 보세요: 

```rust
use std::ptr;

// 주의: 이 정의는 순진합니다. Vec을 구현하는 챕터를 참고하세요.
pub struct Vec<T> {
    ptr: *mut T,
    len: usize,
    cap: usize,
}

// 이 구현은 무량 타입을 제대로 다루지 못하니 주의하세요.
// Vec을 구현하는 챕터를 보세요.
impl<T> Vec<T> {
    pub fn push(&mut self, elem: T) {
        if self.len == self.cap {
            // 이 예제에는 중요하지 않은 부분입니다
            self.reallocate();
        }
        unsafe {
            ptr::write(self.ptr.add(self.len), elem);
            self.len += 1;
        }
    }
    # fn reallocate(&mut self) { }
}

# fn main() {}
```

이 코드는 합리적으로 검사하고 편하게 확인할 수 있을 만큼 간단합니다. 이제 이런 메서드를 추가하는 것을 생각해 보세요: 

<!-- ignore: simplified code -->
```rust,ignore
fn make_room(&mut self) {
    // 용량을 늘립니다
    self.cap += 1;
}
```

이 코드는 100% 안전한 러스트이지만, 동시에 완전히 불건전합니다. 용량을 변경하는 것은 Vec의 불문율(`cap` 이 Vec의 할당된 용량을 반영한다는 것)을 파괴합니다. 이것은 Vec의 나머지 코드가 방어할 수 있는 것이 아닙니다. Vec은 `cap` 필드를 검증할 방법이 없기 때문에 믿는 *수밖에 없습니다*.

이 `unsafe` 코드는 구조체 필드의 불문율에 의존하기 때문에, 한 함수 전체를 오염시키는 것보다 더한 짓을 합니다: 한 *모듈* 전체를 오염시키죠. 

Generally, the only bullet-proof way to limit the scope of unsafe code is at the
module boundary with privacy.

However this works *perfectly*. The existence of `make_room` is *not* a
problem for the soundness of Vec because we didn't mark it as public. Only the
module that defines this function can call it. Also, `make_room` directly
accesses the private fields of Vec, so it can only be written in the same module
as Vec.

It is therefore possible for us to write a completely safe abstraction that
relies on complex invariants. This is *critical* to the relationship between
Safe Rust and Unsafe Rust.

We have already seen that Unsafe code must trust *some* Safe code, but shouldn't
trust *generic* Safe code. Privacy is important to unsafe code for similar reasons:
it prevents us from having to trust all the safe code in the universe from messing
with our trusted state.

Safety lives!
