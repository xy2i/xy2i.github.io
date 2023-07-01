+++
title = "Rust compiler-as-a-library"
date = 2022-07-01
+++

What if you could extend Rust with your own lints, compiler errors, custom
standard library and semantics?

For example, using [annotations in the Linux kernel][klint] to determine atomic
context violations, or writing [Postgres functions][plrustc] in Rust.

Well, what if I told you `rustc`, the Rust compiler, is a library, and you can
do that without forking the compiler?

In this post, we'll go compare them
to their C/C++ version, why you would use or not use them and how they work
at a high level.

# From compiler plugin to compiler driver

Both [Clang] and [GCC] have their own version of what they call "compiler
plugins". These let you write code that extend the compiler, and are nice
because you don't have to change compiler code.

Rust's approach is a bit different.

Rust provides the entire compiler API to developers. Everything is public, so
you can use [all the compiler crates][ccrates] freely, as a library. Your
program becomes a wrapper over the Rust compiler.

One use for this are *tools* that you can run over a Rust codebase:
- [Clippy] (and the lesser known [Dylint]) wrap the compiler to add lints.
- [Miri] runs against Rust's control graph and detects [undefined behavior].
- [cargo-doc-coverage] gives the amount of documented items in a crate.

A more niche use is for Rust compiler authors to prototype quickly against the
Rust codebase without having to maintain a separate branch or fork: for example,
[Polonius] implements a next generation borrow checker, the part of Rust that
checks lifetimes.

Finally, these can be also used as custom compilers for a specific project,
replacing `rustc` and adding extra analysis or behavior. For example, `klint` in
Rust for Linux tracks [preemption count] at compile time, by reading the [CFG],
and erroring otherwise (check out their full [blog post][klint]!):

```rust
error: this call expects the preemption count to be 0
  --> samples/rust/rust_sync.rs:76:17
   |
76 |  kernel::delay::coarse_sleep(core::time::Duration::from_secs(1));
   |  ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
   |
   = note: but the possible preemption count at this point is 1
```

# Benefits and drawbacks

The main drawback is that nothing is stable. Apart from the
`rustc-driver` and associated APIs, any internal APIs you depend on may break
between compiler updates.

Of course, maintaining a compiler driver also has significant complexity, not
only for implementers but
especially if you want to keep it up to date.

If you don't have esoteric needs or you don't fit one or the above use cases,
see if [proc macros] can solve them.

# Cool, how do I make one?

Because drivers are forever unstable, any code I give here is likely to be
outdated in a year's time. Instead I'll stick to high level recommendations.

**To learn how the whole system works**, check out the [Peeking at
compiler-internal data][talk] talk ([slides here][slides]).

**For reference**, check out the [rustc dev guide][rustc-dev], the official Rust
compiler doc. In particuliar the sections on [rustc-driver][driver-doc] and
[queries] are of great interest.

**For reference implementations**, the highest quality one is probably
[plrustc]. Of note is their [build script] which outputs a statically linked
`rustc` without dependencies. Make sure to [use `RUSTC_BOOTSTRAP`][1] in your
`.cargo/config` so your driver [can depend on stable versions][2].

# Summary

Rust's compiler-as-a-library approach offers you the ability to extend the Rust
compiler with custom lints, compiler errors, and even create custom standard
libraries and semantics.

While there are benefits to this approach, like making special-purpose tools and
custom compilers, it also comes with drawbacks, most notably the lack of
stability and the need for ongoing maintenance to keep up with new Rust
versions.

But if you have specific needs, it opens up new possibilities for extending and customizing Rust.

[clang]:https://clang.llvm.org/doxygen/group__CINDEX.html
[gcc]:https://gcc.gnu.org/wiki/plugins
[klint]: https://www.memorysafety.org/blog/gary-guo-klint-rust-tools/
[plrustc]: https://github.com/tcdi/plrust/tree/main/plrustc
[ccrates]: https://doc.rust-lang.org/nightly/nightly-rustc/
[driver]: https://rustc-dev-guide.rust-lang.org/rustc-driver.html
[macros]: https://xy2.dev/blog/simple-proc-macro/
[clippy]: https://github.com/rust-lang/rust-clippy
[dylint]: https://github.com/trailofbits/dylint
[miri]: https://github.com/rust-lang/miri
[undefined behavior]: https://en.wikipedia.org/wiki/Undefined_behavior
[cargo-doc-coverage]: https://michael-f-bryan.github.io/cargo-metrics/introduction.html
[polonius]:https://github.com/rust-lang/polonius
[cfg]:https://en.wikipedia.org/wiki/Control-flow_graph
[preemption count]: https://lwn.net/Articles/831678/
[proc macros]: https://doc.rust-lang.org/reference/procedural-macros.html
[talk]: https://www.youtube.com/watch?v=SKmd5A-1cSE
[slides]: https://hackmd.io/RiztubvfT4eOk4-4nM8Y7Q?both
[rustc-dev]: https://rustc-dev-guide.rust-lang.org/
[driver-doc]: https://rustc-dev-guide.rust-lang.org/rustc-driver.html
[queries]: https://rustc-dev-guide.rust-lang.org/query.html
[plrustc]: https://github.com/tcdi/plrust/tree/main/plrustc
[build script]: https://github.com/tcdi/plrust/blob/main/plrustc/build.sh

[1]:https://github.com/tcdi/plrust/blob/main/plrustc/.cargo/config.toml
[2]: https://github.com/tcdi/plrust/blob/main/plrustc/rust-toolchain.toml