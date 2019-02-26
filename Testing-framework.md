Because SAFE Client Libs consists of many distinct components (Authenticator, the app framework, and FFI layer), we need a way to test them individually and in integration. This is what the Client Libs suite of test utilities provides us: our testing framework simplifies testing both Rust and FFI applications, and on this page we'll cover both.

We assume that you're familiar with the [FFI calling conventions](./FFI-calling-conventions) we use in SAFE Client Libs.

## Testing FFI

Although Rust fully supports calling FFI functions with callbacks, usually it's not very convenient and practical, given the asynchronous nature of most functions exposed in the Client Libs API.

To simplify testing, we use a generic pattern to synchronously wait for the results to come back out of async functions. It can be generalised as follows:

```rust
// We have the following API function defined somewhere in Client Libs:
unsafe extern "C" fn some_ffi_fn_from_safe_app(
    user_data: *mut c_void,
    o_cb: unsafe extern "C" fn(user_data: *mut c_void,
                               result: *const FfiResult,
                               some_value: u32)
);

// And we want to test it:
#[test]
fn testing_ffi() {
    // We create an async channel to send the result of the function execution
    let (tx, rx) = mpsc::channel::<u32>()

    // then we convert it into a raw pointer and pass it as a `user_data` argument
    some_ffi_fn_from_safe_app(&tx as *mut _,
                              callback);

    // .. then we block the thread execution until we receive the value back:
    let some_value = unwrap!(rx.recv());
    assert!(some_value, 42);

    unsafe extern "C" fn callback(user_data: *mut c_void,
                                  result: *const FfiResult,
                                  some_value: u32) {
        // in the callback we 'unpack' the sender part of the async channel from the user_data argument
        let tx: mpsc::Sender<u32> = unsafe { *user_data as *mut _ };
        // and send the value back to the receiver
        tx.send(some_value);
    }
}
```

Instead of setting up this pattern again and again, you can use [the `call_*` family of functions](https://docs.rs/ffi_utils/0.11.0/ffi_utils/test_utils/index.html) from the FFI Utils library. The following functions are available:

- [`call_0`](https://docs.rs/ffi_utils/0.11.0/ffi_utils/test_utils/fn.call_0.html): calls and waits for an FFI function that returns no results.
- [`call_1`](https://docs.rs/ffi_utils/0.11.0/ffi_utils/test_utils/fn.call_1.html): same for an FFI function that returns a single result.
- [`call_2`](https://docs.rs/ffi_utils/0.11.0/ffi_utils/test_utils/fn.call_2.html): same for an FFI function with two outputs.
- [`call_vec`](https://docs.rs/ffi_utils/0.11.0/ffi_utils/test_utils/fn.call_vec.html): calls and waits for an FFI function that returns an array, copies the data into a vector, and returns it as a result.
- [`call_vec_u8`](https://docs.rs/ffi_utils/0.11.0/ffi_utils/test_utils/fn.call_vec_u8.html): calls and waits for an FFI function that returns a string or a character array, copies the data into a `Vec<u8>`, and returns it as a result.

The manual code from above can now be significantly simplified:

```rust
#[test]
fn testing_ffi() {
    // call an FFI function and wait for the result
    let some_value =
        call_1(|user_data, callback| some_ffi_fn_from_safe_app(user_data, callback));
    assert!(some_value, Ok(42));
}
```

Generally, the usage of the above-mentioned functions follows the same pattern:

```rust
call_*(|user_data, callback| ffi_function(user_data, ..., callback))`.
```

Note that the compiler may complain that the type is unknown, in which case you'll need to add a type annotation. Here's a real example (note that the function `mdata_info_random_public` is unsafe, as are all of our FFI functions):

```rust
let mdata_info: MDataInfo = unsafe { unwrap!(call_1(|ud, cb| mdata_info_random_public(type_tag, ud, cb))) };
```

### Testing for errors

We allow the use of the `unwrap!` macro in tests if we expect an `Ok()` value, but you may also be testing error cases, in which case you can match on the error and expect `Err()`. Here's an example:

```rust
let size: Result<u64, i32> = unsafe { call_1(|ud, cb| file_size(&app, write_h, ud, cb)) };
match size {
    Err(code) if code == AppError::InvalidFileMode.error_code() => (),
    Err(x) => panic!("Unexpected: {:?}", x),
    Ok(_) => panic!("Unexpected success"),
}
```

Note that we test for the specific error code that we expect, rather than on just any error.

## Testing SAFE Authenticator

SAFE Authenticator provides a number of test utilities for common test operations such as creating test apps and quickly authenticating apps. The utilities can be found in the [safe_authenticator::test_utils](https://github.com/maidsafe/safe_client_libs/blob/master/safe_authenticator/src/test_utils.rs) module and require the `testing` [feature](https://github.com/maidsafe/safe_client_libs/wiki/Building-Client-Libs#features).

### Authenticating apps

First you will need to create an authenticator instance. The quickest way to do this is with the `create_account_and_login()` function:

```rust
let authenticator = test_utils::create_account_and_login();
```

To authenticate an app, you first need an app. `test_utils` provides a function for creating test apps:

```rust
let app = test_utils::rand_app();
```

Then, you need to create an authentication request:

```rust
let auth_req = AuthReq {
    app,
    app_container: false,
    containers: Default::default(),
};
```

The `app_container` field specifies whether the app requests its own dedicated container. If you need one, pass in `true`.

The `containers` field is the list of containers the app wishes to access and their desired permissions. Its type is `HashMap<String, ContainerPermissions>`.

We can now go ahead and authenticate the app:

```rust
let auth_granted = unwrap!(test_utils::register_app(&authenticator, &auth_req));
```

### Running Authenticator clients

Most functions that do anything useful (e.g. interacting with the network) require an Authenticator client as an input. To obtain a client that you can pass to these functions, use the `run` function:

```rust
use access_container;

let (entry_version, entries) = test_utils::run(&authenticator, |client| {
    access_container::fetch_authenticator_entry(client).map_err(AuthError::from)
});
```

As you can see, `run` expects an Authenticator as an input (you can use one obtained with `test_utils::create_account_and_login`) as well as a *closure*, which is Rust parlance for an anonymous function. The closure itself takes an Authenticator *client* as an input (this is the meaning of `|client|`). The `run` function provides `client` to the closure so that the body of the closure, which you supply with your code, has access to it. You can thus call functions such as `fetch_authenticator_entry` which expect an Authenticator client as input.

Note that this closure will panic on failure -- if you expect a failure to occur in your test, use `try_run` which returns the `Result` of a future.

## Testing SAFE App

SAFE App also provides several test utilities. These utilities are found in the [safe_app::test_utils](https://github.com/maidsafe/safe_client_libs/blob/master/safe_app/src/test_utils.rs) module and require the `testing` [feature](https://github.com/maidsafe/safe_client_libs/wiki/Building-Client-Libs#features).

### Running App clients

This is similar to running clients in Authenticator tests, with the main difference being that App clients have an associated *context* which is accessible from the `run` function. Context holds some internal data required for the app functioning, like e.g. the app keys or the app's access container entry. The context object also provides some [helper functions](https://github.com/maidsafe/safe_client_libs/blob/8c98425e7ec8e751528d6cb32a1dd3c900764273/safe_app/src/lib.rs#L425-L453) to work with the access container. In general though, you don't need to access it directly.

Here is an example from one of our tests. You'll first have to have created an app (affectionately called `app`):

```rust
let app = test_utils::create_app();
```

Now you can call `run`:

```rust
let container_info = test_utils::run(&app, move |client, context| {
    context.get_access_info(client).then(move |res| {
        let mut access_info = unwrap!(res);
        Ok(unwrap!(access_info.remove("_videos")).0)
    })
});
```
