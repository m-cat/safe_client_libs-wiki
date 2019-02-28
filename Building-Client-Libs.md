## Installing Rust

The Rust compiler is required in order to build Client Libs. Please follow the official [Rust installation instructions](https://www.rust-lang.org/en-US/install.html).

The latest **stable** version of Rust is required.

If you already have Rust installed, you may need to upgrade to the latest stable:

```
rustup update stable
```

## Downloading Client Libs

The Client Libs repository can be downloaded either as a zip archive from the [official repository](https://github.com/maidsafe/safe_client_libs) or by using Git:

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

See [Building for mobile](./Building-for-mobile).

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

## Docker

One option for building the libraries is to use Docker. If you're not familiar with it, the official [getting started guide](https://docs.docker.com/get-started/) is a good general introduction to the concepts and terminology. Docker provides a mechanism to build Client Libs without having to install anything else in the build environment.

Though it would be possible to use Docker in a day-to-day development workflow, it has certain practicalities that probably don't make it as suitable for that. It's more useful if you want to reproduce our remote build environment locally, usually for debugging purposes. If you want to get a shell inside the container, run the `make debug` target.

This repository provides a Dockerfile that defines a container with the prerequisites necessary for building Client Libs. You can build the container by running `make build-container`. The container build process runs a build of the safe_authenticator and safe_app libraries. This is so it can resolve and compile all the dependencies; subsequent runs of the container will then have these dependencies cached. Since the dependencies for the non-mocked versions of safe_authenticator and safe_app are a superset of the mocked versions, the container also has all the dependencies necessary for performing a mocked build.

After building the container, you can build the Client Libs libraries just by running `make` at the root of this repository, or you can also run a build with tests by running `make tests`. If you want to run a build with mocked routing, run `make build-mock`. These targets wrap the details of using the container, making it much easier to work with. The current source directory is mounted into the container with the correct permissions, and the artifacts produced by the build are copied out of the container and placed at the root of this repository. See the Makefile for more details.

## Configuring SAFE Client Libs

There are multiple configuration options available and some might affect the working of Client Libs, especially if you're running your own local vaults.

Please refer to the [Configuring Client Libs](./Configuring-Client-Libs) page for details.