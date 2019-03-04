The `Client` trait provides an interface for self-authentication client implementations. Its implementers are the `CoreClient`, `AuthClient`, and `AppClient` structures. The trait provides functionality for interfacing all requests from high-level APIs to the actual routing layer, and for managing all interactions with it.

Clients are non-blocking, with an asynchronous API using the futures abstraction from the futures-rs crate.

## Some history

`Client` used to be [a single structure](https://github.com/maidsafe/safe_client_libs/blob/1a26890ca211bc0380b095e6468f8a0107b24e06/safe_core/src/client/mod.rs#L111) which lived in Safe Core.

It had to be in SAFE Core because it was used by both SAFE Auth and SAFE App, as well as by SAFE Core itself (but only in tests). However, it contained SAFE Auth-only functionality like the `registered()` and `login()` functions, as well as support for [access containers](https://github.com/maidsafe/safe_client_libs/blob/1a26890ca211bc0380b095e6468f8a0107b24e06/safe_core/src/client/mod.rs#L868). This "reverse dependency" of SAFE Core on the needs of Safe Auth (and SAFE App as well) was awkward and poor design.

Another problem with the old struct was that it required a type parameter for the context type. Only Clients in SAFE App had an associated context, which meant that Clients in SAFE Core and SAFE Auth always needed an unused type parameter: [example](https://github.com/maidsafe/safe_client_libs/blob/1a26890ca211bc0380b095e6468f8a0107b24e06/safe_core/src/utils/test_utils/mod.rs#L97). Clearly, this "one-size-fits-all" approach to the Client abstraction was not optimal.

## The current architecture

What used to be a single struct has been split out into:

- The `Client` trait, containing functionality common across SAFE Core, SAFE Auth, and SAFE App.
- The `ClientInner` struct, containing inner fields expected by the trait.
- A `CoreClient` implementing struct, a Client type used by SAFE Core in tests.
- An `AuthClient` implementer, the Client type used by SAFE Auth which provides functions like `register()` and `login()`.
- An `AppClient` implementer, the Client type used by SAFE App which provides functions like `unregistered()`.

## The Client trait

Defined in [safe_core::client::mod.rs](https://github.com/maidsafe/safe_client_libs/blob/master/safe_core/src/client/mod.rs).

The `Client` trait provides an interface of functionality that its implementers are expected to define. For example, its implementers may contain the address of the Client Manager, so they must define `cm_addr()`, and so on. Much of the heavy lifting is done by the `Client` trait, however, and implementers immediately have access to functions like `get_idata()`, `put_idata()`, and so on, without having to implement them.

Note that this trait extends the `Clone` trait, so all implementers must implement `clone` as well. It also has the `'static` lifetime, so implementers must be `Sync`. This lets us use Clients safely across threads.

Functions in SAFE Core expect a generic `Client` as an input so that they can be called using any implementing type: [example](https://github.com/maidsafe/safe_client_libs/blob/2e0ac2b49e94d7c3f837be4afcb11498ef10ba0c/safe_core/src/nfs/dir.rs#L19).

### `ClientInner`

Defined in [safe_core::client::mod.rs](https://github.com/maidsafe/safe_client_libs/blob/master/safe_core/src/client/mod.rs).

This inner struct contains fields expected by the trait. This only really exists because traits [cannot have associated fields yet](https://github.com/nikomatsakis/fields-in-traits-rfc), so the idea is that implementing Clients compose this struct and return it when [`inner()`](https://github.com/maidsafe/safe_client_libs/blob/bc0230a7a271ceb935692b15b8d205d64706b082/safe_core/src/client/mod.rs#L112) is called. Then, functions implemented by the trait can call `inner()` to get fields for the trait: [example](https://github.com/maidsafe/safe_client_libs/blob/bc0230a7a271ceb935692b15b8d205d64706b082/safe_core/src/client/mod.rs#L144). Unfortunately, for now this has an extra level of indirection as implementing Clients are expected to wrap `ClientInner` using the `Rc<RefCell<>>` pattern.

## Core Client

Defined in [safe_core::client::core_client.rs](https://github.com/maidsafe/safe_client_libs/blob/master/safe_core/src/client/core_client.rs).

`CoreClient` is a barebones Client type, used solely for testing purposes in SAFE Core, and does not appear in the production API. It is therefore not strictly needed, but it allows functions which are provided by SAFE Core, such as the NFS functions, to be tested in SAFE Core itself.

This Client implements the interface required by `Client`, as well as a `new` function to allow it to be created (the `Client` trait does not provide this, as `AuthClient` and `AppClient` are created in different ways). Apart from that, it is just a wrapper around the `ClientInner` type, with no associated context.

## Auth Client

Defined in [safe_authenticator::client.rs](https://github.com/maidsafe/safe_client_libs/blob/master/safe_authenticator/src/client.rs).

`AuthClient` is the Authenticator Client. It can be created using the `login()` and `registered()` functions, which can be considered the entrypoints into SAFE Authenticator. These functions themselves are only crate-public, and must be accessed through the corresponding wrappers such as [`Authenticator::login`](https://docs.rs/safe_authenticator/0.8.0/safe_authenticator/struct.Authenticator.html#method.login).

It is a wrapper around `ClientInner` as well as `AuthInner`, which are both contained in `Rc<RefCell<>>` so that the `AuthClient` object can be cloned easily. `AuthInner` contains associated fields such as the account, the user credentials, and the session packet version. `AuthClient` provides several functions for getting and setting these fields, in addition to implementing the required interface.

## App Client

Defined in [safe_app::client.rs](https://github.com/maidsafe/safe_client_libs/blob/master/safe_app/src/client.rs).

`AppClient` is the App Client. It can be created using the `unregistered()` and `from_keys()` functions. `from_keys()` is called by `App::registered()` to create a registered app client.

It is a wrapper around `ClientInner` as well as `AppInner`, which are both contained in `Rc<RefCell<>>` so that the `AppClient` object can be cloned easily. The `AppInner` struct contains associated fields, which are all `Option`s, as calling `unregistered()` does not populate these values.

The App Client is the only one that has an associated context. In the case of an unregistered client, the context only contains the object cache; in the case of a registered client, it additionally contains the app ID, the symmetric encryption key, the access container info, and the access info.
