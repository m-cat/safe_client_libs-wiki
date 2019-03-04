## Background

This page documents which are the functions in the SAFE App API that interface with the network as well as how they do so (GETs and/or mutations). This information may be useful for app developers trying to design an efficient application. Their priority should be to minimize mutations as they are limited for accounts. However, while GETs are unlimited, a developer interested in efficiency should also try to keep network GETs low to minimize latency for the user.

## MutableData

| Function                      | GETs  | Mutations | Notes |
|-------------------------------|-------|-----------|-------|
| `mdata_put`                   | 0     | **1**     |       |
| `mdata_get_version`           | **1** | 0         |       |
| `mdata_serialised_size`       | **1** | 0         |       |
| `mdata_get_value`             | **1** | 0         |       |
| `mdata_entries`               | **1** | 0         |       |
| `mdata_list_keys`             | **1** | 0         |       |
| `mdata_list_values`           | **1** | 0         |       |
| `mdata_mutate_entries`        | 0     | **1**     |       |
| `mdata_list_permissions`      | **1** | 0         |       |
| `mdata_list_user_permissions` | **1** | 0         |       |
| `mdata_set_user_permissions`  | 0     | **1**     |       |
| `mdata_del_user_permissions`  | 0     | **1**     |       |

## MDataInfo

| Function                     | GETs | Mutations | Notes |
|------------------------------|------|-----------|-------|
| `mdata_info_new_private`     | 0    | 0         |       |
| `mdata_info_random_public`   | 0    | 0         |       |
| `mdata_info_random_private`  | 0    | 0         |       |
| ...                          |      |           |       |
| all `mdata_info_*` functions | 0    | 0         |       |

## ImmutableData

| Function                           | GETs  | Mutations | Notes                           |
|------------------------------------|-------|-----------|---------------------------------|
| `idata_new_self_encryptor`         | 0     | 0         |                                 |
| `idata_write_to_self_encryptor`    | 0     | 0         | done locally until SE is closed |
| `idata_close_self_encryptor`       | 0     | **1**     | +3 mut for medium files         |
| `idata_fetch_self_encryptor`       | **1** | 0         | +3 GET for medium files         |
| `idata_serialised_size`            | **1** | 0         |                                 |
| `idata_size`                       | 0     | 0         |                                 |
| `idata_read_from_self_encryptor`   | 0     | 0         |                                 |
| `idata_self_encryptor_writer_free` | 0     | 0         |                                 |
| `idata_self_encryptor_reader_free` | 0     | 0         |                                 |

### ImmutableData Notes

The actual number of GETs and mutations depends on the size of the data. Any idata at least 3kb in size will be self-encrypted to at least 3 chunks. Smaller than that, and the entire idata is held in the datamap. Each chunk is a separate GET/mutation. The maximum size of a single chunk is 3mb. So keep in mind that getting a whole idata will almost always be more expensive than e.g. getting the size.

## EntryActions

| Function                             | GETs | Mutations | Notes                                                                                         |
|--------------------------------------|------|-----------|-----------------------------------------------------------------------------------------------|
| all `mdata_entry_action_*` functions | 0    | 0         | All queue'd up actions perform a single mutation total when `mdata_mutate_entries` is called. |

## NFS

| Function          | GETs       | Mutations | Notes                                                                                                                      |
|-------------------|------------|-----------|----------------------------------------------------------------------------------------------------------------------------|
| `dir_fetch_file`  | **1**      | 0         |                                                                                                                            |
| `dir_insert_file` | **1**      | **1**     |                                                                                                                            |
| `dir_update_file` | **1**      | **1**     |                                                                                                                            |
| `dir_delete_file` | 0          | **1**     |                                                                                                                            |
|                   |            |           |                                                                                                                            |
| `file_open`       | **0 or 1** | 0         | 1 GET to get the datamap only if reading or writing in `APPEND` mode, +3 GET for medium files (except in `OVERWRITE` mode) |
| `file_size`       | 0          | 0         |                                                                                                                            |
| `file_read`       | 0          | 0         |                                                                                                                            |
| `file_write`      | 0          | 0         | done locally until file is closed                                                                                          |
| `file_close`      | 0          | **1**     | 1 mut to put the new datamap, +3 mut for a medium file                                                                     |

### NFS Notes

- `file_open` initially does 3 GETs for a medium file when writing in `APPEND` mode, same as reading. We need to get the existing data and decrypt so we can re-encrypt it again after writing – in `OVERWRITE` mode the existing data doesn’t matter.
- The amount of muts by `file_close` depends on the file size *after* writing, as you can open a small file and it can be medium or large by the time you’re done writing.
