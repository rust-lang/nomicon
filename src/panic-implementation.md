## #[panic_implementation]

`#[panic_implementation]` is used to define the behavior of `panic!` in `#![no_std]` applications.
The `#[panic_implementation]` attribute must be applied to a function with signature `fn(&PanicInfo)
-> !` and such function must appear *once* in the dependency graph of a binary / dylib / cdylib
crate. The API of `PanicInfo` can be found in the [API docs].

[API docs]: https://doc.rust-lang.org/nightly/core/panic/struct.PanicInfo.html

Given that `#![no_std]` applications have no *standard* output and that some `#![no_std]`
applications, e.g. embedded applications, need different panicking behaviors for development and for
release it can be helpful to have panic crates, crate that only contain a `#[panic_implementation]`.
This way applications can easily swap the panicking behavior by simply linking to a different panic
crate.

Below is shown an example where an application has a different panicking behavior depending on
whether is compiled using the dev profile (`cargo build`) or using the release profile (`cargo build
--release`).

``` rust
// crate: panic-semihosting -- log panic message to the host stderr using semihosting

#![feature(core_intrinsics)]
#![feature(panic_implementation)]
#![no_std]

#[panic_implementation]
fn panic(info: &PanicInfo) -> ! {
    let host_stderr = /* .. */;

    writeln!(host_stderr, "{}", info).ok();

    core::intrinsics::breakpoint();

    loop {}
}
```

``` rust
// crate: panic-abort -- abort on panic!

#![feature(core_intrinsics)]
#![feature(panic_implementation)]
#![no_std]

#[panic_implementation]
fn panic(info: &PanicInfo) -> ! {
    unsafe { core::intrinsics::abort() }
}
```

``` rust
// crate: app

#![no_std]

// dev profile
#[cfg(debug_assertions)]
extern crate panic_semihosting;

// release profile
#[cfg(not(debug_assertions))]
extern crate panic_abort;

// omitted: other `extern crate`s

fn main() {
    // ..
}
```
