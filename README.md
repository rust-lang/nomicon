# 러스토노미콘

심오하고 비안전[comm1]한 러스트 프로그래밍의 흑마법들

별명은 "노미콘"

## 주의: 이것은 수정중인 문서이고, 심각한 오류들을 포함할 수 있습니다.

> 내가 바랐던 프로그램들 대신, 살떨리는 어둠과 표현할 수 없는 외로움만이 있었다. 그리고 난 마침내 보고야 말았다.
  아무도 그 앞에서 감히 숨도 쉬지 못한 두려운 진실, 속삭일 수조차 없는 비밀들 중의 비밀을 보고야 만 것이다.
  이 돌과 끼긱거리는 소리로 이루어진 언어가 러스트의 의식적인 후계가 아니라는 것이었다. 런던은 옛날 정겨운
  런던이었고 파리도 그랬지만, 이 언어는 아니었다. 이것은 꽤나 비(非)안전했고,
  그 뻗어있는 몸은 거의 미라가 되어 있었고 컴파일할 때는 없었던, 움직이는 요상한 것들로 들끓고 있었다.

This book digs into all the awful details that are necessary to understand in
order to write correct Unsafe Rust programs. Due to the nature of this problem,
it may lead to unleashing untold horrors that shatter your psyche into a billion
infinitesimal fragments of despair.

## Requirements

Building the Nomicon requires [mdBook]. To get it:

[mdBook]: https://github.com/rust-lang/mdBook

```bash
cargo install mdbook
```

### `mdbook` usage

To build the Nomicon use the `build` sub-command:

```bash
mdbook build
```

The output will be placed in the `book` subdirectory. To check it out, open the
`index.html` file in your web browser. You can pass the `--open` flag to `mdbook
build` and it'll open the index page in your default browser (if the process is
successful) just like with `cargo doc --open`:

```bash
mdbook build --open
```

There is also a `test` sub-command to test all code samples contained in the book:

```bash
mdbook test
```

### `linkcheck`

We use the `linkcheck` tool to find broken links.
To run it locally:

```sh
curl -sSLo linkcheck.sh https://raw.githubusercontent.com/rust-lang/rust/master/src/tools/linkchecker/linkcheck.sh
sh linkcheck.sh --all nomicon
```

## Contributing

Given that the Nomicon is still in a draft state, we'd love your help! Please
feel free to open issues about anything, and send in PRs for things you'd like
to fix or change. If your change is large, please open an issue first, so we can
make sure that it's something we'd accept before you go through the work of
getting a PR together.


[comm1]: # "a comment"
