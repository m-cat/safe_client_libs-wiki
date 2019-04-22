# Building for mobile

## iOS (Building on OSX)

### Prerequisites/Assumptions

- Working on an OSX machine.
- Have xcode and xcode command line tools installed.

### Step-by-step guide

Steps:

1. Checkout the latest Client Libs master branch and `cd` into the Client Libs directory.
1. Run the commands below to invoke cargo script on package.rs in the scripts directory:
    1. **SAFE App mock:** `./scripts/package.rs --lib --name safe_app -d <OUTPUT_DIRECTORY> --mock -a ios`
    1. **SAFE App non-mock:** `./scripts/package.rs --lib --name safe_app -d <OUTPUT_DIRECTORY> -a ios`
    1. **SAFE Authenticator mock:** `./scripts/package.rs --lib --name safe_authenticator -d <OUTPUT_DIRECTORY> --mock -a ios`
    1. **SAFE Authenticator non-mock:** `./scripts/package.rs --lib --name safe_authenticator -d <OUTPUT_DIRECTORY> -a ios`
1. Confirm that the four zip files have been created in your chosen `<OUTPUT_DIRECTORY>`.
1. *MaidSafe only:* Upload final zips to `safe-client-libs` folder in s3 to be consumed by app. Set them to be read-only, to avoid any overwriting.

This generates a universal .a file in the zips (universal as in it works for both `aarch64-apple-ios` and `x86_64-apple-ios` targets).

## Cross compilation of native libraries for mobile

**Definition:** Cross compilation is the process of building for a target architecture other than the one on which the compiler is running.

### Prerequisites

- Android NDK (download from the [official site](https://developer.android.com/ndk/downloads/))
- NDK standalone toolchain (for each architecture)
- Rustup targets (for each architecture)

### Building the toolchains

To build the toolchain for a particular architecture and android API version:

```shell
<ndk-folder>/build/tools/make_standalone_toolchain.py --arch <architecture> --api <api version> --install-dir <target directory>
```

Example:

```shell
~/code/android-ndk/build/tools/make_standalone_toolchain.py --arch arm --api 21 --install-dir ~/toolchains/android-21-arm-toolchain
```

More info: [Standalone toolchains](https://developer.android.com/ndk/guides/standalone_toolchain)

### Adding rustup targets

To compile Client Libs for different targets, you will need those targets to be added. This can be done using [rustup](https://github.com/rust-lang-nursery/rustup.rs).

For example, to add the Android arm target, run:

```shell
rustup target add armv7-linux-androideabi
```

More info: [Cross compilation](https://github.com/rust-lang-nursery/rustup.rs#cross-compilation)

### Required environment variables

Before cross compilation we must ensure that the required environment variables are set:

- The `bin` folder of the NDK standalone toolchain must be added to the `PATH` variable:

```shell
export PATH=$PATH:~/toolchains/android-21-arm-toolchain/bin
```
    
- The `CARGO_TARGET_<TARGET>_LINKER` variable must be set to the the gcc executable of the toolchain:
    
    For example, for the `armv7-linux-androideabi` target the variable to be set will be:

```shell
export CARGO_TARGET_ARMV7_LINUX_ANDROIDEABI_LINKER=~/toolchains/android-21-arm-toolchain/bin/arm-linux-androideabi-gcc
```

### Cross compilation

Once you've completed the above steps we can now compile Client Libs for Android targets:

```shell
cargo build --release --features="mock-network" --target="<required target>"
```

For example, to compile SAFE App for `armv7-linux-androideabi`, go inside the `safe_app` folder and run:

```shell
cargo build --release --features="mock-network" --target="armv7-linux-androideabi"
```

### Generating universal native library for iOS

- Add the targets for iOS ARM64 & iOS x64:

```shell
rustup target add aarch64-apple-ios
rustup target add x86_64-apple-ios
```

- To manually generate a universal SAFE App mock binary for iOS:
    - Run the following inside the `safe_app` directory:
`cargo build --release --features="mock-network" --target="aarch64-apple-ios"` and `cargo build --release --features="mock-network" --target="x86_64-apple-ios"`
    - Copy `libsafe_app.a` from `../target/aarch64-apple-ios/release` to a separate directory folder and rename this file to `libsafe_app-arm64.a`
    - Copy `libsafe_app.a` from `../target/x86_64-apple-ios/release` to the same directory as above and rename to `libsafe_app-x64.a`
    - Run the following command in the directory to generate a universal binary: `lipo -create -output libsafe_app.a libsafe_app-arm64.a libsafe_app-x64.a`

    - Now you have a new file called `libsafe_app.a` which includes both native binaries.

- Repeat the above without the `mock-network` feature to generate a non-mock SAFE App binary.
- The steps for SAFE Authenticator will be the same, except that the library name will be `libsafe_authenticator.a`.

### Generating universal libraries using cargo-script

You can generate SAFE App/Authenticator mock/non-mock binaries for iOS using cargo-script. Remove `--mock` to generate non-mock libs.

```shell
./scripts/package.rs --lib --name safe_app/safe_authenticator -d OUTPUT_DIRECTORY --mock -a ios
```
