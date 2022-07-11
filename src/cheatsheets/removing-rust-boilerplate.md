# Removing Rust Boilerplate

**Warning:** I haven't carefully vetted these dependencies for security / safety (beyond using `cargo-audit`):

Suggestions welcome.

## `derive_more::From` for enums, especially error enums

Especially useful making enum construction concise. When used with error types, this allows concisely propagating inner error types with the `?` operator:

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

## `derive_new::new` for struct `new` functions

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

## `testcase` for testing many vectors as independent tests

Sometimes you want to verify a function against a bunch of test vector inputs/outputs. If you do this in a loop in a single unit test, you cannot quickly determine which case failed, slowing down debugging.

Use test case to generate separate tests for each input/output case so that failures help you more quickly find the source of the bug.

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

-so now we immediately see that our `.parse()` call doesn't work for `-0xa0` as an input, whereas the other cases are fine.

## `indoc` for indenting string literals

If you find yourself littering `\n` in string literals a lot, you probably would benefit in code readability with `indoc`. I find it especially useful to use in tests with `test_case` for parsers or formatters.

## `clap` derive API to remove cli option parsing boilerplate

Use `clap` with the [derive API](https://github.com/clap-rs/clap/blob/v3.2.8/examples/derive_ref/README.md) to cut down on cli parsing boilerplate.
