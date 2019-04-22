## Environment variables

The following environment variables can be set to enable custom options. Each one has higher precedence than its respective config file option (see [Configuring Client Libs](./Configuring-Client-Libs#safe_coreconfig)). SAFE Core does not need to be rebuilt for these environment variables to take effect.

**Note:** An environment variable is considered "set" if it is set to *any* value. It must be deleted (`unset` on Unix) in order for its setting to be disabled.

The environment variables are as follow (see [Mock vs. non-mock](./Mock-vs.-non-mock) for more information):

* `SAFE_MOCK_UNLIMITED_MUTATIONS`: If set, switch off the mutations limit in mock-vault. If SAFE Core is built with `--features=mock-network`, then setting this option will allow an unlimited number of mutations.
* `SAFE_MOCK_IN_MEMORY_STORAGE`: If set, use a memory store instead of a file store in Mock Vault. If SAFE Core is built with `--features=mock-network`, then setting this option will use Mock Vault's memory store, which is faster than reading/writing to disk.
* `SAFE_MOCK_VAULT_PATH`: If this is set and file storage is being used (`mock_in_memory_storage` is `false`), use this as the path for the `MockVault` file.

## Configuration files

There are three configuration files relating to Crust, Routing, and SAFE Core.

Most commonly, the configuration files can be found near the executable file and the file naming scheme follows the usual pattern from [`config_file_handler`](https://github.com/maidsafe/config_file_handler/) -- i.e., `<executable name>.<library name>.config`. That is, if your app executable name is `todo`, the config file would be named `todo.safe_core.config`.

### \*.safe\_core.config

Default config:

```json
{
    "dev": {
        "mock_unlimited_mutations": false,
        "mock_in_memory_storage": false,
        "mock_vault_path": null
    }
}
```

The configurable parameters are as follow (see [Mock vs. non-mock](./Mock-vs.-non-mock) for more information):

* `mock_unlimited_mutations`: If `true`, switch off the default mutations limit in Mock Vault, allowing an unlimited number of mutations.
* `mock_in_memory_storage`: If `true`, use memory store instead of file store in Mock Vault. It increases performance at the cost of permanent storage, so use this only if you're OK with losing all your data after an app shutdown. We use this setting for running tests.
* `mock_vault_path`: If `true` and `mock_in_memory_storage` is `false`, set the Mock Vault storage path. Defaults to the system temporary directory.

### \*.crust.config

Default config:

```json
{
    "hard_coded_contacts": [],
    "tcp_acceptor_port": null,
    "force_acceptor_port_in_ext_ep": false,
    "service_discovery_port": null,
    "bootstrap_cache_name": null,
    "whitelisted_node_ips": null,
    "whitelisted_client_ips": null,
    "network_name": null,
    "dev": {
        "disable_external_reachability_requirement": false
    }
}
```

You can follow the [Crust documentation](https://docs.rs/crust/0.31.0/crust/struct.Config.html) for an explanation of each configuration parameter.

### \*.routing.config

Default config:

```json
{
    "dev": {
        "allow_multiple_lan_nodes": false,
        "disable_client_rate_limiter": false,
        "disable_resource_proof": false,
        "min_section_size": null
    }
}
```

You can follow the [Routing documentation](https://docs.rs/routing/0.37.0/routing/struct.DevConfig.html) for an explanation of each configuration parameter.

**Note:** the *network size* must exactly match the network size defined on the vault size (i.e. in the `safe_vault.routing.config`). Otherwise, while performing basic network operations like creating an account, you might see some strange effects like hanging or 'freezing'. Refer to the dev forum thread ['Trying to build a global network'](https://forum.safedev.org/t/trying-to-build-a-global-network/2266) for a case study and more information.
