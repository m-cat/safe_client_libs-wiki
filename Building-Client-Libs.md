## Installing Rust

The Rust compiler is required in order to build Client Libs. Please follow the official [Rust installation instructions](https://www.rust-lang.org/en-US/install.html).

The latest **stable** version of Rust is required.

If you already have Rust installed, you may need to upgrade to the latest stable:

```
rustup update stable
```

## Downloading Client Libs

The Client Libs repository can be downloaded either as a zip archive from the [https://github.com/maidsafe/safe_client_libs](official repository) or by using Git:

```
git clone https://github.com/maidsafe/safe_client_libs.git
```

## Building The Libraries

To build one of the libraries, first navigate to its directory: this will be either `safe_core`, `safe_authenticator`, or `safe_app`.

To build the library in debug mode, simply use the command:

```
cargo build --release
```

This builds the library in release mode, which is how we build our official binaries.

To run tests:

```
cargo test --release
```

**Note:** Make sure to always build and test in release mode (indicated by the `--release` flag). When testing, this will catch rare FFI bugs that may not manifest themselves in debug mode. Debug mode is also unoptimized and can take an inordinate amount of time to run tests or examples.

## Building for Mobile Platforms

See [Manually building SCL for mobile targets](.)

## Features

Rust supports conditional compilation through a mechanism called features. Features allow us to provide builds with different capabilities. The Client Libs features are:

- `use-mock-routing`: This feature enables Mock Routing, a fake routing network that does not make a connection to the live network. This is what we use when running most of our tests. The entire mock network is stored locally in a file named `MockVault`, which gives us the ability to reproduce conditions of any local mock network. You can find more details on the [Mock vs. non-mock](./Mock-vs.-non-mock) page.
- `testing`: This feature enables building with test utilities, which include functions for setting up random test clients. Some test utilities, such as the functions that set up routing hooks, also require the `use-mock-routing` flag to be built. More details about the test utilities can be found on the [Testing framework](.) page.
- `bindings`: This feature enables the generation of bindings for C, C\# and Java so that the Safe Authenticator and Safe App libraries can be natively called from those languages. This feature is not available for Safe Core. More details about this feature can be found on the [Interfacing with other languages & Safe-Bindgen](.) page.

To build a library with a feature, pass the `--features` flag:

```
cargo test --release --features "use-mock-routing"
```

You can pass multiple features:

```
cargo build --release --features "use-mock-routing testing"
```

## Configuring SAFE Client Libs

There are multiple configuration options available and some might affect the working of Client Libs, especially if you're running your own local vaults.

Please refer to the [Configuring Client Libs](./Configuring-Client-Libs) page for details.