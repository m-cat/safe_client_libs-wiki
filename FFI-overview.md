## Background

Client Libs, like any other library, provides a set of functions, constants and types which constitute the Client Libs API, which allows other programs to access its functionality. This API can be split into two distinct groups. One is the set of native Rust functions and structs provided by Client Libs which can be accessed by other Rust programs. The other group is the Client Libs FFI, or Foreign Function Interface.

As the native Rust API can only be used by other Rust programs, it would be of limited usefulness by itself as it would require consuming programs to be written in Rust. So while a native set of functions and structs is the most natural, and easiest, way to use Client Libs, we needed to also provide a way to access its functionality from any other language. This is what we mean by FFI: it is an interface which uses C function and struct conventions, the so-called *lingua franca* of the programming world. Since most popular languages are interoperable with C, many languages apart from Rust will be able to call into the FFI provided by Client Libs.

## FFI Utils

Client Libs makes use of a wide number of utilities provided by our crate [FFI Utils](https://github.com/maidsafe/ffi_utils). This crate used to be part of Client Libs itself, but as it is a useful standalone crate and was being relied upon by an increasing number of our other projects, we split it out into its own repository.

Please see the [FFI Utils wiki](https://github.com/maidsafe/ffi_utils/wiki) for an overview of this crate and a list of the FFI goodies it provides.

## FFI in Client Libs

### SAFE Core

The SAFE Core [FFI module](https://docs.rs/safe_core/*/safe_core/ffi/index.html) provides FFI versions of [Core data types](./Core-data-types), such as an [`MDataInfo`](https://docs.rs/safe_core/*/safe_core/ffi/struct.MDataInfo.html) and [`File`](https://docs.rs/safe_core/*/safe_core/ffi/nfs/struct.File.html) that C can understand.

It also provides definitions for [array types](https://docs.rs/safe_core/*/safe_core/ffi/arrays/index.html) used in our FFI and [IPC data types](https://docs.rs/safe_core/*/safe_core/ffi/ipc/index.html).

These FFI definitions are used extensively in the SAFE Authenticator and SAFE App FFIs.

### SAFE Authenticator

The SAFE Authenticator [FFI module](https://docs.rs/safe_authenticator/*/safe_authenticator/ffi/index.html) provides the FFI versions of the SAFE Authenticator API. 

### SAFE App

The SAFE App [FFI module](https://docs.rs/safe_app/*/safe_app/ffi/index.html) provides the FFI versions of the SAFE App API.

#### The object cache

SAFE App makes use of an [object cache](https://docs.rs/safe_app/*/safe_app/ffi/object_cache/index.html) to store objects. This means that when you request, for example, an `MDataEntries` using the SAFE App FFI, you get an `MDataEntriesHandle` instead of the entire object. This handle is what you pass to further FFI calls. 

For example, having obtained an `MDataEntriesHandle` using [`mdata_entries_new`](https://docs.rs/safe_app/*/safe_app/ffi/mutable_data/entries/fn.mdata_entries_new.html), you must call [`mdata_entries_insert`](https://docs.rs/safe_app/*/safe_app/ffi/mutable_data/entries/fn.mdata_entries_insert.html) by passing in this handle. At this point `mdata_entries_insert` will internally get the corresponding `MDataEntries` with the following line of code:

```rust
let mdata_entries = context.object_cache().get_mdata_entries(entries_h)?;
```

Note the use of `context`; this is the context associated with the App Client, as described in [Client trait and implementations](./Client-trait-and-implementations). It is this context which stores the object cache, along with some other data, in the Client.

## FFI calling conventions

For more information about how calls to our FFI work, see [FFI calling conventions](./FFI-calling-conventions).

For even more details, see [FFI quirks and utils](./FFI-quirks-and-utils).