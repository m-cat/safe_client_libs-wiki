This page documents the data types that are central to understanding Client Libs. It doesn't cover every data type in Client Libs, and especially not the ones that are internal implementation details, but only the ones that are truly "core" to the Client Libs design.

## Immutable Data

The Immutable Data type is one of the fundamental data types in the network along with Mutable Data. Immutable Data has the properties that its value never changes and its hash is a deterministic XOR address. The SAFE Network takes advantage of these invariants to prevent duplicates from being stored on the network (a process called de-duplication). In other words, for any given Immutable Data, only one copy of it will ever exist on the network. This means that, given an Immutable Data value, you can calculate its hash and find its network location; given its hash, you can look up the value on the network.

### Definition

Immutable Data is not actually defined in Client Libs, but in [Routing](https://docs.rs/routing/*/routing/struct.ImmutableData.html). Nevertheless, it is a very simple object so we can cover it here. Its Client Libs API is covered below.

The structure of `ImmutableData` is as follows:

```rust
pub struct ImmutableData {
    name: XorName,
    value: Vec<u8>,
}
```

The fields are:

* `name`: The Xor Name for the data. It is a hash derived from the value itself, so it does not technically need to be stored explicitly except for performance reasons.
* `value`: The data stored.

There is a maximum size of Immutable Data, provided by [`MAX_IMMUTABLE_DATA_SIZE_IN_BYTES`](https://docs.rs/routing/*/routing/constant.MAX_IMMUTABLE_DATA_SIZE_IN_BYTES.html). Users of the Client Libs API do not have to worry about this limit because the API automatically ensures that a user does not surpass it. The API uses [Self Encryption](https://github.com/maidsafe/self_encryption) instead of dealing with Immutable Data directly. Self Encryption will split up data that's too big into appropriately-sized chunks.

### Immutable Data API

The Immutable Data API is defined in [safe_core::immutable_data](https://docs.rs/safe_core/*/safe_core/immutable_data/index.html) and provides functions to create Immutable Data and extract their values. The functions all have optional cryptographic keys as input, and if they are provided the Immutable Data is encrypted or decrypted automatically, as appropriate. The `create` function will ensure that the maximum Immutable Data size is respected.

The client types (see below) provide the functions `get_idata` and `put_idata` for getting and putting Immutable Data from and to the network.

Note that SAFE App also provides an [Immutable Data FFI API](https://docs.rs/safe_app/*/safe_app/ffi/immutable_data/index.html).

## Mutable Data

The Mutable Data type is the other fundamental data type in the network and is also defined in Routing.

It is more complex than Immutable Data so we cover it separately on the [Mutable Data](./Mutable-Data) page.

### Mutable Data API

The API for working with local Mutable Data is provided by [Routing](https://docs.rs/routing/*/routing/struct.MutableData.html) (it is re-exported by SAFE App, making a direct dependency on Routing is unnecessary). This API provides functions for creating Mutable Data, getting information about it, and mutating it. 

None of these functions interface with the network -- you'll need access to a type implementing `Client` for that. The client types (see below) provides various functions for dealing with Mutable Data on the network. These include getting and putting entire Mutable Data, listing and setting partial information about it, and changing its owner.

Client Libs provides an API for [MDataInfo](https://docs.rs/safe_core/*/safe_core/client/mdata_info/index.html).

SAFE App provides a [Mutable Data FFI API](https://docs.rs/safe_app/*/safe_app/ffi/mutable_data/index.html) which provides FFI access to the Routing and Client APIs described above.

## The client types

There are three Client types, implementing the `Client` trait, that enable interfacing requests from high-level APIs to the routing layer. These types are `CoreClient`, `AuthClient`, and `AppClient`. A full treatment of these types can be found on the [Client trait and implementations](./Client-trait-and-implementations) page.

## `Account` and `ClientKeys`

The [`Account`](https://docs.rs/safe_core/*/safe_core/client/account/struct.Account.html) structure is a representation of the user's account information on the network. Its definition is:

```rust
/// Representing the User Account information on the network.
#[derive(Clone, Debug, PartialEq, Deserialize, Serialize)]
pub struct Account {
    /// The User Account Keys.
    pub maid_keys: ClientKeys,
    /// The user's access container.
    pub access_container: MDataInfo,
    /// The user's configuration directory.
    pub config_root: MDataInfo,
    /// Set to `true` when all root and standard containers
    /// have been created successfully. `false` signifies that
    /// previous attempt might have failed - check on login.
    pub root_dirs_created: bool,
}
```

The [`ClientKeys`](https://docs.rs/safe_core/*/safe_core/client/account/struct.ClientKeys.html) object stores the various User Account keys. Its definition is:

```rust
/// Client signing and encryption keypairs
#[derive(Clone, Debug, PartialEq, Deserialize, Serialize)]
pub struct ClientKeys {
    /// Signing public key
    pub sign_pk: sign::PublicKey,
    /// Signing secret key
    pub sign_sk: shared_sign::SecretKey,
    /// Encryption public key
    pub enc_pk: box_::PublicKey,
    /// Encryption private key
    pub enc_sk: shared_box::SecretKey,
    /// Symmetric encryption key
    pub enc_key: shared_secretbox::Key,
}
```

The user's account can be queried from the network using the `get_account_info` method of the `Client` trait.