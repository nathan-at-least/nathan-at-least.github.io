# Removing Rust Boilerplate

Remove all the boilerplate!

**Warning:** I haven't carefully vetted these dependencies for security /
safety (beyond using `cargo-audit`):

<!-- toc -->

# Why?

Here are recipes I use to remove boilerplate from rust code. I'm a big fan of doing so for a few reasons:

- This style makes it apparent to a reader at a glance what behavior is provided.
- More concise code is often better for readability/comprehensability.
- Using "boilerplate removal" tools also helps ensure consistency via the tool.
- Boilerplate-removal crates and tools can be developed and improved
  which can potentially feed improvements into consuming code, whereas
  hand-written boilerplate must be updated on a case-by-case basis.
- If boilerplate removal is consistently used, then any boilerplate-like
  code is more obviously unique/custom for a purpose.

  For example, if a codebase always uses `#[derive(PartialEq)]` when
  possible, then any custom `derive PartialEq for MyType …` can
  immediately cue the reader that there is some unique behavior of this
  impl. By contrast, if boilerplate is always written out, then readers
  have to examine each case carefully to determine if it's "standard"
  or unique behavior.

There are potential downsides to boiler-plate removal:

- Sometimes what you need has an impedence mismatch with some
  boilerplate-removal tool, so you may try to work around that limitation,
  and at some point it becomes more complex or cumbersome than hand-written
  code. (Maybe submit a patch to the tool! 😉)
- Often boiler-plate removal relies on procedural macros, which can slow
  down build times.
- Changes to boiler-plate removal code may change your code unexpectedly.
- Compiler errors stemming either from the boiler-plate removal _or_ from
  your code's interaction with it may have more confusing error messages.

Do you have favorite boiler-plate removal not mentioned here?
Suggestions welcome.

# Use `std` derives

Many `std` traits can be derived, see [this
reference](https://doc.rust-lang.org/book/appendix-03-derivable-traits.html)
for more detail:

- `Clone`
- `Debug`
- `Default`
- `PartialEq`, `Eq`, `PartialOrd`, `Ord`
- `Hash`

# [`derive_more`](https://docs.rs/derive_more)

https://docs.rs/derive_more

This crate is full of goodies to remove common trait derivations.

## `derive_more::From` for enums, especially error enums

Especially useful making enum construction concise. When used with error
types, this allows concisely propagating inner error types with the
`?` operator:

```rust
#[derive(derive_more::From)]
pub enum Error {
    Stdio(std::io::Error),
    MyApp(crate::AppError),
}

fn do_stuff() -> Result<(), Error> {
    // Propagate a `std::io::Error`:
    let data = std::fs::read("/input/file")?;

    // Propagate a `crate::AppError`:
    process_data(data)?;
    Ok(())
}

fn process_data(data: Vec<u8>) -> Result<(), crate::AppError> {
    todo!();
}
```

# [`derive_new`](https://docs.rs/derive_new) for struct `new` functions

Create a `new` method automatically.

```rust
#[derive(derive_new::new)]
pub struct MyThing {
    foo: String,
    bar: bool,
}

fn do_stuff() {
    let thing = MyThing::new("alice".to_string(), true);
    todo!("do stuff with thing");
}
```

# [`derive_builder`](https://docs.rs/derive_builder) for builder patterns

If you need a builder pattern interface, this crate can save a lot of
tedious boilerplate.

# [`anyhow`](https://docs.rs/anyhow) for concise contextual errors

I've become a big fan of `anyhow` because it removes the need for
spelling out custom error types. This can be a trade-off, though, since
"transparent" error types are more ergonomic if the consuming code needs
to frequently behave differently based on different error types.

Keep in mind consumers can _still_ distinguish specific errors and
behave accordingly with slightly more verbose code, so assuming that's
relatively uncommon, I believe `anyhow` is a good option.

Typically I try to avoid introducing `anyhow` into reusable library-style
crates, and only use it in application-level code.

# [`anyhow-std`](https://docs.rs/anyhow-std) provides error context for many `std` APIs

This is a crate I created after a few too many error messages about file
IO without understanding which path was the culprit. See above about
whether or not to introduce `anyhow` in your API, but I definitely reach
for this crate any time I'm writing application-level code.

# [`testcase`](https://docs.rs/test-case) for testing many vectors as independent tests

Sometimes you want to verify a function against a bunch of test vector
inputs/outputs. If you do this in a loop in a single unit test, you
cannot quickly determine which case failed, slowing down debugging.

Use test case to generate separate tests for each input/output case so
that failures help you more quickly find the source of the bug.

```rust
#[test_case("3" => 3)]
#[test_case("7" => 7)]
#[test_case("+42" => 42)]
#[test_case("-5" => -5)]
#[test_case("-0xa0" => -160)]
fn parse_int(s: &str) -> i32 {
  s.parse().unwrap()
}
```

Running `cargo test` quickly shows which cases pass/fail:

```
running 5 tests
test parse_int::_0xa0_expects_160 ... FAILED
test parse_int::_42_expects_42 ... ok
test parse_int::_7_expects_7 ... ok
test parse_int::_5_expects_5 ... ok
test parse_int::_3_expects_3 ... ok
```

-so now we immediately see that our `.parse()` call doesn't work for
`-0xa0` as an input, whereas the other cases are fine.

# [`indoc`](https://docs.rs/indoc) for indenting string literals

If you find yourself littering `\n` in string literals a lot, you probably
would benefit in code readability with `indoc`. I find it especially
useful to use in tests with `test_case` for parsers or formatters.

# [`clap`](https://docs.rs/clap) derive API to remove cli option parsing boilerplate

Use `clap` with the [derive
API](https://github.com/clap-rs/clap/blob/v3.2.8/examples/derive_ref/README.md)
to cut down on cli parsing boilerplate.

# [`cargo-checkmate`](https://docs.rs/cargo-checkmate) to simplify running many checks and producing basic Github CI

This is a crate I created specifically because I got tired of repeatedly
running different checks on each commit. By using the git hook, this
helps me ensure a crate or workspace builds, passes clippy tests,
generates docs without warnings, etc…
