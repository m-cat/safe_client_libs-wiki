This document is meant as a reference page for the Network File System (NFS) API, found [here](https://docs.rs/safe_app/*/safe_app/ffi/nfs/index.html).

## Files vs. File Handles

The `dir_*` set of functions all work directly with [`File`](https://docs.rs/safe_core/*/safe_core/ffi/nfs/struct.File.html) objects:

- `dir_delete_file`
- `dir_fetch_file`
- `dir_insert_file`
- `dir_update_file`

The `file_open` function also takes a `File` as input.

The `file_*` functions, on the other hand, all work on [`FileContextHandle`](https://docs.rs/safe_app/*/safe_app/ffi/object_cache/type.FileContextHandle.html) objects:

- `file_open`
- `file_close`
- `file_read`
- `file_size`
- `file_write`

Once a `FileContextHandle` is obtained from `file_open`, the rest of these operations all work locally, so no network operations are involved until `file_close` is called, returning a saved `File` (saved meaning its datamap has been PUT on the network). **The `FileContextHandle` is a local object and points to a local `SelfEncryptor` instance while `File` points to data on the network.**

See the [GET and PUT Info](./GET-and-PUT-reference) page for more information about the network costs of each of these operations.

## File Modes

### Read vs. Write

`file_open` can open a `File` in three modes:

- Overwrite
- Append
- Read

Append costs more network GETs than Overwrite because it has to retrieve the existing data in order to be able to append to it.

The File Context Handle returned from `file_open` will contain inner Reader and/or Writer handles, depending on whether it has been opened for reading, writing, or both.

**Note:** For more about the case when the handle contains both a Reader and a Writer, read "Simultaneous Read/Write" below for caveats.

**Note:** You can even open a file in both append and overwrite mode at once, in which case the mode defaults to append. We may change this to be an error in the future, so it is not recommended to rely on this behavior - see [here](https://github.com/maidsafe/safe_client_libs/issues/749).

### New (Empty) Files

A brand new File that hasn’t been written to will not have a valid data map and trying to open it in read or append modes will fail to retrieve the data and result in an error.

### Simultaneous Read/Write

It is valid to open a file in both read and write modes. How it works is like this: when you open a File you get a File Context Handle which points to either a Reader, a Writer, or both. These are stored locally. The Reader and Writer aren’t aware of each other, as they have their own internal state, so you can write new contents while still being able to read the original contents. When the file is closed, the Reader is dropped and the data written to the Writer is saved to a new file object which is returned.

It seems counterintuitive; a file opened for reading and writing will return a single file handle, but the internal Reader and Writer can have completely different contents. So you’d be using the same file handle for two different purposes which can be confusing and error-prone. This may change in the future - see [here](https://github.com/maidsafe/safe_client_libs/issues/732).

### Simultaneous Opens

It is valid to open the same file multiple times simultaneously. Again, each Reader/Writer has its own internal state, which will not be affected by any changes to the file since being opened.

Let’s say that you have two Writers opened, A and B. You call `file_close` on A as well as on B. You now have three distinct files: the original, the one returned by closing A, and the one returned by closing B. Any changes made in B will not affect A, etc.

Basically you can consider files, once opened, to exist as buffers locally. Any changes to the buffer do not affect existing buffers, and are only saved, to new files, when `file_close` is called. This design helps minimize unnecessary network requests.

### File Sizes

`file_size` gets the size of *the original `File` when opened for reading*. It’s not currently possible with the API to get the number of bytes written.

**Note:** `file_size` currently returns an error if a file is only opened in write mode. This may change in the future - see [here](https://github.com/maidsafe/safe_client_libs/issues/719).
