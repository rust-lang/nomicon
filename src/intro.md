# 러스토노미콘

<div class="warning">

경고:
이 책은 미완성입니다.
모든 것을 문서화하고, 예전에는 맞았지만 지금은 아닌 내용들을 고치는 데는 시간이 걸립니다.
[이슈 트래커]에 없는 기능이나, 예전에는 맞았지만 지금은 아닌 내용들을 보실 수 있고, 아직 보고되지 않은 실수나 생각들이 있다면 거기에 얼마든지 새로운 이슈를 열어 주세요.

</div>

[이슈 트래커]: https://github.com/nomicon-kr/nomicon-kr.github.io/issues

## 비안전 러스트의 흑마법들

> **이 지식은 "있는 그대로" 제공되며, 표현할 수 없는 공포스러운 것들을 해방시켜 당신의 정신을 산산조각내고, 당신의 마음을 알 수 없는 무한한 우주에 떠다니게 하는 것, 혹은 그 이상의 범위를 포함한 사항에 있어서, 명시적 혹은 묵시적인 어떠한 보증도 하지 않는다.**

러스토노미콘은 *비안전한* 러스트 프로그램을 작성할 때 알아야 하는 모든 무시무시한 하나하나를 다 파헤칩니다.

만일 당신이 러스트 프로그램을 작성하는 데 있어서 길고 행복한 나날을 바란다면, 당장 뒤돌아서서 이 책을 봤다는 것도 잊어버리세요. 이것이 필수적인 것은 아닙니다. 
그러나 당신이 *비안전한* 코드를 쓰려고 하거나 - 아니면 그냥 언어의 속을 파 보고 싶다면 - 이 책은 유용한 정보를 많이 담고 있습니다.

*[러스트 프로그래밍 언어][trpl]* 와 다르게, 상당한 양의 사전 지식을 갖추고 있다고 가정하겠습니다. 특별히, 당신은 기본적인 시스템 프로그래밍과 러스트에 익숙해야 합니다. 이 주제들이 익숙하지 않다면, [기본 책][trpl] 을 먼저 읽으셔야 할 겁니다. 그렇긴 하지만, 그 책을 읽었다고 가정하진 않을 것이고, 필요하다고 여기는 부분에서는 때때로 기본을 다시 다질 것입니다. 원하시면 바로 이 책으로 건너뛰어도 됩니다 - 모든 것을 처음부터 설명하지는 않을 거라는 것만 알아 두세요.

이 책은 주로 높은 수준에서 [러스트 언어 참조서(영문)][ref] 와 함께 가는 용도로 존재합니다. 참조서가 언어의 모든 부분의 문법과 의미를 자세하게 알기 위해 존재한다면, 러스토노미콘은 이런 부분들을 어떻게 짜맞추어 쓰느냐, 그리고 그러는 동안 부딪힐 난관들을 조명하기 위해 존재합니다. 

참조서는 레퍼런스, 소멸자, 그리고 되감기에 대한 문법과 의미를 말해 주겠지만, 그들을 결합하는 것이 어떻게 프로그램이 예외에도 견딜 수 있게 하는 데에 문제를 가져다 줄 수 있는지, 혹은 그 문제들을 어떻게 해결해야 하는지를 말해 주지는 않을 겁니다.



It should be noted that we haven't synced The Rustnomicon and The Reference well, so they may have duplicate content.
In general, if the two documents disagree, The Reference should be assumed to be correct (it isn't yet considered normative, it's just better maintained).

Topics that are within the scope of this book include: the meaning of (un)safety, unsafe primitives provided by the language and standard library, techniques for creating safe abstractions with those unsafe primitives, subtyping and variance, exception-safety (panic/unwind-safety), working with uninitialized memory, type punning, concurrency, interoperating with other languages (FFI), optimization tricks, how constructs lower to compiler/OS/hardware primitives, how to **not** make the memory model people angry, how you're **going** to make the memory model people angry, and more.

The Rustonomicon is not a place to exhaustively describe the semantics and guarantees of every single API in the standard library, nor is it a place to exhaustively describe every feature of Rust.

Unless otherwise noted, Rust code in this book uses the Rust 2021 edition.

[trpl]: https://doc.rust-kr.org
[ref]: https://doc.rust-lang.org/reference/index.html
