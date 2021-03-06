# Deriving De/Serialize for type in a different crate

Rust's [coherence rules] requires that either the trait or the type for which you
are implementing the trait must be defined in the same crate as the impl, so it
is not possible to implement `Serialize` and `Deserialize` for a type in a
different crate directly.

[coherence rules]: https://doc.rust-lang.org/book/traits.html#rules-for-implementing-traits

```diff
- use serde::Serialize;
- use other_crate::Duration;
-
- // Not allowed by coherence rules.
- impl Serialize for Duration {
-     /* ... */
- }
```

To work around this, Serde provides a way of deriving `Serialize` and
`Deserialize` implementations for types in other people's crates. The only catch
is that you have to provide a definition of the type for Serde's derive to
process. At compile time, Serde will check that all the fields in the definition
you provided match the fields in the remote type.

```rust
# #![allow(dead_code)]
#
// Pretend that this is somebody else's crate, not a module.
mod other_crate {
    // Neither Serde nor the other crate provides Serialize and Deserialize
    // impls for this struct.
    pub struct Duration {
        pub secs: i64,
        pub nanos: i32,
    }
}

////////////////////////////////////////////////////////////////////////////////

#[macro_use]
extern crate serde_derive;

extern crate serde;

use other_crate::Duration;

// Serde calls this the definition of the remote type. It is just a copy of the
// remote type. The `remote` attribute gives the path to the actual type.
#[derive(Serialize, Deserialize)]
#[serde(remote = "Duration")]
struct DurationDef {
    secs: i64,
    nanos: i32,
}

// Now the remote type can be used almost like it had its own Serialize and
// Deserialize impls all along. The `with` attribute gives the path to the
// definition for the remote type. Note that the real type of the field is the
// remote type, not the definition type.
#[derive(Serialize, Deserialize)]
struct Process {
    command_line: String,

    #[serde(with = "DurationDef")]
    wall_time: Duration,
}
#
# fn main() {}
```

If the remote type is a struct with all public fields or an enum, that's all
there is to it. If the remote type is a struct with one or more private fields,
getters must be provided for the private fields and a conversion must be
provided to construct the remote type.

```rust
// Pretend that this is somebody else's crate, not a module.
mod other_crate {
    // Neither Serde nor the other crate provides Serialize and Deserialize
    // impls for this struct. Oh, and the fields are private.
    pub struct Duration {
        secs: i64,
        nanos: i32,
    }

    impl Duration {
        pub fn new(secs: i64, nanos: i32) -> Self {
            Duration { secs: secs, nanos: nanos }
        }

        pub fn seconds(&self) -> i64 {
            self.secs
        }

        pub fn subsec_nanos(&self) -> i32 {
            self.nanos
        }
    }
}

////////////////////////////////////////////////////////////////////////////////

#[macro_use]
extern crate serde_derive;

extern crate serde;

use other_crate::Duration;

// Provide getters for every private field of the remote struct. The getter must
// return either `T` or `&T` where `T` is the type of the field.
#[derive(Serialize, Deserialize)]
#[serde(remote = "Duration")]
struct DurationDef {
    #[serde(getter = "Duration::seconds")]
    secs: i64,
    #[serde(getter = "Duration::subsec_nanos")]
    nanos: i32,
}

// Provide a conversion to construct the remote type.
impl From<DurationDef> for Duration {
    fn from(def: DurationDef) -> Duration {
        Duration::new(def.secs, def.nanos)
    }
}

#[derive(Serialize, Deserialize)]
struct Process {
    command_line: String,

    #[serde(with = "DurationDef")]
    wall_time: Duration,
}
#
# fn main() {}
```
