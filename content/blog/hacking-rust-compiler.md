+++
title = 'Hacking the Rust compiler for fun and profit'
date = 2022-06-28
+++

> Summary: An explaination of how to implement practical procedural macros in
> the [Rust][rust] programming language. Explains the
> different types of macros, then shows an
> implementation of a procedural macro following best practices, focusing
> on testing and ergonomics.
> *Assumes some familiarity with Rust.*

In [Rust][rust], a [macro][macros] is code that writes other code, enabling
[metaprogramming][metaprogramming].

Rust uses macros to add features on top of the core language, like
the [`println!()`][print] macro, providing variadic arguments and string
interpolation. They can also be used for code generation,
removing the need for error-prone copy-and-paste programming. Macros are even
powerful enough to create [domain-specific languages][dsl], or [type-checked
SQL at compile-time](https://github.com/launchbadge/sqlx).

# When should you use or not use procedural macros?

To be clear - **most of the time, you don't need procedural macros**.
- If you want to use a declarative macro in `#[my_macro]` or `#[derive(MyMacro)`
position, then use the [`macro_rules_attribute`][macrorulesattr] library.
- If you need more complicated logic in your declarative macros, take a look at
[The Little Book of Rust Macros][macrobook] if you can use their techniques.

Declarative macros, the classic form of macros, rely on purely declarative
logic. They look something like this:

```rust
macro_rules! expr {
    (eval $a:expr + $b: expr) => {
        println!("{}", $a + $b);
    }
}

expr!(eval 1 + 2);
// evaluates to
println!("{}", 1 + 2);
```

They use their own separate, declarative language. But declarative macros are
[not without warts][3] and are pretty hard to understand if they get bigger and
bigger.

Rust introducted procedural macros in [Rust
1.30][1] for custom `#[attributes]`. Unlike declarative macros,
proc macros use Rust directly, so you're writing the macros in the same
language, which makes it much easier to scale. For this reason, you should
consider proc macros if you need to deal with a large macro.

But procedural macros have drawbacks. First, to do anything practical, you'll
likely need the [`syn`][2] library, notorious for long build times. And second,
there are still many footguns left in proc macros today. This guide will
explain the ones I know of.

Note that once you do invest into procedural macros, it's easier to write
the next ones, because a fair bit of the cost is up-front.

Overall, consider procedural macros if one of these applies:
- You need to develop a large, complex macro and are hitting declarative macro
  limitations.
- Your declarative macro has grown too large and hard to understand, and you
  need to extend it further.
- You need complex procedural logic in your macro.

# A short primer

There are [three different
types](https://doc.rust-lang.org/reference/procedural-macros.html) of
procedural macros:
- function like: `my_macro!(...)`
- derive macros: `#[derive(MyMacro)]`
- attribute macros: `#[my_macro(arguments)]`

These types differ only by their syntax when invoked. Function-like, as their
name suggests, act like your usual declarative macros.

Derive macros can apply in any `#[derive(..)]` block. This makes them most
interesting for macros that need to generate code on structs.

Attributes macros are a more general version, which can be applied to any
"item" in Rust: structs, traits, functions, types and so on. The
[`tokio::main`](https://docs.rs/tokio/latest/tokio/attr.main.html) macro is a
good example of why to use this kind of macro: it decorates a main function,
without having to add an extra level of indentation. Compare with the function
like syntax:

```rust
tokio_main! { // extra indentation
    fn main() {
        println!("hello world")
    }
}
```

The most important difference is procedural macros operate on tokens, not the
AST directly, so the proc macro has to reimplement parsing of Rust source code.

This is most often done with [`syn`][2].

Procedural macros are *compiler extensions*: they are currently run as part of
the compiler. This is the main reason why they have to be declared in a
[separate
crate](https://stackoverflow.com/questions/56713877/why-do-proc-macros-have-to-be-defined-in-proc-macro-crate).

Finally, proc macros are not fully hygienic ([it's
complicated](https://veykril.github.io/tlborm/proc-macros/hygiene.html)). Most
importantly, like declarative macros, they are not path hygenic, so you need to
use full paths, eg. `std::thread::spawn`, or include `use` statements if you
can.

Use [`cargo-expand`](https://github.com/dtolnay/cargo-expand) to manually test
and see the output of a macro.

## Creating a proc macro

As we have to declare the proc macro in a separate crate, and assuming we have
some code already, we'll need at least two crates: one for your proc macros and
another one for the rest of your code.

I recommend using a [workspace][workspace] to organise this. Workspaces are an
integrated way within Cargo to have multiple crates. If you use other systems
to build your macro code, take a look at the proc macro Rust rules for [gn][4]
or [Bazel][5].

Here's an example with two crates, your library crate and the proc macro crate.
You'll need to create the workspace layout yourself as there is no tool to do
so automatically yet.

```
my_project/             // new folder
├─ my_crate/
│  ├─ Cargo.toml
│  ├─ lib.rs
├─ my_crate_proc_macro/ // macro crate (cargo new --lib my_crate_proc_macro)
│  ├─ Cargo.toml
│  ├─ lib.rs
├─ Cargo.toml           // new file
```

Now to declare the proc macro:

```toml
# my_crate_proc_macro/Cargo.toml
[lib]
proc-macro = true
```

Add your newly declared proc macro crate as a dependency to your main crate's
`Cargo.toml`, so it can use it:

```toml
# my_crate/Cargo.toml
[dependencies]
my_crate_proc_macro = { path = "../my_crate_proc_macro" }
```

Then add the workspace at the top level, above the two crates:

```toml
# Cargo.toml
[workspace]
members = [
    "my_crate",
    "my_crate_proc_macro",
]
```

Here's an example of an function-like proc-macro. Add the `#[proc_macro]`
attribute to a function in your proc macro crate to make it a proc macro.
Each kind of proc macro has a different signature, but they all take and return
[`TokenStream`](https://doc.rust-lang.org/proc_macro/struct.TokenStream.html),
a list of tokens.

```rust
#[proc_macro]
pub fn my_macro(item: proc_macro::TokenStream) -> proc_macro::TokenStream {
    item
}
```
# Foundational crates for best practices proc macros

It's possible to use proc macros without any dependencies, but using these
will save you trouble and headaches over the long run.

## Testing: [`proc_macro2`](https://docs.rs/proc-macro2/*/proc_macro2/)

This crate is a wrapper around the `proc_macro` crate from the compiler, which
you use to create a proc macro. However, `proc_macro` is not usable in
non proc-macro contexts, meaning you can't use it in tests.

`proc_macro2` is a wrapper around the compiler crate. This means, most
importantly for us, that you can use the proc macro in tests.

Many other crates in the ecosystems have separate functions for `proc_macro`
and `proc_macro2`, for example [`syn::parse`][6] vs [`syn::parse2`][7].

Here's an example of how `proc_macro2` can be useful. In this example,
we separate the parts with
`proc_macro` and `proc_macro2`, and test the `proc_macro2` parts. The actual
proc macro does a roundtrip from `proc_macro2` structures and back. (Do not use
the test in this example, see below.)

```rust
#[proc_macro]
pub fn my_macro(item: proc_macro::TokenStream) -> proc_macro::TokenStream {
    my_macro_impl(item.into()).into()
}

fn my_macro_impl(_item: proc_macro2::TokenStream) -> proc_macro2::TokenStream {
    "fn macro_generated() -> u32 {
        42
     }"
    .parse()
    .unwrap()
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn unit() {
        // test for illustrative purposes
        // do not use this test, see `quote` section below
        assert_eq!(
            my_macro_impl("".parse().unwrap()).to_string(),
            "fn macro_generated() -> u32 { 42 }"
        );
    }
}
```

## Creating trees: [`quote`](https://docs.rs/quote/latest/quote/)

In our test, we converted the `TokenStream`s to strings, because they don't
implement `Eq`, so we can't compare them directly.

Can you spot the error in the test?
```rust
assert_eq!(
    my_macro_impl("".parse().unwrap()).to_string(),
    "fn macro_generated() -> u32 { 42 }"
);
```
The proper result is actually `"fn macro_generated () -> u32 { 42 }"`, with a
space between `macro_generated` and the `()`.

To test this macro, we used the string conversion of `TokenStream`, which just
outputs each distinct token with a space between, stripping newlines.

To avoid this, convert to and from a `TokenStream`:

```rust
assert_eq!(
    my_macro_impl("".parse().unwrap()).to_string(),
    "fn macro_generated() -> u32 { 42 }".parse::<proc_macro2::TokenStream>()
        .unwrap().to_string()
);
```

As you can see, this way is pretty error prone. At this point you'd likely make
a helper.

The confusion stems from our using strings as a way to
represent Rust source code, because we must create the `TokenStream` by some
method. Here, we confused a string with code written raw, but not
converted, with one converted from a `TokenStream`.

There's a more convenient way: the third-party `quote` crate, turning any code
passed into a token tree directly. `quote` has a few niceties, including
interpolation and repetition.

Here's the previous example rewritten to use `quote`:

```rust
use quote::quote;

#[proc_macro]
pub fn my_macro(item: proc_macro::TokenStream) -> proc_macro::TokenStream {
    my_macro_impl(item.into()).into()
}

fn my_macro_impl(_item: proc_macro2::TokenStream) -> proc_macro2::TokenStream {
    quote! {
        fn macro_generated() -> u32 {
            42
        }
    }
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn unit() {
        assert_eq!(
            my_macro_impl(quote!("")).to_string(),
            quote! {
                fn macro_generated() -> u32 {
                    42
                }
            }
            .to_string()
        );
    }
}
```

We'll see more of the power of `quote` as we go. Overall, it's more
a convenience than anything, though be careful when testing if you choose not
to use it.

## Parsing Rust: [`syn`](https://docs.rs/syn/2.0.22/syn/)

What if our macro takes some Rust input, and we want to parse it? There are two
choices.

First, we can manually parse the subset of Rust we want. This is doable but
especially tedious. If you choose to go this route, the best way to go is
probably looking at macros not using `syn`, [`doc-comment`'s source
code](https://github.com/GuillaumeGomez/doc-comment/blob/master/src/lib.rs)
could be a good starting point. You save out on the long compile times of
`syn`.

However, this is painful, and brittle in the face of new syntax, so you'll
probably want to use `syn` if you're doing anything pretty complex.

We've talked a lot about tokens so far, but not so much about [syntax
trees](https://en.wikipedia.org/wiki/Abstract_syntax_tree)
. Put
simply, abstract syntax trees are much easier to deal with than tokens, as they
don't need to represent details in the real syntax, like the parentheses and
brackets, as well as the actual symbols you type out in source code.

Here's the abstract syntax tree for a simple `pub fn main() {}`:

```rust
Item::Fn {
    attrs: [],
    vis: Visibility::Public(
        Pub,
    ),
    sig: Signature {
        constness: None,
        asyncness: None,
        unsafety: None,
        abi: None,
        fn_token: Fn,
        ident: Ident {
            sym: main,
            span: bytes(8..12),
        },
        generics: Generics {
            lt_token: None,
            params: [],
            gt_token: None,
            where_clause: None,
        },
        paren_token: Paren,
        inputs: [],
        variadic: None,
        output: ReturnType::Default,
    },
    block: Block {
        brace_token: Brace,
        stmts: [],
    },
}
```
Of course, `syn` can parse any Rust syntax, not just functions.

I recommend using the [AST Explorer](https://astexplorer.net/) and putting Rust
code in. This is an easy way to see what data is available on an `syn` AST.

Here's an example that reverses the name of the function.
For this example, we're using an attribute macro. It takes an extra
argument (the `arguments` in `#[my_macro(arguments)]`).

```rust
use quote::quote;

#[proc_macro_attribute]
pub fn my_macro(
    _attr: proc_macro::TokenStream,
    item: proc_macro::TokenStream,
) -> proc_macro::TokenStream {
    my_macro_impl(_attr.into(), item.into()).into()
}

fn my_macro_impl(
    _attr: proc_macro2::TokenStream,
    item: proc_macro2::TokenStream,
) -> proc_macro2::TokenStream {
    let mut func: syn::ItemFn = match syn::parse2(item.clone()) {
        Ok(it) => it,
        Err(e) => return e.into_compile_error(),
    };

    let ident = func.sig.ident;
    let reversed: String = ident.to_string().chars().rev().collect();
    func.sig.ident = syn::Ident::new(&reversed, ident.span());

    quote! { #func }
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn unit() {
        assert_eq!(
            my_macro_impl(
                quote!(""),
                quote! {
                    fn my_function(a: u32) {}
                }
            )
            .to_string(),
            quote! {
                fn noitcnuf_ym(a: u32) {}
            }
            .to_string()
        );
    }
}
```

We use [`syn::parse2`](https://docs.rs/syn/latest/syn/fn.parse2.html) to parse
into the type on the left, here an [`ItemFn`](https://docs.rs/syn/latest/syn/struct.ItemFn.html)
which is a function.

As an aside,
because the macro may be applied to something else than a function, eg. in
front on a struct, we might not be able to parse a function. In this case we
bail and return the error.

The naive way to do this would be to panic, but this gives subpar error
messages (`proc macro panicked`). So it would be nice at least to reuse `syn`'s
error.

But we can't return an error in proc macros - only tokens. The
main way around this is to return tokens containing [`core::compile_error`](https://doc.rust-lang.org/core/macro.compile_error.html)
from the Rust standard library.

When that compile error is evaluated, it will return a nice compiler error for
the user. There's a convenient
[`into_compile_error()`](https://docs.rs/syn/latest/syn/struct.Error.html#method.into_compile_error)
in `syn` to do this.

## Testing compiler errors: [`trybuild`](https://github.com/dtolnay/trybuild)

We've talked a bit about errors. It would be nice to solidfy this in our
tests: let's test our errors.

Problem: an error message isn't just a message, so testing macro expansions
like we did before is missing a bit of the picture. Here's what a parse error
looks like for our macro currently, when we try to apply it, for example, to a
struct:

```
error: expected `fn`
 --> src/main.rs:3:1
  |
3 | struct A;
  | ^^^^^^
```

We have the error message, generated by `syn` with `into_compile_error`:
``expected `fn`.`` But that's not all. Under the error is the code it applies
to highlighted.

This is why strings aren't enough for macros. On top of the code, Rust includes
a [`Span`](https://docs.rs/proc-macro2/latest/proc_macro2/struct.Span.html),
that describes where the token was in the original code. (Spans are also used
for [macro hygiene
purposes](https://veykril.github.io/tlborm/proc-macros/hygiene.html), mainly
relevant for identifiers.)

Okay, so how do we test compiler errors? I recommend using
[`trybuild`](https://crates.io/crates/trybuild). Let's try it out with our
error output from above put in `tests/struct.stderr`:

```rust
// tests/struct.rs
use my_crate_proc_macro::my_macro;
#[my_macro]
struct A;
```

```rust
mod tests { // ...
    #[test]
    fn compile_errors() {
        let t = trybuild::TestCases::new();
        t.compile_fail("tests/*.rs");
    }
}
```

For each test that it finds, `trybuild` will compile it, check that it fails
and compare the error output to the corresponding `.stderr` file. `trybuild`
can generate these `.stderr` files for you as well.

# Summary

I hope this article has given you a good overview of procedural macros, as well
as how to get started in a practical manner.

Proc macros are (too?) complicated, and the lack of a fully integrated guide
hurts their usability. The documention is there, just scattered around,
and it's hard to get started.

If you're looking for good
practice, the best resource is likely the [Proc Macro
Workshop](https://github.com/dtolnay/proc-macro-workshop) made by David Tolnay,
the author of many of the libraries listed here.

[rust]: https://www.rust-lang.org/
[macros]: https://doc.rust-lang.org/book/ch19-06-macros.html
[metaprogramming]: https://en.wikipedia.org/wiki/Metaprogramming
[print]: https://doc.rust-lang.org/std/macro.println.html
[dsl]: https://github.com/victorporof/rsx
[macro_rules]: https://doc.rust-lang.org/rust-by-example/macros.html
[rbe]: https://doc.rust-lang.org/rust-by-example/macros.html
[workspace]: https://doc.rust-lang.org/book/ch14-03-cargo-workspaces.html
[macrorulesattr]: https://docs.rs/macro_rules_attribute/latest/macro_rules_attribute/
[macrobook]: https://veykril.github.io/tlborm/introduction.html
[1]: https://blog.rust-lang.org/2018/10/25/Rust-1.30.0.html
[2]: https://docs.rs/syn/latest/syn/
[3]: https://veykril.github.io/tlborm/decl-macros/minutiae.html
[4]: http://bazelbuild.github.io/rules_rust/defs.html#rust_proc_macro
[5]: https://gn.googlesource.com/gn/+/main/docs/reference.md#func_rust_proc_macro
[6]: https://docs.rs/syn/latest/syn/parse/index.html
[7]: https://docs.rs/syn/latest/syn/fn.parse2.html
