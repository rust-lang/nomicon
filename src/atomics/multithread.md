# Multithreaded Execution

When you write Rust code to run on your computer, it may surprise you but you’re
not actually writing Rust code to run on your computer — instead, you’re writing
Rust code to run on the _abstract machine_ (or AM for short). The abstract
machine, to be contrasted with the physical machine, is an abstract
representation of a theoretical computer: it doesn’t actually exist _per se_,
but the combination of a compiler, target architecture and target operating
system is capable of emulating a subset of its possible behaviours.

The Abstract Machine has a few properties that are essential to understand:
1. It is architecture and OS-independent. The Abstract Machine doesn’t care
	whether you’re on x86_64 or iOS or a Nintendo 3DS, the rules are the same
	for everyone. This enables you to write code without having to think about
	what the underlying system does or how it does it, as long as you obey the
	Abstract Machine’s rules you know you’ll be fine.
1. It is the lowest common denominator of all supported computer systems. This
	means it is allowed to result in executions no sane computer would actually
	generate in real life. It is also purposefully built with forward
	compatibility in mind, giving compilers the opportunity to make better and
	more aggressive optimizations in the future. As a result, it can be quite
	hard to test code, especially if you’re on a system that exploits fewer of
	the AM’s allowed semantics, so it is highly recommended to utilize tools
	that intentionally produce these executions like [Loom] and [Miri].
1. Its model is highly formalized and not representative of what goes on
	underneath. Because C++ needs to be defined by a formal specification and
	not just hand-wavy rules about “this is what is allowed and this is what
	isn’t”, the Abstract Machine defines things in a very mathematical and,
	well, _abstract_, way; instead of saying things like “the compiler is
	allowed to do X” it will find a way to define the system such that the
	compiler’s ability to do X simply follows as a natural consequence. This
	makes it very elegant and keeps the mathematicians happy, but you should
	keep in mind that this is not how computers actually function, it is merely
	a representation of it.

With that out of the way, let’s look into how the C++20 Abstract Machine is
actually defined.

The first important thing to understand is that **the abstract machine has no
concept of time**. You might expect there to be a single global ordering of
events across the program where each happens at the same time or one after the
other, but under the abstract model no such ordering exists; instead, a possible
execution of the program must be treated as a single event that happens
instantaneously. There is never any such thing as “now”, or a “latest value”,
and using that terminology will only lead you to more confusion. Of course, in
reality there does exist a concept of time, but you must keep in mind that
you’re not programming for the hardware, you’re programming for the AM.

However, while no global ordering of operations exists _between_ threads, there
does exist a single total ordering _within_ each thread, which is known as its
_sequence_. For example, given this simple Rust program:

```rust
println!("A");
println!("B");
```

its sequence during one possible execution can be visualized like so:

```text
╭───────────────╮
│ println!("A") │
╰───────╥───────╯
╭───────⇓───────╮
│ println!("B") │
╰───────────────╯
```

That double arrow in between the two boxes (`⇒`) represents that the second
statement is _sequenced-after_ the first (and similarly the first statement is
_sequenced-before_ the second). This is the strongest kind of ordering guarantee
between any two operations, and only comes about when those two operations
happen one after the other and on the same thread.

If we add a second thread to the mix:

```rust
// Thread 1:
println!("A");
println!("B");
// Thread 2:
eprintln!("01");
eprintln!("02");
```

it will simply coexist in parallel, with each thread getting its own independent
sequence:

```text
    Thread 1              Thread 2
╭───────────────╮    ╭─────────────────╮
│ println!("A") │    │ eprintln!("01") │
╰───────╥───────╯    ╰────────╥────────╯
╭───────⇓───────╮    ╭────────⇓────────╮
│ println!("B") │    │ eprintln!("02") │
╰───────────────╯    ╰─────────────────╯
```

We can say that the prints of `A` and `B` are _unsequenced_ with regard to the
prints of `01` and `02` that occur in the second thread, since they have no
sequenced-before arrows connecting the boxes together.

Note that these diagrams are **not** a representation of multiple things that
_could_ happen at runtime — instead, this diagram describes exactly what _did_
happen when the program ran once. This distinction is key, because it highlights
that even the lowest-level representation of a program’s execution does not have
a global ordering between threads; those two disconnected chains are all there
is.

Now let’s make things more interesting by introducing some shared data, and have
both threads read it.

```rust
// Initial state
let data = 0;
// Thread 1:
println!("{data}");
// Thread 2:
eprintln!("{data}");
```

Each memory location, similarly to threads, can be shown as another column on
our diagram, but holding values instead of instructions, and each access (read
or write) manifests as a line from the instruction that performed the access to
the associated value in the column. So this code can produce (and is in fact
guaranteed to produce) the following execution:

```text
Thread 1     data     Thread 2
╭──────╮    ┌────┐    ╭──────╮
│ data ├╌╌╌╌┤  0 ├╌╌╌╌┤ data │
╰──────╯    └────┘    ╰──────╯
```

That is, both threads read the same value of `0` from `data`, and the two
operations are unsequenced — they have no relative ordering between them.

That’s reads done, so we’ll look at the other kind of data access next: writes.
We’ll also return to a single thread for now, just to keep things simple.

```rust
let mut data = 0;
data = 1;
```

Here, we have a single variable that the main thread writes to once — this means
that in its lifetime, it holds two values, first `0`, and then `1`.
Diagrammatically, this code’s execution can be represented like so:

```text
 Thread 1        data
╭───────╮       ┌────┐
│  = 1  ├╌╌╌┐   │  0 │
╰───────╯   ├╌╌╌┼╌╌╌╌┤
            └╌╌╌┼╌╌╌╌┤
                │  1 │
                └────┘
```

Note the use of dashed padding in between the values of `data`’s column. Those
spaces won’t ever contain a value, but they’re used to represent an
unsynchronized (non-atomic) write — it is garbage data and attempting to read it
would result in a data race.

Now let’s put all of our knowledge thus far together, and make a program both
that reads _and_ writes data — woah, scary!

```rust
let mut data = 0;
data = 1;
println!("{data}");
data = 2;
```

Working out executions of code like this is rather like solving a Sudoku puzzle:
you must first lay out all the facts that you know, and then fill in the blanks
with logical reasoning. The initial information we’ve been given is both the
initial value of `data` and the sequential order of Thread 1; we also know that
over its lifetime, `data` takes on a total of three different values that were
caused by two different non-atomic writes. This allows us to start drawing out
some boxes:

```text
 Thread 1        data
╭───────╮       ┌────┐
│  = 1  ├╌?     │  0 │
╰───╥───╯     ?╌┼╌╌╌╌┤
╭───⇓───╮     ?╌┼╌╌╌╌┤
│  data ├╌?     │  ? │
╰───╥───╯     ?╌┼╌╌╌╌┤
╭───⇓───╮     ?╌┼╌╌╌╌┤
│  = 2  ├╌?     │  ? │
╰───────╯       └────┘
```

We know all of those lines need to be joined _somewhere_, but we don’t quite
know _where_ yet. This is where we need to bring in our first rule, a rule that
universally governs all accesses to every location in memory:

> From the point at which the access occurs, find every other point that can be
> reached by following the reverse direction of arrows, then for each one of
> those, take a single step across every line that connects to the relevant
> memory location. **It is not allowed for the access to read or write any value
> that appears above any one of these points**.

In our case, there are two potential executions: one, where the first write
corresponds to the first value in `data`, and two, where the first write
corresponds to the second value in `data`. Considering the second case for a
moment, it would also force the second write to correspond to the first
value in `data`. Therefore its diagram would look something like this:

```text
 Thread 1        data
╭───────╮       ┌────┐
│  = 1  ├╌╌┐    │  0 │
╰───╥───╯  ┊ ┌╌╌┼╌╌╌╌┤
╭───⇓───╮  ┊ ├╌╌┼╌╌╌╌┤
│  data ├╌?┊ ┊  │  2 │
╰───╥───╯  ├╌┼╌╌┼╌╌╌╌┤
╭───⇓───╮  └╌┼╌╌┼╌╌╌╌┤
│  = 2  ├╌╌╌╌┘  │  1 │
╰───────╯       └────┘
```

However, that second line breaks the rule we just established! Following up the
arrows from the third operation in Thread 1, we reach the first operation, and
from there we can take a single step to reach the space in between the `2` and
the `1`, which excludes the third access from writing any value above that point
— including the `2` that it is currently writing!

So evidently, this execution is no good. We can therefore conclude that the only
possible execution of this program is the other one, in which the `1` appears
above the `2`:

```text
 Thread 1     data
╭───────╮     ┌────┐
│  = 1  ├╌╌┐  │  0 │
╰───╥───╯  ├╌╌┼╌╌╌╌┤
╭───⇓───╮  └╌╌┼╌╌╌╌┤
│  data ├╌?   │  1 │
╰───╥───╯  ┌╌╌┼╌╌╌╌┤
╭───⇓───╮  ├╌╌┼╌╌╌╌┤
│  = 2  ├╌╌┘  │  2 │
╰───────╯     └────┘
```

Now to sort out the read operation in the middle. We can use the same rule as
before to trace up to the first write and rule out us reading either the `0`
value or the garbage that exists between it and `1`, but how do we choose
between the `1` and the `2`? Well, as it turns out there is a complement to the
rule we already defined which gives us the exact answer we need:

> From the point at which the access occurs, find every other point that can be
> reached by following the _forward_ direction of arrows, then for each one of
> those, take a single step across every line that connects to the relevant
> memory location. **It is not allowed for the access to read or write any value
> that appears below any one of these points**.

Using this rule, we can follow the arrow downwards and then across and finally
rule out `2` as well as the garbage before it. This leaves us with exactly _one_
value that the read operation can return, and exactly one possible execution
guaranteed by the Abstract Machine:

```text
 Thread 1      data
╭───────╮     ┌────┐
│  = 1  ├╌╌┐  │  0 │
╰───╥───╯  ├╌╌┼╌╌╌╌┤
╭───⇓───╮  └╌╌┼╌╌╌╌┤
│  data ├╌╌╌╌╌┤  1 │
╰───╥───╯  ┌╌╌┼╌╌╌╌┤
╭───⇓───╮  ├╌╌┼╌╌╌╌┤
│  = 2  ├╌╌┘  │  2 │
╰───────╯     └────┘
```

These two rules combined make up the more generalized rule known as _coherence_,
which is put in place to guarantee that a thread will never see a value earlier
than the last one it read or later than a one it will in future write. Coherence
is basically required for any program to act in a sane way, so luckily the C++20
standard guarantees it as one of its most fundamental principles.

You might be thinking that all this has been is the longest, most convoluted
explanation ever of the most basic intuitive semantics of programming — and
you’d be absolutely right. But it’s essential to grasp these fundamentals,
because once you have this model in mind, the extension into multiple threads
and the complicated semantics of real atomics becomes completely natural.

[Loom]: https://docs.rs/loom
[Miri]: https://github.com/rust-lang/miri
