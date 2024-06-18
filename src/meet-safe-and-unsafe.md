# 안전함과 불안전함을 마주하라

![safe and unsafe](img/safeandunsafe.svg)

낮은 레벨의 구현 세부사항에 대해 걱정하지 않아도 되면 참 좋을 것입니다. 빈 튜플이 얼마만큼의 공간을 차지하는지 대체 누가 신경쓸까요?
슬프게도 이런 것들은 어떤 때에는 중요하고, 우리는 이런 것들을 걱정해야 합니다. 개발자들이 구현 세부사항에 대해서 걱정하기 시작하는 가장 흔한 이유는 성능이지만 그보다 더 중요한 것은, 
하드웨어, 운영체제, 혹은 다른 언어들과 직접적으로 상호작용할 때 이런 세부적인 것들이 올바른 코드를 작성하는 것에 대한 문제가 될 수 있습니다.

안전한 프로그래밍 언어에서 구현 세부사항이 중요해지기 시작할 때, 프로그래머들은 보통 3가지 선택지가 있습니다:

- 컴파일러나 런타임이 최적화를 수행하도록 코드에 꼼수 쓰기
- 원하는 구현을 얻기 위해 훨씬 부자연스럽거나 거추장스러운 디자인을 채용하기
- 구현을 그런 세부사항을 다룰 수 있는 언어로 다시 작성하기

마지막 선택지에서, 프로그래머들이 주로 쓰는 언어는 *C* 입니다. 이 언어는 C 인터페이스만 정의하는 시스템들과 
상호작용하기 위해 보통 필요합니다. 

불행하게도 C는 (때때로 좋은 이유로) 사용하기에 엄청나게 불안전하고, 다른 언어들과 협업하려고 할 때 
이 불안전함은 증폭됩니다. C와 다른 언어가 어떤 일이 일어날지 서로 동의하고, 서로의 발을 밟지 않게 하기 위해 
주의가 필요합니다.

그래서 이것이 러스트와 무슨 상관일까요?

그게, C와 다르게, 러스트는 안전한 프로그래밍 언어입니다.

하지만, C와 마찬가지로, 러스트는 불안전한 언어입니다.

더 정확하게 말하자면, 러스트는 안전한 언어와 불안전한 언어 두 가지를 모두 *포함하고 있습니다.*

러스트는 *안전한 러스트* 와 *불안전한 러스트* 라는 두 개의 프로그래밍 언어의 조합이라고 볼 수 있습니다. 
편리하게도, 이 이름들은 말 그대로입니다: 안전한 러스트는 안전하고, 불안전한 러스트는, 음, 그렇지 않죠. 
사실 불안전한 러스트는 *매우* 불안전한 것들을 할 수 있게 해 줍니다. 러스트 개발자들이 하지 말라고 간곡히 
부탁하는 것들이지만, 우리는 그냥 무시하고 할 겁니다. 

안전한 러스트는 *진정한* 러스트 프로그래밍 언어입니다. 만약 안전한 러스트만 작성한다면, 
타입 안정성이나 메모리 안정성은 절대 걱정할 필요가 없을 겁니다. 달랑거리는 포인터나, 해제 후 사용, 
혹은 다른 종류의 미정의 동작은 겪지 않을 테니까요.



The standard library also gives you enough utilities out of the box that you'll
be able to write high-performance applications and libraries in pure idiomatic
Safe Rust.

But maybe you want to talk to another language. Maybe you're writing a
low-level abstraction not exposed by the standard library. Maybe you're
*writing* the standard library (which is written entirely in Rust). Maybe you
need to do something the type-system doesn't understand and just *frob some dang
bits*. Maybe you need Unsafe Rust.

Unsafe Rust is exactly like Safe Rust with all the same rules and semantics.
It just lets you do some *extra* things that are Definitely Not Safe
(which we will define in the next section).

The value of this separation is that we gain the benefits of using an unsafe
language like C — low level control over implementation details — without most
of the problems that come with trying to integrate it with a completely
different safe language.

There are still some problems — most notably, we must become aware of properties
that the type system assumes and audit them in any code that interacts with
Unsafe Rust. That's the purpose of this book: to teach you about these assumptions
and how to manage them.
