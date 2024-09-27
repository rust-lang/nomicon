# 검사받는 초기화되지 않은 메모리

C와 같이, 러스트에서 모든 스택 변수는 값이 배정되기 전까지 초기화되지 않은 상태입니다. C와 다르게, 러스트는 값을 배정하기 전에 그 변수들을 읽는 것을 컴파일 때 방지합니다:

```rust,compile_fail
fn main() {
    let x: i32;
    println!("{}", x);
}
```

```text
  |
3 |     println!("{}", x);
  |                    ^ use of possibly uninitialized `x`
```

이것은 기본적인 경우 검사에 기반한 것입니다: `x`가 처음 쓰이기 전에 모든 경우에서 `x`에 값을 할당해야 합니다. 짧게 말하자면, 우리는 "`x`가 초기화됐다" 혹은 "`x`가 미초기화됐다"고도 말합니다.

흥미롭게도, 만약 늦은 초기화를 할 때 모든 경우에서 변수에 값을 정확히 한 번씩 할당한다면, 러스트는 변수가 가변이어야 한다는 제약을 걸지 않습니다. 하지만 이 분석은 상수 분석 같은 것을 이용하지 못합니다. 따라서 이 코드는 컴파일되지만:

```rust
fn main() {
    let x: i32;

    if true {
        x = 1;
    } else {
        x = 2;
    }

    println!("{}", x);
}
```

이건 아닙니다:

```rust,compile_fail
fn main() {
    let x: i32;
    if true {
        x = 1;
    }
    println!("{}", x);
}
```

```text
  |
6 |     println!("{}", x);
  |                    ^ use of possibly uninitialized `x`
```

이 코드는 컴파일됩니다:

```rust
fn main() {
    let x: i32;
    if true {
        x = 1;
        println!("{}", x);
    }
    // 초기화되지 않는 경우가 있지만 신경쓰지 않습니다
    // 그 경우에서는 그 변수를 사용하지 않기 때문이죠
}
```

당연하게도, 이 분석이 진짜 값을 신경쓰는 건 아니지만, 프로그램 구조와 의존성에 대한 비교적 높은 수준의 이해를 하고 있습니다. 예를 들어, 다음의 코드는 작동합니다:

```rust
let x: i32;

loop {
    // Rust doesn't understand that this branch will be taken unconditionally,
    // because it relies on actual values.
    if true {
        // But it does understand that it will only be taken once because
        // we unconditionally break out of it. Therefore `x` doesn't
        // need to be marked as mutable.
        x = 0;
        break;
    }
}
// It also knows that it's impossible to get here without reaching the break.
// And therefore that `x` must be initialized here!
println!("{}", x);
```

If a value is moved out of a variable, that variable becomes logically
uninitialized if the type of the value isn't Copy. That is:

```rust
fn main() {
    let x = 0;
    let y = Box::new(0);
    let z1 = x; // x is still valid because i32 is Copy
    let z2 = y; // y is now logically uninitialized because Box isn't Copy
}
```

However reassigning `y` in this example *would* require `y` to be marked as
mutable, as a Safe Rust program could observe that the value of `y` changed:

```rust
fn main() {
    let mut y = Box::new(0);
    let z = y; // y is now logically uninitialized because Box isn't Copy
    y = Box::new(1); // reinitialize y
}
```

Otherwise it's like `y` is a brand new variable.
