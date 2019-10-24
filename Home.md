Welcome to the **SAFE Client Libs** wiki!

This is the official public knowledge base for Client Libs. Please use the sidebar to navigate around.

## Introduction

Please see [Introduction to Client Libs](./Introduction-to-Client-Libs) for a high-level description of the project.

## Crate graph

This is how all the pieces of the MaidSafe backend stack fit together. Each arrow points from a crate to one of its dependencies.

![safe_app dependencies](safe-client-libs.png)

_Last updated Feb 24_

**Legend:**
* Box node: top-level crate (`safe_app`)
* Black: normal dependencies
* Blue: dev dependencies (`safe_app` -> `safe_authenticator`)
* Purple: build dependencies (`safe_app` -> `safe_bindgen`)
* Yellow: crates in Client Libs that `safe_app` does not depend on

<details>
<summary>More info</summary>

This was generated using [cargo-deps](https://github.com/m-cat/cargo-deps) and the following command:

```shell
cargo deps --all-deps --include-orphans --subgraph safe_app safe_app_jni safe_authenticator safe_authenticator_jni safe_core --subgraph-name "SAFE Client Libs" --filter accumulator config_file_handler crust ffi_utils fake_clock lru_time_cache maidsafe_utilities parsec resource_proof routing rust_sodium safe_app safe_app_jni safe_authenticator safe_authenticator_jni safe_bindgen safe_core safe_crypto safe_vault secure_serialisation self_encryption system_uri tokio_utp --manifest-path safe_app/Cargo.toml | dot -Tpng -Nfontname=Iosevka -Gfontname=Iosevka >| safe-client-libs.png
```

</details>

## Contributing to this wiki

Note that this this wiki is only directly editable by contributors, as all pages are expected to undergo a review process. If you wish to submit a page or make a change to the wiki, please raise an [Issue](https://github.com/maidsafe/safe_client_libs/issues/new) describing the change and we'll get on it!
