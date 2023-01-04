# `mos-test`

`mos-test` is port of great [`defmt-test`] to [mos] architecture. It is a alternative test harness that lets you write and run unit tests on all llvm-mos platforms as if you were using the built-in `#[test]` attribute.

It is compatible with [rust-analyzer]'s `▶ Run Test` button, which means you can run your tests straight from VS Code

For a full list of mos-test's capabilities, please refer to the documentation below.

[rust-analyzer]: https://rust-analyzer.github.io

## Adding `mos-test` to an existing project

If you want to add `mos-test` to an existing Cargo project / package, for each *crate* that you want to test you need to do these changes in `Cargo.toml`:

- add `mos-test` as a `dev-dependency`
- for each crate that you want to test, set `harness` to `false` to disable the default test harness, the `test` crate which depends on `std`. examples below

``` toml
# Cargo.toml

# for the library crate (src/lib.rs)
[lib]
harness = false

# for the bin crate (src/main.rs)
[[bin]]
name = "binary_name"
harness = false

# for each crate in the `tests` directory
[[test]]
name = "test-name" # tests/test-name.rs
harness = false

[[test]]
name = "second" # tests/second.rs
harness = false
```

The other thing to be aware is that `cargo test` will compile *all* crates in the package, or workspace.
This may include crates that you don't want to test, like `src/main.rs` or each crate in `src/bin` or `examples`.
To identify which crates are being compiled by `cargo test`, run `cargo test -j1 -v` and look for the `--crate-name` flag passed to each `rustc` invocation.

To test only a subset of the crates in the package / workspace you have two options:

- you can specify each crate when you invoke `cargo test`. for example, `cargo test --lib --test integration` tests two crates: the library crate (`src/lib.rs`) and `tests/integration.rs`
- you can disable tests for the crates that you don't want to test -- example below -- and then you can use `cargo test` to test all crates that were not disabled.

if you have this project structure

``` console
$ tree .
.
├── Cargo.toml
├── src
│  ├── lib.rs
│  └── main.rs
└── tests
   └── integration.rs
```

and have `src/lib.rs` set up for tests but don't want to test `src/main.rs` you'll need to disable tests for `src/main.rs`

``` toml
# Cargo.toml
[package]
# ..
name = "app"

[[bin]] # <- add this section
name = "app" # src/main.rs
test = false
```

## Adding state

An `#[init]` function can be written within the `#[tests]` module.
This function will be executed before all unit tests and its return value, the test suite *state*, can be passed to unit tests as an argument.

``` rust
// state shared across unit tests
struct MyState {
    flag: bool,
}

#[defmt_test::tests]
mod tests {
    #[init]
    fn init() -> super::MyState {
        // state initial value
        super::MyState {
            flag: true,
        }
    }

    // This function is called before each test case.
    // It accesses the state created in `init`,
    // though like with `test`, state access is optional.
    #[before_each]
    fn before_each(state: &mut super::MyState) {
        defmt::println!("State flag before is {}", state.flag);
    }

    // This function is called after each test
    #[after_each]
    fn after_each(state: &mut super::MyState) {
        defmt::println!("State flag after is {}", state.flag);
    }

    // this unit test doesn't access the state
    #[test]
    fn assert_true() {
        assert!(true);
    }

    // but this test does
    #[test]
    fn assert_flag(state: &mut super::MyState) {
        assert!(state.flag)
        state.flag = false;
    }
}
```

``` console
$ cargo test -p testsuite
0.000000 (1/2) running `assert_true`...
└─ integration::tests::__defmt_test_entry @ tests/integration.rs:37
0.000001 State flag before is true
└─ integration::tests::before_each @ tests/integration.rs:26
0.000002 State flag after is true
└─ integration::tests::after_each @ tests/integration.rs:32
0.000003 (2/2) running `assert_flag`...
└─ integration::tests::__defmt_test_entry @ tests/integration.rs:43
0.000004 State flag before is true
└─ integration::tests::before_each @ tests/integration.rs:26
0.000005 State flag after is false
└─ integration::tests::after_each @ tests/integration.rs:32
0.000006 all tests passed!
└─ integration::tests::__defmt_test_entry @ tests/integration.rs:11
```

## Test Outcome

Test functions may either return `()` and panic on failure, or return any other type that implements the `TestOutcome` trait, such as `Result`.

This allows tests to indicate failure via `Result`, which allows using the `?` operator to propagate errors.

Similar to Rust's built-in `#[should_panic]` attribute, `mos-test` supports a `#[should_error]` attribute, which inverts the meaning of the returned `TestOutcome`.
`Err` makes the test pass, while `Ok`/`()` make it fail.

## License

Licensed under either of

- Apache License, Version 2.0 ([LICENSE-APACHE](LICENSE-APACHE) or
  http://www.apache.org/licenses/LICENSE-2.0)

- MIT license ([LICENSE-MIT](LICENSE-MIT) or http://opensource.org/licenses/MIT)

at your option.

### Contribution

Unless you explicitly state otherwise, any contribution intentionally submitted
for inclusion in the work by you, as defined in the Apache-2.0 license, shall be
licensed as above, without any additional terms or conditions.

[Knurling]: https://knurling.ferrous-systems.com
[Ferrous Systems]: https://ferrous-systems.com/
[GitHub Sponsors]: https://github.com/sponsors/knurling-rs
[`defmt-test`]: https://github.com/knurling-rs/defmt/tree/main/firmware/defmt-test
[mos]: https://github.com/mrk-its/rust-mos