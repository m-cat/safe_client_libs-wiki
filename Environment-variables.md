The following environment variables can be set to enable custom options. Each one has higher precedence than its respective config file option (see [Configuring Client Libs](./Configuring-Client-Libs#safe_coreconfig)). SAFE Core does not need to be rebuilt for these environment variables to take effect.

**Note:** An environment variable is considered "set" if it is set to *any* value. It must be deleted (`unset` on Unix) in order for its setting to be disabled.

## `SAFE_MOCK_UNLIMITED_MUTATIONS`

If set, switch off the mutations limit in mock-vault. If SAFE Core is built with `--features=use-mock-routing`, then setting this option will allow an unlimited number of mutations.

## `SAFE_MOCK_IN_MEMORY_STORAGE`

If set, use a memory store instead of a file store in Mock Vault. If SAFE Core is built with `--features=use-mock-routing`, then setting this option will use Mock Vault's memory store, which is faster than reading/writing to disk.

## `SAFE_MOCK_VAULT_PATH`

If this is set and file storage is being used (`mock_in_memory_storage` is `false`), use this as the path for the `MockVault` file.