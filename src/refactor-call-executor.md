# refactor call executor

| Repository                   | Pull Request                 | Code |
|------------------------------|------------------------------|------|
| [patractlabs/vertic][vertic] | [patractlabs/vertic#88][#88] |      |

This sample kick out the wasm exection from substrate client, and removed a lot of
generics.

## 0. construct a native-only executor

```rust
//! vertic/client/service/src/client/executor.rs

pub struct NativeExecutor {
    /// NOTE:
    ///
    /// use a simple pointer instead of a customized trait
    ///
    pub dispatch: fn(&str, &[u8]) -> Option<Vec<u8>>,
    /// native version
    pub version: NativeVersion,
}

impl ReadRuntimeVersion for NativeExecutor {
    fn read_runtime_version(
        &self,
        _wasm_code: &[u8],
        _ext: &mut dyn Externalities,
    ) -> std::result::Result<Vec<u8>, String> {
        Ok(self.version.runtime_version.encode())
    }
}

impl RuntimeVersionOf for NativeExecutor {
    fn runtime_version(
        &self,
        _ext: &mut dyn Externalities,
        _runtime_code: &RuntimeCode,
    ) -> Result<RuntimeVersion> {
        Ok(self.version.runtime_version.clone())
    }
}

impl CodeExecutor for NativeExecutor {
    type Error = Error;

    fn call<
        R: Decode + Encode + PartialEq,
        NC: FnOnce() -> result::Result<R, Box<dyn std::error::Error + Send + Sync>> + UnwindSafe,
    >(
        &self,
        ext: &mut dyn Externalities,
        _runtime_code: &RuntimeCode,
        method: &str,
        data: &[u8],
        use_native: bool,
        native_call: Option<NC>,
    ) -> (Result<NativeOrEncoded<R>>, bool) {
        let result = match (use_native, native_call) {
            (true, Some(call)) => with_externalities_safe(ext, move || (call)())
                .and_then(|r| r.map(NativeOrEncoded::Native).map_err(Error::ApiError)),
            _ => match with_externalities_safe(ext, move || (self.dispatch)(method, data)) {
                Ok(v) => v
                    .map(NativeOrEncoded::Encoded)
                    .ok_or_else(|| Error::MethodNotFound(method.to_owned())),
                Err(e) => Err(e),
            },
        };

        (result, true)
    }
}
```

[vertic/client/service/src/client/executor.rs][executor]


## 1. use native executor in our node

```rust
pub fn new_partial(config: &Configuration) -> Result<Service, ServiceError> {
    let executor = NativeExecutor {
        dispatch: magnet_runtime::api::dispatch,
        version: magnet_runtime::native_version(),
    };
    
    // ...
}
```

[vertic/node/magnet/src/service.rs#L40][service]


[#88]: https://github.com/patractlabs/vertic/pull/88
[vertic]: https://github.com/patractlabs/vertic
[executor]: https://github.com/patractlabs/vertic/blob/master/client/service/src/client/executor.rs
[service]: https://github.com/patractlabs/vertic/blob/422dd12c0b096ebf65ad4065d17872d0e62ed9ac/node/magnet/src/service.rs#L40
