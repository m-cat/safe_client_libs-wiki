We follow a very specific convention for all functions we export to be consumed externally in our [FFI](https://en.wikipedia.org/wiki/Foreign_function_interface). This convention has evolved during the SAFE Client Libs design process, and was chosen as the most robust approach for us.

In this document, we assume prior knowledge of [raw pointers in Rust](https://doc.rust-lang.org/book/ch19-01-unsafe-rust.html#dereferencing-a-raw-pointer), [unsafe Rust](https://doc.rust-lang.org/book/ch19-01-unsafe-rust.html), and [Rust FFI](https://doc.rust-lang.org/1.2.0/book/ffi.html) generally.

## Convention

A typical Rust function that takes a parameter and returns a result would look like this:

```rust
pub fn native_function(input_param: i32) -> Result<i32, Error> {
    Ok(input_param * 42)
}
```

When translated to use our foreign calling convention, it looks very different:

```rust
use ffi_utils::test_utils::{call_1, TestError};
use ffi_utils::{catch_unwind_cb, FfiResult, OpaqueCtx, FFI_RESULT_OK};
use std::os::raw::c_void;

#[no_mangle]
pub unsafe extern "C" fn foreign_function(
    input_param: i32,
    user_data: *mut c_void,
    o_callback: extern "C" fn(user_data: *mut c_void,
                              result: *const FfiResult,
                              value: i32))
{
    let user_data = OpaqueCtx(user_data);

    catch_unwind_cb(user_data, o_callback, || -> Result<_, TestError> {
        o_callback(user_data.0, FFI_RESULT_OK, input_param * 42);
        
        Ok(())
    })
}
```

You can see multiple things are going on here at once:

* The function is now marked `unsafe extern "C"`.
* The function has the `#[no_mangle]` annotation.
* There are new `user_data` and `o_callback` parameters, and the return value for the function is gone.
* The function body is wrapped in `catch_unwind_cb`.
* The error status is wrapped in `FFI_RESULT_OK`.
* The actual result (`input_param * 42`) is returned using `o_callback`.

Let's dive into these changes in more depth.

## Explanation and design rationale

### The function signature

As you may already be aware, different languages have different Application Binary Interfaces, or ABIs. A program written in another language will not usually be able to call Rust functions, as Rust has its own ABI. `extern "C"` indicates to the Rust compiler that this function should be compiled using the C ABI, which is the most common. As most programming languages understand this ABI, this means that our FFI is almost universally callable. The function is also marked `#[no_mangle]` to prevent name mangling so that the function is nameable by other languages. For more details, please see [the Rust book](https://doc.rust-lang.org/book/ch19-01-unsafe-rust.html#using-extern-functions-to-call-external-code).

This pattern is common to FFI functions defined in Rust, and is not something unique to our library.

#### Unsafe

Our new function is also marked `unsafe`. This is not technically a requirement of FFI functions, because although FFI functions usually make wide use of `unsafe`, it is possible for them to provide a safe interface around the underlying unsafe implementation. However, our functions make no pretense as to their safety and provide no guarantee that calling them is safe in all circumstances.

For example, we do not perform null checks on any pointers that are passed in as input. We made this decision not only for efficiency reasons and to prevent having large, bloated functions, but also because even a non-null pointer is no guarantee that the pointer points to valid data. For these reasons, we put the responsibility on the caller to provide valid pointers -- therefore, if the FFI is being called from Rust code (such as from our FFI test suite), it must wrap the FFI calls inside `unsafe` blocks, explicitly signaling that it has taken sufficient care to call these functions properly and understands the risks.

For consistency, almost all of our FFI functions are `unsafe`, as the majority of them take raw pointers as input.

### Why callbacks?

As you noticed, we return output values by calling a user-provided *callback function* rather than by the usual means (returning a value). There are two reasons for that.

First, the entire Client Libs [core is asynchronous](./Routing-event-loop) and we use *futures* to work with asynchronous I/O & computation in a more simple manner. Because we can't guarantee that an externally-called function will return its result immediately, and because we can't block the caller waiting for a result either, we use the pattern commonly found in C libraries dealing with asynchronicity: callbacks.

Another thing that it helps with is *memory management*, which is a challenging topic when applied to FFI. Consider a simple case when you want to return a structure. You allocate it on a heap and return a pointer to it:

```rust
fn retn_struct() -> *const SomeFfiStruct {
    let some_struct = Box::new(SomeFfiStruct {
      ...
    });
    Box::into_raw(some_struct) as *const _
}
```

This works as expected, but now you've made a user *responsible* for deallocation of memory associated with this function, and if you don't provide a `free` function or if a user never calls it, you'll get a memory leak.

Arguably, it's a common way to manage memory in C libraries, but it didn't work for us. Instead, we rely on Rust's [`Drop` semantics](https://doc.rust-lang.org/book/ch15-03-drop.html). All memory allocated by us is automatically cleaned up when a user's callback finishes execution:

```rust
fn retn_struct(o_cb: extern "C" fn(*const SomeFfiStruct))) {
    let some_struct = SomeFfiStruct {
       ...
    };
    o_cb(&some_struct)

    // SomeFfiStruct::drop(&mut some_struct); <-- will be inserted here
    // by the Rust compiler
}
```

This approach makes it a requirement for a user to *copy* the data they need in the callback, but it also allows us to make the majority of deallocations fully automated.

### `FfiResult` and `FFI_RESULT_OK`

We don't have a conventional way of transferring native Rust data structures, which have implementation-specific memory layouts. Instead, we need to have predictable structures that other languages, and most importantly C -- the *lingua-franca* & the lowest common denominator of systems languages -- can understand.

Because of this, we need to convert a native Rust `Result<..>` into a structure [defined by us](https://docs.rs/ffi_utils/0.11.0/ffi_utils/struct.FfiResult.html) specifically for passing the status of the function completion:

```rust
struct FfiResult {
    error_code: i32,
    description: *const c_char,
}
```

The `error_code` is a value with a special meaning: `0` means 'success', and non-zero means a failure. The error code is unique and it tells the exact cause of the failure (e.g.: `-301` stands for `ERR_FILE_NOT_FOUND`). Error codes are unique for [safe_app](https://docs.rs/safe_app/0.9.0/safe_app/#constants) and [safe_authenticator](https://docs.rs/safe_authenticator/0.9.0/safe_authenticator/#constants), and generally you *should not* hard-code numbers for error codes: instead, use the exported constants.

`FFI_RESULT_OK` is a handy variant of `FfiResult` which signifies that no error occurred. Its `error_code` is 0 and its `description` is a null pointer -- calling code should always check if it is null before trying to read it.

**Note:** You should not create an `FfiResult` directly; create `NativeResult` first and convert it to `FfiResult` using its `to_repr_c()` method. Always use `FFI_RESULT_OK` if no error occurred. For more about using `FfiResult` correctly, see the corresponding section in the [FFI Utils](https://github.com/maidsafe/ffi_utils/wiki#FfiResult) manual.

### `ErrorCode`

It is required for the error type returned within `catch_unwind_cb` to implement the [`ErrorCode`](https://docs.rs/ffi_utils/*/ffi_utils/trait.ErrorCode.html) trait. Errors must implement this in order to return `i32` error codes to the caller. `AuthError` and `AppError` in SAFE Authenticator and SAFE App already implement this, and for testing purposes, a `TestError` type is provided by FFI Utils that lets you skip the boilerplate of defining your own error type.

### What is `user_data` and `OpaqueCtx`?

All this and we still haven't explained the `user_data` parameter! It appears to be a mysterious one, as it is a void pointer (`*mut c_void`, where `c_void` is the Rust-provided name for the C `void` type). This means `user_data` needs to be cast to a pointer to an actual type before it can be dereferenced, but how would our FFI functions know what the type is? Moreover, our functions seem to never actually dereference it.

The clue lies in the name: it is "user data", arbitrary data that only the caller ("user") understands. It can be dereferenced in the `o_cb` callback function, because this function is provided by the caller along with `user_data`. Note that this is optional -- the caller can also pass in a null pointer for `user_data`. However, the caller probably wants to have an idea of when a called function has finished execution -- remember that our FFI functions run asynchronously. We therefore provide this mechanism so that the caller can pass in, say, a channel sender as `user_data`, derefence it in the callback function, and send data back to the main thread of the calling program.

You are probably wondering why we wrap `user_data` inside of an `OpaqueCtx` structure. Defined in [FFI Utils](https://docs.rs/ffi_utils/*/ffi_utils/struct.OpaqueCtx.html), `OpaqueCtx` gives the `*mut c_void` type the `Send` trait. The standard library does not implement `Send` for mutable pointers as two threads can both try to write to the same pointer at the same time. By overriding the compiler's guarantees with `OpaqueCtx`, you are saying that you understand the risks and are promising to only ever call `o_cb` in just one of the threads you create in an FFI function.

**Note:** As the example given does not actually create any new threads, wrapping `user_data` in `OpaqueCtx` is optional here; in fact, you will not see this pattern in many of our FFI functions. If you omit this pattern when it is needed, you will just get a compiler error.

### `catch_unwind_cb`

Finally, the body of the function must be wrapped in `catch_unwind_cb`, a function defined in [FFI Utils](https://github.com/maidsafe/ffi_utils/blob/master/src/catch_unwind.rs). This function itself is a wrapper around the Rust standard library function [`catch_unwind`](http://doc.rust-lang.org/1.33.0/std/panic/fn.catch_unwind.html).

The purpose of these functions is to capture potential panics that might occur at the FFI boundary, where a panic would be undefined behavior. See [FFI quirks and utils](./FFI-quirks-and-utils#Panics-and-unwinding) for a deeper discussion.

## Calling from Rust

There is usually no reason to call FFI functions from Rust, but FFI functions have to be tested somehow, so we have a collection of utilities to make them easier to test from Rust code. See [Testing framework](./Testing-framework) for the details.