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