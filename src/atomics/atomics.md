# Atomics

Rust pretty blatantly just inherits the memory model for atomics from C++20. This is not
due to this model being particularly excellent or easy to understand. Indeed,
this model is quite complex and known to have [several flaws][C11-busted].
Rather, it is a pragmatic concession to the fact that *everyone* is pretty bad
at modeling atomics. At very least, we can benefit from existing tooling and
research around the C/C++ memory model.
(You'll often see this model referred to as "C/C++11" or just "C11". C just copies
the C++ memory model; and C++11 was the first version of the model but it has
received some bugfixes since then.)

Trying to fully explain the model in this book is fairly hopeless. It's defined
in terms of madness-inducing causality graphs that require a full book to
properly understand in a practical way. If you want all the nitty-gritty
details, you should check out the [C++ specification][C++-model] —
note that Rust atomics correspond to C++’s `atomic_ref`, since Rust allows
accessing atomics via non-atomic operations when it is safe to do so.
In this section we aim to give an informal overview of the topic to cover the
common problems that Rust developers face.

## Motivation

The C++ memory model is very large and confusing with lots of seemingly
arbitrary design decisions. To understand the motivation behind this, it can
help to look at what got us in this situation in the first place. There are
three main factors at play here:

1. Users of the language, who want fast, cross-platform code;
2. compilers, who want to optimize code to make it fast;
3. and the hardware, which is ready to unleash a wrath of inconsistent chaos on
  your program at a moment's notice.

The memory model is fundamentally about trying to bridge the gap between these
three, allowing users to write the algorithms they want while the compiler and
hardware perform the arcane magic necessary to make them run fast.

### Compiler Reordering

Compilers fundamentally want to be able to do all sorts of complicated
transformations to reduce data dependencies and eliminate dead code. In
particular, they may radically change the actual order of events, or make events
never occur! If we write something like:

<!-- ignore: simplified code -->
```rust,ignore
x = 1;
y = 3;
x = 2;
```

The compiler may conclude that it would be best if your program did:

<!-- ignore: simplified code -->
```rust,ignore
x = 2;
y = 3;
```

This has inverted the order of events and completely eliminated one event.
From a single-threaded perspective this is completely unobservable: after all
the statements have executed we are in exactly the same state. But if our
program is multi-threaded, we may have been relying on `x` to actually be
assigned to 1 before `y` was assigned. We would like the compiler to be
able to make these kinds of optimizations, because they can seriously improve
performance. On the other hand, we'd also like to be able to depend on our
program *doing the thing we said*.

### Hardware Reordering

On the other hand, even if the compiler totally understood what we wanted and
respected our wishes, our hardware might instead get us in trouble. Trouble
comes from CPUs in the form of memory hierarchies. There is indeed a global
shared memory space somewhere in your hardware, but from the perspective of each
CPU core it is *so very far away* and *so very slow*. Each CPU would rather work
with its local cache of the data and only go through all the anguish of
talking to shared memory only when it doesn't actually have that memory in
cache.

After all, that's the whole point of the cache, right? If every read from the
cache had to run back to shared memory to double check that it hadn't changed,
what would the point be? The end result is that the hardware doesn't guarantee
that events that occur in some order on *one* thread, occur in the same
order on *another* thread. To guarantee this, we must issue special instructions
to the CPU telling it to be a bit less smart.

For instance, say we convince the compiler to emit this logic:

```text
initial state: x = 0, y = 1

THREAD 1        THREAD 2
y = 3;          if x == 1 {
x = 1;              y *= 2;
                }
```

Ideally this program has 2 possible final states:

* `y = 3`: (thread 2 did the check before thread 1 completed)
* `y = 6`: (thread 2 did the check after thread 1 completed)

However there's a third potential state that the hardware enables:

* `y = 2`: (thread 2 saw `x = 1`, but not `y = 3`, and then overwrote `y = 3`)

It's worth noting that different kinds of CPU provide different guarantees. It
is common to separate hardware into two categories: strongly-ordered and
weakly-ordered, where strongly-ordered hardware implements weak orderings like
`Relaxed` using strong orderings like `Acquire`, while weakly-ordered hardware
makes use of the optimization potential that weak orderings like `Relaxed` give.
Most notably, x86/64 provides strong ordering guarantees, while ARM provides
weak ordering guarantees. This has two consequences for concurrent programming:

* Asking for stronger guarantees on strongly-ordered hardware may be cheap or
  even free because they already provide strong guarantees unconditionally.
  Weaker guarantees may only yield performance wins on weakly-ordered hardware.

* Asking for guarantees that are too weak on strongly-ordered hardware is
  more likely to *happen* to work, even though your program is strictly
  incorrect. If possible, concurrent algorithms should be tested on
  weakly-ordered hardware.

[C11-busted]: http://plv.mpi-sws.org/c11comp/popl15.pdf
[C++-model]: https://en.cppreference.com/w/cpp/atomic/memory_order
