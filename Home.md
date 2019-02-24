Welcome to the SAFE Client Libs wiki!

This is the official public knowledge base for Client Libs. Please use the sidebar to navigate around.

## Crate graph

This is how all the pieces of the MaidSafe backend stack fit together:

![safe_app dependencies](safe-client-libs.png)

<details>
<summary>More info</summary>

This was generated using [cargo-deps](https://github.com/m-cat/cargo-deps) and the following command:

```
cargo deps --all-deps --include-orphans --subgraph safe_app safe_app_jni safe_authenticator safe_authenticator_jni safe_core --subgraph-name "SAFE Client Libs" --filter accumulator config_file_handler crust ffi_utils fake_clock lru_time_cache maidsafe_utilities parsec resource_proof routing rust_sodium safe_app safe_app_jni safe_authenticator safe_authenticator_jni safe_bindgen safe_core safe_crypto safe_vault secure_serialisation self_encryption system_uri tokio_utp --manifest-path safe_app/Cargo.toml | dot -Tpng -Nfontname=Iosevka -Gfontname=Iosevka >| safe-client-libs.png
```

</details>

## Contributing

Note that this this wiki is only directly editable by contributors, as all pages are expected to undergo a review process. If you wish to submit a page or make a change to the wiki, please raise an [Issue](https://github.com/maidsafe/safe_client_libs/issues/new) describing the change and we'll get on it!
