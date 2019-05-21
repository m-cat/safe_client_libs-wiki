# Mock Implementation

## Mock Vault

This is a mock implementation of the [vault](https://github.com/maidsafe/safe_vault) and its various operations:

- Storing data (in a local MockVault file by default, though this can be configured)
- Authorising operations like reads and mutations
- Committing mutations and counting them (limit of 1000 by default)

It can be configured, see [Configuration Parameters](./Mock-vs.-non-mock#configuration-parameters).

The data stored in the MockVault file can be moved between computers, so an error on an end user's computer can be reproduced locally and investigated by a developer.

The mock vault contains a `Cache` which mocks two authorities, the client manager and the NAE (Network Address Element) manager which manages client accounts and data (Mutable and Immutable) respectively. They hold references to `Account` and `Data` objects mapped against a unique identifier (XorName for Account and DataId for Data). This Cache is held in memory / stored in a file according to the configuration specified.

## Mock Routing

This module has the same interface as [routing](https://github.com/maidsafe/routing) and works with the mock vault instead of vaults. All of the operations are mocked, but made to be as realistic as possible, even incurring some delays via `sleep` so that they are not nearly-instantaneous.

Mock routing allows inserting hooks to override a request or response for testing purposes. This allows the testing framework to handle the various types of responses that routing sends back to the clients.

There are some additional APIs that can be used to simulate events such as disconnections and network timeouts for testing the behaviour of the clients in such scenarios. For example, operations recovery i.e. how the libraries are able to robustly handle network interruptions.

## Mock Account

The client manager authority maps a client's XoR name (i.e. SHA3 hash of the Client's public key) to a mock account object. This object keeps a track of the number of mutations performed and the remaining mutations available along with other account properties such as storing the public key of apps that have been authenticated by a user for an account. The version of the account data is also maintained and validated before every mutation. These properties are tracked separately for each account. 

Multiple accounts can be created for testing purposes without the need of a valid invitation token. However, to keep the API uniform across mock and non-mock environments the invite token field is still required. On the mock network, any random alphanumeric can be used as an invite token. It will not be validated.

## Testing

When testing, separate instances of `MockRouting` can be created with custom configuration parameters, i.e. to turn off the mutation limit for one test only. This can be done with the `setup_with_config` API which is conditionally compiled only for testing. For an example of its usage see the test `low_balance_check`.

Note that by default, a single shared instance of `Vault` is created for all tests, and new instances of `MockRouting` will interface with this shared instance. However, the `setup_with_config` API will create a separate, custom vault, with e.g. the mutation limit turned off or in-memory storage turned on, so that other tests are not affected.