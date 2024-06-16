# 러스토노미콘

<div class="warning">

경고:
이 책은 미완성입니다.
모든 것을 문서화하고, 예전에는 맞았지만 지금은 아닌 내용들을 고치는 데는 시간이 걸립니다.
[이슈 트래커]에 없는 기능이나, 예전에는 맞았지만 지금은 아닌 내용들을 보실 수 있고, 아직 보고되지 않은 실수나 생각들이 있다면 거기에 얼마든지 새로운 이슈를 열어 주세요.

</div>

[이슈 트래커]: https://github.com/nomicon-kr/nomicon-kr.github.io/issues

## 비안전 러스트의 흑마법들

> THE KNOWLEDGE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF UNLEASHING INDESCRIBABLE HORRORS THAT SHATTER YOUR PSYCHE AND SET YOUR MIND ADRIFT IN THE UNKNOWABLY INFINITE COSMOS.

The Rustonomicon digs into all the awful details that you need to understand when writing Unsafe Rust programs.

Should you wish a long and happy career of writing Rust programs, you should turn back now and forget you ever saw this book.
It is not necessary.
However if you intend to write unsafe code — or just want to dig into the guts of the language — this book contains lots of useful information.

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

[trpl]: ../book/index.html
[ref]: ../reference/index.html
