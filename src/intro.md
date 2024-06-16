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

*[러스트 프로그래밍 언어][trpl]* 와 다르게, 어느 정도의 준비 지식은 갖추고 있다고 가정하겠습니다. 

Unlike *[The Rust Programming Language][trpl]*, we will be assuming considerable prior knowledge.
In particular, you should be comfortable with basic systems programming and Rust.
If you don't feel comfortable with these topics, you should consider reading [The Book][trpl] first.
That said, we won't assume you have read it, and we will take care to occasionally give a refresher on the basics where appropriate.
You can skip straight to this book if you want; just know that we won't be explaining everything from the ground up.

This book exists primarily as a high-level companion to [The Reference][ref].
Where The Reference exists to detail the syntax and semantics of every part of the language, The Rustonomicon exists to describe how to use those pieces together, and the issues that you will have in doing so.

The Reference will tell you the syntax and semantics of references, destructors, and unwinding, but it won't tell you how combining them can lead to exception-safety issues, or how to deal with those issues.

It should be noted that we haven't synced The Rustnomicon and The Reference well, so they may have duplicate content.
In general, if the two documents disagree, The Reference should be assumed to be correct (it isn't yet considered normative, it's just better maintained).

Topics that are within the scope of this book include: the meaning of (un)safety, unsafe primitives provided by the language and standard library, techniques for creating safe abstractions with those unsafe primitives, subtyping and variance, exception-safety (panic/unwind-safety), working with uninitialized memory, type punning, concurrency, interoperating with other languages (FFI), optimization tricks, how constructs lower to compiler/OS/hardware primitives, how to **not** make the memory model people angry, how you're **going** to make the memory model people angry, and more.

The Rustonomicon is not a place to exhaustively describe the semantics and guarantees of every single API in the standard library, nor is it a place to exhaustively describe every feature of Rust.

Unless otherwise noted, Rust code in this book uses the Rust 2021 edition.

[trpl]: https://doc.rust-kr.org
[ref]: ../reference/index.html
