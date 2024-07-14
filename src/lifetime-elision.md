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

* 입력 위치에서 생략된 수명들은 각각 별개의 수명 매개변수가 됩니다.

* 입력 위치에서 수명이 하나밖에 없다면 (생략되든 되지 않든), 그 수명이 출력 수명들 *모두에* 적용됩니다.

* 만약 입력 위치에서 수명이 여러 개가 있는데, 그 중의 하나가 `&self`이거나 `&mut self`라면, `self`의 수명이 출력 위치에서 수명이 생략된 *모든* 수명들에게 할당됩니다.

* 이 모든 경우가 해당하지 않는다면, 출력 수명을 생략하는 것은 오류입니다.

예제입니다:

<!-- ignore: simplified code -->
```rust,ignore
fn print(s: &str);                                      // 생략됨
fn print<'a>(s: &'a str);                               // 확장됨

fn debug(lvl: usize, s: &str);                          // 생략됨
fn debug<'a>(lvl: usize, s: &'a str);                   // 확장됨

fn substr(s: &str, until: usize) -> &str;               // 생략됨
fn substr<'a>(s: &'a str, until: usize) -> &'a str;     // 확장됨

fn get_str() -> &str;                                   // 오류

fn frob(s: &str, t: &str) -> &str;                      // 오류

fn get_mut(&mut self) -> &mut T;                        // 생략됨
fn get_mut<'a>(&'a mut self) -> &'a mut T;              // 확장됨

fn args<T: ToCStr>(&mut self, args: &[T]) -> &mut Command                  // 생략됨
fn args<'a, 'b, T: ToCStr>(&'a mut self, args: &'b [T]) -> &'a mut Command // 확장됨

fn new(buf: &mut [u8]) -> BufWriter;                    // 생략됨
fn new(buf: &mut [u8]) -> BufWriter<'_>;                // 생략됨 (`rust_2018_idioms`을 사용해서)
fn new<'a>(buf: &'a mut [u8]) -> BufWriter<'a>          // 확장됨
```
