## Non-Mock (Real) Libraries

### What is it

The "real" version of the libraries is used in the production SAFE Browser. The real Client Libraries interface with the Routing library, the backbone of the SAFE Network which provides all of its security and privacy guarantees, to connect to the public network.

### How to build

Follow the instructions at [Building Client Libs](.).

### Advantages

- As the real libraries interface with Routing and use a real account, they can test many things that the mock libraries are unable to. See the integration tests in the `tests` subdirectory.

## Mock Libraries

### What is it

As opposed to the real libraries, which connect to the public network, the mock libraries merely simulate all network operations locally using a database created on your system.

### How to build

Follow the instructions at [Building Client Libs](.) and use the `–-features "mock-network"` flag.

### Advantages

- The real libraries require an actual account to use the network, which has a PUT limit. The mock libraries are therefore more useful for testing, as new accounts can be created on the fly or the PUT limit removed (see "Configuration Parameters").

### Configuration Parameters

For more information on configuring the behavior of the mock libraries (e.g. lifting the PUT limit), please see [Environment Variables](https://docs.rs/safe_core/0.32.0/safe_core/#environment-variables) and [Configuring Client Libs](./Configuring-Client-Libs).

## Mock Development

You can use the `mock-routing-feature` to conditionally compile code only for the mock libraries. For example, you can configure code to either use the real `Routing` struct when compiled without `mock-network`, or the the `MockRouting` struct, renamed as `Routing`, when `mock-network` is enabled:

```rust
#[cfg(feature = "mock-network")]
use self::mock::Routing;
#[cfg(not(feature = "mock-network"))]
use routing::Client as Routing;
```

The compiler will generally help you along in knowing when and where to use feature flags. If you write a new function that calls `mock-network`-only functions, the compiler will complain that those functions do not exist when you try to compile without `mock-network`. You'll therefore know to either gate the new function with `mock-network`, or to avoid using those functions if your new function needs to be in production.

### Dependencies

We are more lax about dependencies in the mock libraries because they are not used in production and do not require the same level of scrutiny for inclusion in our code.

A mock-only dependency can be included as per usual in the `Cargo.toml` file:

`lazy_static = "~1.0.0"`

But needs to be feature-gated in `lib.rs`:

```rust
#[cfg(feature = "mock-network")]
#[macro_use]
extern crate lazy_static;
```

### Modules

You can configure certain modules to only be included for the mock libraries. The mock routing implementation itself is configured this way:

```rust
#[cfg(feature = "mock-network")]
mod mock;
```

### Functions

You may want to define certain functions only for the mock libraries. For example, the following function, which adds custom hooks to Mock Routing to alter its behavior, will only be compiled if `mock-network` is enabled AND tests are being run:

```rust
#[cfg(all(feature = "mock-network", any(test, feature = "testing")))]
/// Allows customising the mock Routing client before registering a new account
pub fn registered_with_hook<F>(
```

### Tests

Some tests should only be run on the mock libraries. Generally this applies to any test that simulates network operations, or which adds custom routing hooks.

```rust
// Test simulating network disconnects.
#[cfg(feature = "mock-network")]
#[test]
fn simulate_network_disconnect() {
```

### Exports

You may wish to publicly export an API only for the mock libraries, e.g. for testing purposes:

```rust
#[cfg(feature = "mock-network")]
pub use self::client::{mock_vault_path, MockRouting};
```
