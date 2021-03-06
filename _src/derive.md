# Using derive

Serde provides a derive macro to generate implementations of the `Serialize` and
`Deserialize` traits for data structures defined in your crate, allowing them to
be represented conveniently in all of Serde's data formats.

**You only need to set this up if your code is using `#[derive(Serialize,
Deserialize)]`.**

This functionality is based on Rust's `#[derive]` mechanism, just like what you
would use to automatically derive implementations of the built-in `Clone`,
`Copy`, `Debug`, or other traits. It is able to generate implementations for
most structs and enums including ones with elaborate generic types or trait
bounds. On rare occasions, for an especially convoluted type you may need to
[implement the traits manually](custom-serialization.md).

These derives require a Rust compiler version 1.15 or newer.

!CHECKLIST
- Add `serde = "1.0"` as a dependency in Cargo.toml.
- Add `serde_derive = "1.0"` as a dependency in Cargo.toml.
- Ensure that all other Serde-based dependencies (for example serde_json) are on
  a version that is compatible with serde 1.0.
- If you have a main.rs, add `#[macro_use] extern crate serde_derive` there.
- If you have a lib.rs, add `#[macro_use] extern crate serde_derive` there.
- Use `#[derive(Serialize)]` on structs and enums that you want to serialize.
- Use `#[derive(Deserialize)]` on structs and enums that you want to
  deserialize.

Here is the `Cargo.toml`:

!FILENAME Cargo.toml
```toml
[package]
name = "my-crate"
version = "0.1.0"
authors = ["Me <user@rust-lang.org>"]

[dependencies]
serde = "1.0"
serde_derive = "1.0"

# serde_json is just for the example, not required in general
serde_json = "1.0"
```

Now the `src/main.rs` which uses Serde's custom derives:

!FILENAME src/main.rs
!PLAYGROUND 454091616f81d99a48e72d1c5a430f2a
```rust
#[macro_use]
extern crate serde_derive;

extern crate serde;
extern crate serde_json;

#[derive(Serialize, Deserialize, Debug)]
struct Point {
    x: i32,
    y: i32,
}

fn main() {
    let point = Point { x: 1, y: 2 };

    let serialized = serde_json::to_string(&point).unwrap();
    println!("serialized = {}", serialized);

    let deserialized: Point = serde_json::from_str(&serialized).unwrap();
    println!("deserialized = {:?}", deserialized);
}
```

Here is the output:

```
$ cargo run
serialized = {"x":1,"y":2}
deserialized = Point { x: 1, y: 2 }
```

### Troubleshooting

Sometimes you may see compile-time errors that tell you:

```
the trait `serde::de::Deserialize` is not implemented for `...`
```

even though the struct or enum clearly has `#[derive(Deserialize)]` on it.

This almost always means that you are using libraries that depend on
incompatible versions of Serde. You may be depending on serde 1.0 and
serde_derive 1.0 in your Cargo.toml but using some other library that depends on
serde 0.9. So the `Deserialize` trait from serde 1.0 may be implemented, but the
library expects an implementation of the `Deserialize` trait from serde 0.9.
From the Rust compiler's perspective these are totally different traits.

The fix is to upgrade or downgrade libraries as appropriate until the Serde
versions match. The [`cargo tree -d`] command is helpful for finding all the
places that duplicate dependencies are being pulled in.

[`cargo tree -d`]: https://github.com/sfackler/cargo-tree
