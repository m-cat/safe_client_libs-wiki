Here we document the public export strategy we use in SAFE Client Libs. We developed it for two purposes:

- Exporting objects and modules through our public API.
- Making FFI objects visible to [SAFE Bindgen](https://github.com/maidsafe/safe_bindgen) for the purpose of generating bindings.

At first glance these seem like simple concepts, but there are some subtle points to be aware of.

We assume some prior knowledge, including:

- The usage of [`pub use`](https://doc.rust-lang.org/1.40.0/reference/items/use-declarations.html#use-visibility) to re-export items.

## Note about `pub use` and `pub mod`

In many of our Rust crates we use `pub use` to make items outwardly-visible in the public API of a library or module.

It is our convention in our Rust crates to separate `pub use` blocks from `use` statements (which have the important difference that they do not affect the public API). We also try to group all `pub use` statements in a single block. This makes it easier to visually scan a file and determine the public API at a glance.

For example, here is a `pub use` block separated from a `use` block by an empty line:

```rust
pub use self::errors::AuthError;
pub use client::AuthClient;

use crate::ffi::errors::Error;
use futures::stream::Stream;
use futures::sync::mpsc;
```

Again, one block helps define the public API while the other has nothing to do with it.

If we were to remove the blank line and run `rustfmt`, we would end up with this:

```rust
pub use self::errors::AuthError;
use crate::ffi::errors::Error;
pub use client::AuthClient;
use futures::stream::Stream;
use futures::sync::mpsc;
```

The same point about blank lines applies to `pub mod`. Compare

```rust
pub mod entries;
pub mod entry_actions;
pub mod metadata;
pub mod permissions;

mod helper;
#[cfg(test)]
mod tests;
```

to:

```rust
pub mod entries;
pub mod entry_actions;
mod helper;
pub mod metadata;
pub mod permissions;
#[cfg(test)]
mod tests;
```

## Exporting FFI objects

There is a very specific export format we require for our FFI modules so that all FFI objects can be parsed by SAFE Bindgen. (Ideally, we would modify SAFE Bindgen to be able to detect these modules automatically, but we haven't done this yet.)

The format is simple: for every FFI module, we must `pub use` it in `lib.rs` and specify `*` to include all items from the module. For example, here is our FFI export block for `safe_authenticator`:

```rust
// Export FFI interface

pub use crate::ffi::apps::*;
pub use crate::ffi::errors::codes::*;
pub use crate::ffi::ipc::*;
pub use crate::ffi::logging::*;
pub use crate::ffi::*;
```

**Note:** to double-check which modules SAFE Bindgen will parse, you can run:

```shell
cargo build --features bindings -vv
```

## Re-exports in SAFE App

One final note. In SAFE App, we have a section in `lib.rs` that re-exports some functionality from other crates, mostly SAFE Core. By re-exporting certain things, we try to make SAFE App feature-complete with respect to the Rust-only API.

We do this because while the FFI API is complete, someone using the Rust API in SAFE App would not be able to access all of the same functionality of the FFI API without also specifying SAFE Core as a dependency.

**Note:** we intend to one day make each function in the FFI API a wrapper around the corresponding Rust functions, and provide equivalent Native and FFI APIs. This is another thing we haven't had time to plan and implement yet.