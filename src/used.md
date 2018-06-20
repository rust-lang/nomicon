## #[used]

The #[used] attribute forces the *compiler* to keep a `static` variable in the output object file.
This is useful for placing data at specific memory locations: for example, placing the vector table
(interrupt table) in the memory location required by the ARM Cortex-M ABI: `0x0000_0000`.

It's important to note that, on its own, `#[used]` has no effect on the behavior of the *linker*.
That is the linker is free to drop a variable marked as `#[used]` when linking object files; thus,
*a `#[used]` variable may not necessarily make it into the final binary / executable*.

The guaranteed way to keep a variable in the final binary is to pair `#[used]` with the `EXTERN`
linker script command. Linkers are lazy: once they have found all the symbols needed by the first /
root object file they will stop looking at the other object files in their list of arguments.
`EXTERN` forces the linker to look into the other object files until it finds the `EXTERN`-ed
symbol.

Below is an example that shows that both `#[used]` and `EXTERN` are required to keep `static`
variables in an executable.

``` rust
#![feature(panic_implementation)]
#![feature(used)]
#![no_main]
#![no_std]

use core::panic::PanicInfo;

// `#[no_mangle]` makes it easier to `EXTERN` this variable / symbol in the
// linker script
// `pub` is required to make this symbol *external*; the linker ignores
// *internal* symbols when looking for an `EXTERN`-ed symbol
#[no_mangle]
#[used]
pub static FOO: u32 = 0;

// kept by the compiler, but dropped by the linker
#[used]
static BAR: u32 = 0;

// dropped by the compiler
#[allow(dead_code)]
static BAZ: u32 = 0;

#[panic_implementation]
fn panic(_: &PanicInfo) -> ! {
    loop {}
}
```

``` console
$ echo 'EXTERN(FOO);' > link.x

$ rustc -O -C lto \
    -C panic=abort -C relocation-model=static \
    -C link-arg=-nostartfiles -C link-arg=-Wl,-Tlink.x \
    --emit=link,obj \
    foo.rs

$ nm -C foo.o
0000000000000000 R FOO
0000000000000000 r foo::BAR

$ nm -C foo
0000000000000024 R FOO
0000000000000028 r _GLOBAL_OFFSET_TABLE_
```
