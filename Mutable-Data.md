## Introduction

There are two fundamental data types in the SAFE Network: Mutable Data and Immutable Data. Mutable Data stores data in the form of _key-entry_ pairs, much like a [hash table](https://en.wikipedia.org/wiki/Hash_table). As the name implies, Mutable Data can be mutated, while Immutable Data cannot.

Each Mutable Data has an owner and owners are free to modify the data by adding new key-entry pairs or updating existing ones. Mutable Data provide access controls and permit setting fine-grained permissions to allow or forbid other applications to modify the Mutable Data contents. This feature also enables applications to explicitly ask users for permission to modify data. 

Mutable Data are versioned and their entries are versioned as well. Mutable Data can be either private or public.

## Structure of `MutableData`

The `MutableData` definition is as follows:

```rust
pub struct MutableData {
    name: XorName,
    tag: u64,
    data: BTreeMap<Vec<u8>, Value>,
    permissions: BTreeMap<User, PermissionSet>,
    version: u64,
    owners: BTreeSet<PublicKey>,
}
```

It is comprised of **six** fields:

- `name`: The location, given by a Xor Name, where the Mutable Data is stored in the network’s address space.
    - Xor Names can be either randomly generated or deterministically chosen. 
    - Random Xor Names are usually used to reduce predictability in the network.
    - Deterministic Xor Names are used in cases such as public names where the Xor Names are derived from the human-readable _public names_ themselves (e.g. by hashing them), allowing easier access to that public name's Mutable Data.
- `tag`: The type tag, a number that is used to differentiate between Mutable Data in the same address space.
    - Some type tags can have special meaning and behaviour. For instance, type tag `0` is reserved for storing _session packets_ (containing users account information).
    - Type tags 0 to 15,000 are reserved by MaidSafe.

- `version`: The Mutable Data version. Every change done to the `MutableData` shell results in a version increment.
    - Shell: All the `MutableData` fields except `data`.

-  `data`: The actual data, stored in the form of key-value pairs.
    - Every key maps to a `Value` that stores the data.
    - Mutable data has a maximum limit of **1MB** or **1000 entries**.
    - Every change to the `contents` field in `Value` is versioned and `entry_version` holds the latest version number.
    <br />

    ```rust
    pub struct Value {
        content: Vec<u8>,
        entry_version: u64,
    }
    ```

- `permissions`: A permissions list consisting of `User` and `PermissionSet` pairs. The `User` is represented by a signing public key and can be an app, a person, and so on.
    -  The available permissions are Insert, Update, Delete and ManagePermissions.
    - Permissions can be set either for specific applications with their public key or for anyone in general.
    <br />

    ```rust
    pub struct PermissionSet {
        insert: Option<bool>,
        update: Option<bool>,
        delete: Option<bool>,
        manage_permissions: Option<bool>,
    }
    ```

- `owners`: The list of users associated with the data, represented by public signing keys. 
    - These users always have full permissions with respect to the Mutable Data object and can mutate the permissions list. 
    - Currently only a single owner is supported. Apps cannot be set as owners and when adding a new Mutable Data to the network they must set this field to an owner's key.

## Metadata (`UserMetadata`):

The `UserMetadata` object describes the Mutable Data in a human readable format (both `name` and `description` are `String`s).

`UserMetadata` is not a part of the `MutableData` structure, and is implemented on the client side. However, the Metadata is stored as an entry in the Mutable Data itself. The key for the entry is `_metadata` and the value is the serialised `Metadata` structure.

Client Libs provides [helper functions](https://docs.rs/safe_app/*/safe_app/ffi/mutable_data/metadata/index.html) to work with the Metadata (including serialising and deserialising it).

### Structure of Metadata 

```rust
pub struct UserMetadata {
    pub name: Option<String>,
    pub description: Option<String>,
}
```

- `name`: The name or purpose of this Mutable Data. 
- `description`: A description of how this Mutable Data should or should not be shared.

## MutableData Info (`MDataInfo`):

`MDataInfo` decouples `MutableData` itself from the other information such as the cryptographic encryption key(s) and nonce.

The Routing layer knows only about the fundamental data types and not about their crypto properties, hence `MDataInfo` is used as a pointer to the `MutableData` (on the client side) that the Routing layer understands and facilitates encryption/decryption. In this regard it is similar to _data maps_ for Immutable Data. Because `MDataInfo` contains secret encryption keys, it has to be kept private at all times and must be shared with caution.

### Structure of MDataInfo

The `MDataInfo` definition is as follows:

```rust
pub struct MDataInfo {
    pub name: XorName,
    pub type_tag: u64,
    pub enc_info: Option<(Key, Nonce)>,
    pub new_enc_info: Option<(Key, Nonce)>,
}
```

It is comprised of **four** fields:

- `name`: Same as the `name` in `MutableData`.

- `type_tag`: Same as the `tag` in the `MutableData`.
    
- `enc_info`: A `(Key, Nonce)` pair containing the encryption key and the nonce required to encrypt/decrypt the contents.
     - A nonce is an arbitrary number used in cryptographic computations with a general rule that it must be used only once (hence the name).
     - In the case of Mutable Data it is used to derive deterministic nonces for entry keys (to be able to lookup a single value knowing a plaintext key).
    - It is always empty for public Mutable Data.

- `new_enc_info`: Also a ``(Key, Nonce)`` pair, this is used during re-encryption, which happens in a two-phase commit-like fashion.
    - Except during re-encryption, `new_enc_info` should be kept empty for private Mutable Data.
    - It is always empty for public Mutable Data.

## Limitations of Mutable Data:

- Mutable Data's values cannot be fully deleted. When the delete operation is performed, the value is just blanked out and the entry version incremented; the previous value is not retained. This means that currently data can be taken off the network, defying [SAFE Network Fundamental #8](https://safenetforum.org/t/safe-network-fundamentals-context/25352).
 
- The maximum size for a serialised `MutableData` structure is 1MB.

- The maximum number of entries per `MutableData` is 1000.
