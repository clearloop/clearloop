# data exchange

| Repository               | Pull Request | Code |
|--------------------------|--------------|------|
| [clearloop/ceres][ceres] |              |      |

this sample introduces the shared data between wasm environment and the runtime
of smart contract.


## 0. overview, invoke contract method

```rust
// Invoke (ink) method
pub fn invoke(
    &mut self,
    method: InkMethod,
    inner_method: &str,
    args: Vec<Vec<u8>>,
    tx: Option<Transaction>,
) -> Result<Option<Vec<u8>>> {
    // construct transaction
    if let Some(tx) = tx {
        self.sandbox.tx = tx;
    }

    // set input
    self.sandbox.input = Some(method.parse(&self.metadata, inner_method, args)?);

    // get actived frame
    let hash = self
        .cache
        .borrow()
        .active()
        .ok_or(ceres_executor::Error::CodeNotFound)?;
        
    // execute method
    Executor::new(
        convert::to_storage_key(&hash[..]).ok_or(ceres_executor::Error::CodeNotFound)?,
        &mut self.sandbox,
    )?
    .invoke(&method.to_string(), &[], &mut self.sandbox)
    .map_err(|error| Error::CallContractFailed { error })?;

    // flush data
    self.cache
        .borrow_mut()
        .flush()
        .ok_or(Error::FlushDataFailed)?;
    Ok(self.sandbox.ret.take())
}
```

[ceres/crates/runtime/src/runtime.rs#L107][invoke]

this is a overview of invoking a method of a contract

0. set context to the sandbox
1. get actived frame/layer ( contract )
2. call method with sandbox
3. apply changes to storage


## 1. the sandbox

sandbox in ceres is exactly the shared environment between wasm executor and our runtime, we 
can put anything including functions into it.

in the implementation of the environment of ink! contract, sandbox also holds the interfaces
of our substrate chains, for example, `weight_price`, `block_height`, etc.

```rust
/// Wrap host function into `Func`
pub fn wrap_fn<T>(store: &Store, state: usize, f: usize, sig: FuncType) -> Func {
    let func = move |_: Caller<'_>, args: &[Val], results: &mut [Val]| {
        let mut inner_args = vec![];
        for arg in args {
            inner_args.push(from_val(arg.clone()));
        }

        // HACK the LIFETIME
        //
        // # Safety
        //
        // The runtime constructed only for one call, the changed data will be
        // written into storage after the lifetime
        //
        // NOTE: 
        //
        // here we pass the pointer of `T` to function directly to break
        // the lifetime limit
        //
        // in the latest wasmtime, we don't need to do this since there is a
        // state holding in the executor of wasmtime
        let state: &mut T = unsafe { mem::transmute(state) };
        let func: HostFuncType<T> = unsafe { mem::transmute(f) };
        match func(state, &inner_args) {
            Ok(Some(ret)) => {
                // # Safty
                //
                // This `result.len()` should always <= 1 since the length of
                // the result length of `HostFuncType` is 1
                if results.len() == 1 {
                    results[0] = to_val(ret);
                } else {
                    return Err(anyhow::Error::new(Error::UnExpectedReturnValue).into());
                }
                Ok(())
            }
            Ok(None) => Ok(()),
            Err(e) => Err(match e {
                Error::Return(data) => Trap::new(format!("0x{}", hex::encode(data.encode()))),
                e => anyhow::Error::new(e).into(),
            }),
        }
    };
    Func::new(store, sig, func)
}
```

[ceres/crates/executor/src/wasmtime/util.rs#L27][wrap-fn]


[ceres]: https://github.com/clearloop/ceres
[invoke]: https://github.com/clearloop/ceres/blob/1a248b8335a9a229803a298ef373e5c4990a48bb/crates/runtime/src/runtime.rs#L107
[wrap-fn]: https://github.com/clearloop/ceres/blob/1a248b8335a9a229803a298ef373e5c4990a48bb/crates/executor/src/wasmtime/util.rs#L27
