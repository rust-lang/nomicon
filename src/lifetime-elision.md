# 수명 생략

흔한 코드 패턴을 더 편하게 만들기 위해서, 러스트는 함수 시그니처에서 수명이 *생략되는 것을* 허용합니다.

*수명의 위치*는 타입 안에서 수명을 쓸 수 있는 곳 어디나입니다:

<!-- ignore: simplified code -->
```rust,ignore
&'a T
&'a mut T
T<'a>
```

수명의 위치는 "입력" 또는 "출력"으로 나타날 수 있습니다:

* `fn` 정의, `fn` 타입, 그리고 `Fn`, `FnMut`, `FnOnce` 트레잇들에서, 입력은 매개변수들의 타입을 가리키고, 출력은 결과 타입을 가리킵니다. 따라서 `fn foo(s: &str) -> (&str, &str)`는 입력 위치에서 하나의 수명을,
  출력 위치에서 두 개의 수명을 생략한 것이죠. `fn` 메서드 정의에서 입력 위치는 메서드의 `impl` 부분에 있는 수명들(혹은, 기본 메서드에 한해서, 트레잇 헤더에 있는 수명들도요)을 포함하지 않는다는 것을 유의하세요.

* `impl` 부분에서는 모든 타입이 입력입니다. 따라서 `impl Trait<&T> for Struct<&T>`은 입력 위치에서 두 개의 수명을 생략했고, `impl Struct<&T>`은 하나의 수명을 생략한 것입니다.

생략 규칙은 다음과 같습니다:

* Each elided lifetime in input position becomes a distinct lifetime
  parameter.

* If there is exactly one input lifetime position (elided or not), that lifetime
  is assigned to *all* elided output lifetimes.

* If there are multiple input lifetime positions, but one of them is `&self` or
  `&mut self`, the lifetime of `self` is assigned to *all* elided output lifetimes.

* Otherwise, it is an error to elide an output lifetime.

Examples:

<!-- ignore: simplified code -->
```rust,ignore
fn print(s: &str);                                      // elided
fn print<'a>(s: &'a str);                               // expanded

fn debug(lvl: usize, s: &str);                          // elided
fn debug<'a>(lvl: usize, s: &'a str);                   // expanded

fn substr(s: &str, until: usize) -> &str;               // elided
fn substr<'a>(s: &'a str, until: usize) -> &'a str;     // expanded

fn get_str() -> &str;                                   // ILLEGAL

fn frob(s: &str, t: &str) -> &str;                      // ILLEGAL

fn get_mut(&mut self) -> &mut T;                        // elided
fn get_mut<'a>(&'a mut self) -> &'a mut T;              // expanded

fn args<T: ToCStr>(&mut self, args: &[T]) -> &mut Command                  // elided
fn args<'a, 'b, T: ToCStr>(&'a mut self, args: &'b [T]) -> &'a mut Command // expanded

fn new(buf: &mut [u8]) -> BufWriter;                    // elided
fn new(buf: &mut [u8]) -> BufWriter<'_>;                // elided (with `rust_2018_idioms`)
fn new<'a>(buf: &'a mut [u8]) -> BufWriter<'a>          // expanded
```
