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

Follow the instructions at [Building Client Libs](.) and use the `–-features "use-mock-routing"` flag.

### Advantages

- The real libraries require an actual account to use the network, which has a PUT limit. The mock libraries are therefore more useful for testing, as new accounts can be created on the fly or the PUT limit removed (see "Configuration Parameters").

### Configuration Parameters

For more information on configuring the behavior of the mock libraries (e.g. lifting the PUT limit), please see [Environment Variables](https://docs.rs/safe_core/0.32.0/safe_core/#environment-variables) and [Configuring Client Libs](./Configuring-Client-Libs).

## Mock Development

You can use the `mock-routing-feature` to conditionally compile code only for the mock libraries. For example, you can configure code to either use the real `Routing` struct when compiled without `use-mock-routing`, or the the `MockRouting` struct, renamed as `Routing`, when `use-mock-routing` is enabled:

```rs
#[cfg(feature = "use-mock-routing")]
use self::mock::Routing;
#[cfg(not(feature = "use-mock-routing"))]
use routing::Client as Routing;
```

The compiler will generally help you along in knowing when and where to use feature flags. If you write a new function that calls `use-mock-routing`-only functions, the compiler will complain that those functions do not exist when you try to compile without `use-mock-routing`. You'll therefore know to either gate the new function with `use-mock-routing`, or to avoid using those functions if your new function needs to be in production.

### Dependencies

We are more lax about dependencies in the mock libraries because they are not used in production and do not require the same level of scrutiny for inclusion in our code.

A mock-only dependency can be included as per usual in the `Cargo.toml` file:

`lazy_static = "~1.0.0"`

But needs to be feature-gated in `lib.rs`:

```rs
#[cfg(feature = "use-mock-routing")]
#[macro_use]
extern crate lazy_static;
```

### Modules

You can configure certain modules to only be included for the mock libraries. The mock routing implementation itself is configured this way:

```rs
#[cfg(feature = "use-mock-routing")]
mod mock;
```

### Functions

You may want to define certain functions only for the mock libraries. For example, the following function, which adds custom hooks to Mock Routing to alter its behavior, will only be compiled if `use-mock-routing` is enabled AND tests are being run:

```rs
#[cfg(all(feature = "use-mock-routing", any(test, feature = "testing")))]
/// Allows customising the mock Routing client before registering a new account
pub fn registered_with_hook<F>(
```

### Tests

Some tests should only be run on the mock libraries. Generally this applies to any test that simulates network operations, or which adds custom routing hooks.

```rs
// Test simulating network disconnects.
#[cfg(feature = "use-mock-routing")]
#[test]
fn simulate_network_disconnect() {
```

### Exports

You may wish to publicly export an API only for the mock libraries, e.g. for testing purposes:

```rs
#[cfg(feature = "use-mock-routing")]
pub use self::client::{mock_vault_path, MockRouting};
```

## Mock Implementation

### Mock Vault

This is a mock implementation of the [vault](https://github.com/maidsafe/safe_vault) and its various operations:

- Storing data (in a local `MockVault` file by default, though this can be configured)
- Authorising operations like reads and mutations
- Committing mutations and counting them (limit of 1000 by default)

It can be configured, see "Configuration Parameters" above.

The data stored in the MockVault file can be moved between computers, so an error on an end user's computer can be reproduced locally and investigated by a developer.

### Mock Routing

Has the same interface as real routing but sits on top of the mock vault. All of the operations are mocked, but made to be as realistic as possible, even incurring some delays via `sleep` so that they are not nearly-instantaneous.

Also allows inserting hooks (e.g. overriding a request or response) for testing purposes.

Can simulate disconnects and timeouts for testing of certain things (such as operations recovery, i.e. how the libraries are able to robustly handle network interruptions).

### Mock Account

The mock accounts are handled by the mock vault and are what actually count the mutations performed and mutations available. Each mock account has a different counter for these. The mock account also mocks other account properties such as storing the authenticator key and bumping the version when the key is updated.

### Testing

When testing, instances of `MockRouting` can be created with custom configuration parameters, i.e. to turn off the mutation limit for one test only. See `setup_with_config`. For an example of its usage see the test `low_balance_check`.

Note that by default, a single shared instance of `Vault` is created for all tests, and new instances of `MockRouting` will interface with this shared instance. The `setup_with_config` option will create a separate, custom vault, with e.g. the mutation limit turned off or in-memory storage turned on, so that other tests are not affected.
