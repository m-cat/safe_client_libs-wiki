Here we discuss some of the finer points of our FFI calling conventions that were glossed over or not covered in the main [FFI calling conventions](FFI-calling-conventions) page.

## Error handling

Recall the first example on the [calling conventions](FFI-calling-conventions) page:

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

To make things as simple as possible, this example didn't demonstrate any error handling apart from catching panics (which is an absolute requirement).

In the following example we call from the FFI function into a regular, non-FFI function which returns a `Result`. Let's take a look:

```rust
use ffi_utils::call_result_cb;
use ffi_utils::test_utils::TestError;
use ffi_utils::{catch_unwind_cb, FfiResult, NativeResult, OpaqueCtx, FFI_RESULT_OK};
use std::os::raw::c_void;

// Function that returns a Result.
fn multiply_by_42(input_param: i32) -> Result<i32, TestError> {
    let (output, overflow) = input_param.overflowing_mul(42);
    if overflow {
        Err(TestError::FromStr("Overflow detected and prevented".into()))
    } else {
        Ok(output)
    }
}

// A typical FFI function. Returns `input_param * 42`.
#[no_mangle]
unsafe extern "C" fn foreign_function2(
    input_param: i32,
    user_data: *mut c_void,
    o_callback: extern "C" fn(user_data: *mut c_void, result: *const FfiResult, value: i32),
) {
    let user_data = OpaqueCtx(user_data);

    catch_unwind_cb(user_data, o_callback, || -> Result<_, TestError> {
        match multiply_by_42(input_param) {
            Ok(output) => o_callback(user_data.0, FFI_RESULT_OK, output),
            Err(e) => {
                call_result_cb!(Err::<(), _>(e), user_data, o_callback);
            }
        }

        Ok(())
    })
}
```

As you can see, we call the Rust function `multiply_by_42` which, instead of panicking, returns a `TestError` on multiplication overflow.

When no error occurs we return `FFI_RESULT_OK` as before, signaling success. On error, however, we now call the `call_result_cb` function. This converts the returned `Result` into an `FfiResult` that the caller can understand, doing the work of getting the error code and creating a C-style string for us, and then calling the callback for us. As you can see, this isn't too complicated with our utility functions.

**Note:** The examples presented in these docs are taken out of the [FFI Utils tests](https://github.com/maidsafe/ffi_utils/blob/master/tests/lib.rs). Feel free to download the repository and play around with the tests on your machine. And if you make a new test, we'd be more than happy to accept it as part of the test suite!

## Panics and unwinding

If you're writing tests or a quick & dirty app, it's usually fine to use `expect` or `unwrap` (still better used as a macro though, please follow [this guide in the QA repo](https://github.com/maidsafe/QA/blob/master/Documentation/Rust%20Style.md#unwrap)). For production code, however,  it's important to properly [handle errors in Rust](https://doc.rust-lang.org/book/ch09-00-error-handling.html) and it's even more important to properly handle errors at the FFI boundary. You must not use `unwrap` or `expect` in FFI because of panics and [stack unwinding](https://doc.rust-lang.org/nomicon/unwinding.html): when you call these functions and they fail, the Rust program begins to panic, which involves unwinding of the stack. Essentially, it means that the Rust runtime rolls back the call stack, calling the destructors, freeing memory, and seeing if there are any panic handlers. And if the Rust encounters a stack frame that's not out of its world (like e.g. a C or Java/JNI stack frame which has a different structure), the program would crash badly with errors like a segmentation fault, only because of a single error that could've been handled gracefully and prevented.

Hence it's a good idea to catch panics whenever you cross the FFI boundary (i.e. calling Rust functions from `unsafe extern "C"` functions). There's a special function in Rust's stdlib for this purpose called [`catch_unwind`](https://doc.rust-lang.org/beta/std/panic/fn.catch_unwind.html), which will stop stack unwinding at the point its getting called. You should use it to wrap all calls from FFI functions to Rust code. We also have a helper function in FFI Utils, [`catch_unwind_result`](https://docs.rs/ffi_utils/0.11.0/ffi_utils/fn.catch_unwind_result.html), which simplifies the usage of the try operator inside of the closure code block. To provide an example in pseudo-code:

```rust
unsafe extern "C" fn some_ffi_function() -> i32 {
 let res = catch_unwind_result(|| {
   rust_function_returning_result(...)?; // If there's a panic or an error, it will be returned immediately to the outer scope
   client_libs_function(...)?;

   Ok(()) // No panics or errors encountered during the Rust function calls - returning Ok(..)
 });

 return if res.is_ok() { 0 } else { -1 };

 // If there was a panic or an error, the error code -1 will be returned, otherwise the caller of our function will get 0 as a result,
 // meaning everything went well.
}
```

## Defining and using FFI types

If you are writing tests for the FFI from Rust itself, you will probably want to follow the guide in [Testing framework](./Testing-framework). However, to use the functions there, return types must implement `ReprC` so that they can be converted into native Rust types. What does this mean for us?

Recall in the example above how our FFI function accepted an `i32` input and returned an `i32` output. This works when using our test utilities on this function because `i32` is one of the primitive types that automatically implements the [`ReprC`](https://github.com/maidsafe/ffi_utils/wiki#ReprC) trait, which provides conversion functions between a native Rust type and its FFI equivalent. `i32` happens to be the same in both the Rust world and the C world, but this is not always the case, especially for more complicated types.

Let's take a look at the [`AppExchangeInfo`](https://docs.rs/safe_core/*/safe_core/ipc/req/struct.AppExchangeInfo.html) struct as an example:

```rust
pub struct AppExchangeInfo {
    pub id: String,
    pub scope: Option<String>,
    pub name: String,
    pub vendor: String,
}
```

We can't use this in an FFI function because it contains Rust-style strings as well as an `Option`, neither of which C can understand. For this reason we provide an [FFI version](https://docs.rs/safe_core/*/safe_core/ffi/ipc/req/struct.AppExchangeInfo.html) of this struct:

```rust
#[repr(C)]
pub struct FfiAppExchangeInfo {
    pub id: *const c_char,
    pub scope: *const c_char,
    pub name: *const c_char,
    pub vendor: *const c_char,
}
```

Notice that the struct is marked `pub` as well as `#[repr(C)]`: this is **very** important! Also note that this struct is nothing but pointers, which C is very happy with, and that the pointer type is `c_char` instead of the native Rust `char`.

### `into_repr_c`

To convert from the Rust version to the FFI version, we provide the function `to_repr_c`:

```rust
impl AppExchangeInfo {
    /// Construct FFI wrapper for the native Rust object, consuming self.
    pub fn into_repr_c(self) -> Result<FfiAppExchangeInfo, IpcError> {
        let AppExchangeInfo {
            id,
            scope,
            name,
            vendor,
        } = self;

        Ok(FfiAppExchangeInfo {
            id: CString::new(id).map_err(StringError::from)?.into_raw(),
            scope: if let Some(scope) = scope {
                CString::new(scope).map_err(StringError::from)?.into_raw()
            } else {
                ptr::null()
            },
            name: CString::new(name).map_err(StringError::from)?.into_raw(),
            vendor: CString::new(vendor).map_err(StringError::from)?.into_raw(),
        })
    }
}
```

Here we convert the Rust-style strings into FFI strings using `CString::into_raw()`, which creates a pointer to a C-style string that won't get dropped. Note that this pointer is now *owned* by the FFI object, as it has to ensure the data gets cleaned up effectively (by implementing `Drop`; see below).

Notice that the `Option` is only converted to a C-style string pointer if it was `Some`; otherwise, it is set to `null`, indicating the absence of a value.

### `clone_from_repr_c`

To convert in the other direction, we provide `clone_from_repr_c` (which is actually a required method of the `ReprC` trait):

```rust
impl ReprC for AppExchangeInfo {
    type C = *const FfiAppExchangeInfo;
    type Error = IpcError;

    /// Constructs the object from a raw pointer.
    ///
    /// After calling this function, the raw pointer is owned by the resulting object.
    unsafe fn clone_from_repr_c(repr_c: Self::C) -> Result<Self, Self::Error> {
        let FfiAppExchangeInfo {
            id,
            scope,
            name,
            vendor
        } = *repr_c;

        Ok(Self {
            id: from_c_str(id).map_err(StringError::from)?,
            scope: if scope.is_null() {
                None
            } else {
                Some(from_c_str(scope).map_err(StringError::from)?)
            },
            name: from_c_str(name).map_err(StringError::from)?,
            vendor: from_c_str(vendor).map_err(StringError::from)?,
        })
    }
}
```

This is basically the `into_repr_c` operation in reverse, with one important caveat: we are *cloning* from the FFI representation, not consuming it. This means that the C-style strings are cloned and brand new strings are created in Rust representation, but the original C strings are still valid, and will still need to be cleaned up.

Note also that we check if `scope` is `null`, as it is an option type and may not carry a value.

By implementing `ReprC`, we allow this object to be returned in callbacks using our FFI strategy.

### `Drop`

Finally, it is very important to define a custom `Drop` implementation for the FFI struct:

```rust
impl Drop for FfiAppExchangeInfo {
    #[allow(unsafe_code)]
    fn drop(&mut self) {
        unsafe {
            let _ = CString::from_raw(self.id as *mut _);
            if !self.scope.is_null() {
                let _ = CString::from_raw(self.scope as *mut _);
            }
            let _ = CString::from_raw(self.name as *mut _);
            let _ = CString::from_raw(self.vendor as *mut _);
        }
    }
}
```

In order to drop the pointers that were created with `CString::into_raw`, we have to reverse the operation with `CString::from_raw`, which passes ownership of the data back into the Rust lifetime model, which now knows that it can drop the data when it goes out of scope.

**Note:** You do not need to explicitly drop objects that already clean up their data when going out of scope, such as integers, arrays, and even most structs.

### Summary

In summary, there are two important things to note here:

1. **All FFI type should own their data.** If the FFI type's pointers pointed to data elsewhere, that data could go out of scope and become invalid. It is thus very important to keep in mind that FFI types keep their own data, preventing such subtle bugs, but you must not forgot to manually clean it up in `Drop`.
1. **`Option` is represented as null in FFI.**

Finally, not all custom FFI types will carry pointers, and may just be a collection of boolean flags or integer values. In such cases, the conversions are much simpler, and you do not have to manually implement `Drop`.

### Handling vectors

You'll also commonly see vectors in struct fields in the SCL FFI. These are a bit trickier to handle, so let's go through it step by step.

Let's say we have this simple native Rust struct:

```rust
pub struct MDataKey(
    pub Vec<u8>,
);
```

The equivalent C-representation struct, containing the same information:

```rust
#[repr(C)]
#[derive(Debug)]
pub struct MDataKey {
    pub key: *const u8,
    pub key_len: usize,
}
```

This contains all of the necessary internal data of a `Vec`: a pointer to its data and the length of the `Vec` (the capacity of the `Vec` is shrunk to fit the length). **Note:** You may still see some old code which includes the capacity as well. Removal of this field has been a [recent change](https://github.com/maidsafe/ffi_utils/pull/28).

To convert between native and FFI representation, we use several helper functions from [ffi_utils](https://github.com/maidsafe/ffi_utils/blob/master/src/vec.rs).

To *convert* to C representation, we use `vec_into_raw_parts`:

```rust
impl MDataKey {
    /// Construct FFI wrapper for the native Rust object, consuming self.
    pub fn into_repr_c(self) -> FfiMDataKey {
        let (key, key_len) = vec_into_raw_parts(self.0);

        FfiMDataKey {
            key,
            key_len,
        }
    }
}
```

To *clone* from C representation, we use `vec_clone_from_raw_parts`:

```rust
impl ReprC for MDataKey {
    type C = *const FfiMDataKey;
    type Error = ();

    unsafe fn clone_from_repr_c(repr_c: Self::C) -> Result<Self, Self::Error> {
        let FfiMDataKey { key, key_len } = *repr_c;
        let key = vec_clone_from_raw_parts(key, key_len);

        Ok(MDataKey(key))
    }
}
```

To *drop* the `Vec` and its data, which is now all owned by the FFI struct, we use the standard library function `vec_from_raw_parts`:

```rust
impl Drop for FfiMDataKey {
    #[allow(unsafe_code)]
    fn drop(&mut self) {
        let _ = unsafe { vec_from_raw_parts(self.key as *mut u8, self.key_len) };
    }
}
```

## Calling callbacks

In this section we'd like to clarify some points about calling the user-provided callbacks. For example, let's say we have this FFI function signature, with a callback called `o_cb`:

```rust
/// Returns the name of the app's container.
#[no_mangle]
pub unsafe extern "C" fn app_container_name(
    app_id: *const c_char,
    user_data: *mut c_void,
    o_cb: extern "C" fn(
        user_data: *mut c_void,
        result: *const FfiResult,
        container_name: *const c_char,
    ),
)
...
```

As opposed to creating an FFI object, where we pass ownership of data to the new object (see above), we do not pass owned data to the callback (unless it is a `Copy` type). Instead, we pass *pointers* to the data, and the caller is expected to make a copy of the data within the callback. For this reason, it's vitally important to make sure that the data lives long enough -- that is, at least until the callback has completed.

For example, this is not correct, as the pointed-to data (created by `CString::new`) may be dropped:

```rust
...
{
    catch_unwind_cb(user_data, o_cb, || -> Result<_, AppError> {
        let name = safe_core::app_container_name(
            CStr::from_ptr(app_id).to_str()?,
        );
        o_cb(user_data, FFI_RESULT_OK, CString::new(name)?.as_ptr());
        Ok(())
    })
}
```

Instead, we need to make sure that the data lives long enough -- in this case, binding the result of `CString::new` to a variable:

```rust
...
{
    catch_unwind_cb(user_data, o_cb, || -> Result<_, AppError> {
        let name = CString::new(safe_core::app_container_name(
            CStr::from_ptr(app_id).to_str()?,
        ))?;
        o_cb(user_data, FFI_RESULT_OK, name.as_ptr());
        Ok(())
    })
}
```

**Note:** When passing a `Vec` to a callback, it is also important to use the `.as_safe_ptr()` method instead of `.as_ptr()`:

```rust
o_cb(user_data, FFI_RESULT_OK, v.as_safe_ptr(), v.len());
```

See [`SafePtr`](https://docs.rs/ffi_utils/*/ffi_utils/trait.SafePtr.html) for more details.

## Opaque handles

There is one final thing to be aware of. We provide FFI versions of some of our structs, such as [`MDataInfo`](https://docs.rs/safe_core/*/safe_core/ffi/struct.MDataInfo.html), allowing the caller to inspect and modify such structs locally without performing FFI calls. However, we do not expose all of the structs you may encounter as part of our FFI.

For example, the [`app_registered`](https://docs.rs/safe_app/*/safe_app/ffi/fn.app_registered.html) function returns a pointer to an [`App`](https://docs.rs/safe_app/*/safe_app/struct.App.html) struct, but we do not expose the fields or methods of `App`. Instead, for the purposes of our FFI, pointers to `App` can be considered [opaque handles](https://en.wikipedia.org/wiki/Opaque_pointer), and you can't do anything with them apart from pass them into other FFI functions, such as [`file_open`](https://docs.rs/safe_app/*/safe_app/ffi/nfs/fn.file_open.html). This is because of the difficulty in providing complex structs as FFI-compatible objects -- plus, it is dangerous to allow a caller to directly modify a struct like `App` as it can lead to bugs or even security issues.
