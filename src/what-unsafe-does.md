# 불안전한 러스트는 무엇을 할 수 있는가

불안전한 러스트에서 다른 점은 이런 것들이 가능하다는 것뿐입니다:

* 생 포인터 역참조하기
* `unsafe` 함수 호출하기 (C 함수나, 컴파일러 내부, 그리고 할당자를 직접 호출하는 것 포함)
* `unsafe` 트레잇 구현하기
* `static` 변수의 값을 변경하기
* `union` 의 필드를 접근하기

이게 전부입니다. 이런 연산들이 불안전의 영역으로 추방된 이유는, 이것들 중 하나라도 잘못 사용할 경우 그토록 두렵던 미정의 동작을 일으키기 때문입니다. 
미정의 동작을 일으키면 컴파일러가 당신의 프로그램에 임의의 나쁜 짓들을 할 수 있는 모든 권리를 얻게 됩니다. 당연하게도 미정의 동작은 *일으켜서는 안됩니다.*

C와 다르게, 미정의 동작은 러스트에서는 꽤 제한되어 있습니다. 러스트의 코어 언어가 막으려고 하는 것들은 이런 것들입니다: 

* 달랑거리거나 정렬되어 있지 않은 포인터를 역참조하는 것 (`*` 연산자를 사용해서) (밑 참조)
* [레퍼런스 규칙][alias] 을 어기는 것
* 잘못된 호출 ABI를 이용해 함수를 호출하거나 잘못된 되감기 ABI를 가지고 있는 함수에서 되감는 것
* [데이터 경합][race] 을 일으키는 것
* 지금 실행하는 스레드가 지원하지 않는 [타겟 기능들][target] 로 컴파일된 코드를 실행하는 것
* 잘못된 값을 생산하는 것 (혼자서나 `enum`/`struct`/배열/튜플과 같은 복합 타입의 필드로써나):
    * 0도 1도 아닌 `bool`
    * 유효하지 않은 형(形)을 사용하는 `enum`
    * 널 `fn` 포인터
    * [0x0, 0xD7FF] 와 [0xE000, 0x10FFFF] 범위를 벗어나는 `char`
    * `!` 타입의 값 (이 타입의 모든 값은 유효하지 않습니다)
    * [초기화되지 않은 메모리][uninit] 로부터 읽어들인 정수 (`i*`/`u*`), 부동소수점 값 (`f*`), 혹은 생 포인터, 혹은 `str` 안의 초기화되지 않은 메모리.
    * 달랑거리거나, 정렬되지 않았거나, 유효하지 않은 값을 가리키는 레퍼런스/`Box`
    * 잘못된 메타데이터를 가지고 있는 넓은 레퍼런스, `Box`, 혹은 생 포인터:
        * `dyn Trait` 메타데이터는 그것이 `Trait`의 vtable을 가리키는 포인터가 아닐 경우 유효하지 않습니다
        * 슬라이스 메타데이터는 길이가 올바른 `usize` 가 아니면 유효하지 않습니다 (즉, 초기화되지 않은 메모리에서 읽어들이면 안됩니다)
    * 널인 [`NonNull`] 같은, 커스텀으로 잘못된 값이 들어있는 타입 (커스텀으로 잘못된 값을 요청하는 것은 불안정한 기능이지만 `NonNull` 같은, 몇 가지 안정 버전의 표준 라이브러리 타입들은 이것을 사용합니다.)

"미정의 동작"에 관해 더 자세한 설명이 필요하다면 [참조서][behavior-considered-undefined] 를 참고하셔도 됩니다.

값을 "생산하는" 일은 값이 할당되거나, 함수/기본 연산에 전달되거나, 함수/기본 연산에서 반환될 때 일어납니다.

레퍼런스/포인터가 "달랑거린다"는 것은 그것이 널이거나 그것이 가리키는 바이트가 모두 같은 할당처에 있는 것이 아니라는 뜻입니다 (그 바이트들은 모두 *어떤* 할당처에는 있어야 합니다). 
그것이 가리키는 바이트들의 너비는 포인터 값과 참조되는 타입의 크기에 따라 결정됩니다. 따라서 만약 너비가 비어 있다면, "달랑거리는" 것은 "널"인 것과 같습니다. 슬라이스와 문자열은 그들의 전체 범위를 가리킨다는 것을 유의한다면, 
길이 메타데이터가 너무 크지 않도록 하는 것이 중요해집니다 (특히, 할당량과 그에 따른 슬라이스와 문자열은 `isize::MAX` 바이트보다 클 수 없습니다). 만약 어떤 이유로 이것이 거추장스럽다면, 생 포인터를 쓰는 것을 고려해 보세요.

그게 전부입니다. 그것이 러스트에 있는 미정의 동작의 모든 원인입니다. 물론 불안전한 함수들과 트레잇들은 프로그램이 지켜야 하는 임의의 다른 제약들을 걸 수 있고, 그것을 어기면 미정의 동작이 일어나겠죠. 
예를 들어, 할당자 API는 할당되지 않은 메모리를 해제하는 것은 미정의 동작이라고 정의합니다.

그러나 이런 제약들을 어기면 결국 위의 문제들 중 하나로 이어지게 될 것입니다. 어떤 추가적인 제약들은 컴파일러 내부가 코드를 최적화하는 과정에서 하는 특별한 가정들에서부터 비롯될 수도 있습니다. 
예를 들어, `Vec` 과 `Box` 는 그들의 포인터가 항상 널이 아니도록 하는 내부 코드를 사용합니다. 

러스트는 이 외의 다른 애매한 작업들에는 꽤나 관대합니다. 러스트는 이런 작업들을 "안전하다"고 판단합니다: 

* 데드락 (교착 상태)
* [경합 조건][race] 이 있는 것
* 메모리 누수
* (`+` 등의 기본 연산자를 이용한) 정수 오버플로우
* 


* Abort the program
* Delete the production database

For more detailed information, you may refer to [the reference][behavior-not-considered-unsafe].

However any program that actually manages to do such a thing is *probably*
incorrect. Rust provides lots of tools to make these things rare, but
these problems are considered impractical to categorically prevent.

[alias]: references.html
[uninit]: uninitialized.html
[race]: races.html
[target]: https://doc.rust-lang.org/reference/attributes/codegen.html#the-target_feature-attribute
[`NonNull`]: https://doc.rust-lang.org/std/ptr/struct.NonNull.html
[behavior-considered-undefined]: https://doc.rust-lang.org/reference/behavior-considered-undefined.html
[behavior-not-considered-unsafe]: https://doc.rust-lang.org/reference/behavior-not-considered-unsafe.html
