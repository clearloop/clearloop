# method selector

| Repository               | Pull Request |
|--------------------------|--------------|
| [clearloop/ceres][ceres] |              |

this sample basicly introduces the selector implementation in ceres, which just 
like the implementation in `pallet-contracts`.

with this knowledge, you already can master the secret of passing arguments to the
executor of webassembly smart contract.


```rust
/// Parse `Vec<u8>` to `Vec<RuntimeValue>`
///
/// selector is a string with hex encoding, here we simply concat 
/// arguments to the selector, which will build the data ink needs
///
/// see https://paritytech.github.io/ink-docs/macros-attributes/selector/
pub fn parse_args(selector: &str, args: Vec<Vec<u8>>, tys: Vec<u32>) -> Result<Vec<u8>> {
    // signature check
    if args.len() != tys.len() {
        return Err(Error::InvalidArgumentLength {
            expect: tys.len(),
            input: args.len(),
        });
    }

    let mut res = step_hex(selector)
        .ok_or(Error::DecodeSelectorFailed)?
        .to_vec();
    for mut arg in args {
        log::debug!("-----> {:?}", arg);
        res.append(&mut arg);
    }

    Ok(res)
}
```

[ceres/crates/runtime/src/method.rs#L22][method]

for how to handle this data in the executor, plz read the next chapter.

[ceres]: https://github.com/clearloop/ceres
[method]: https://github.com/clearloop/ceres/blob/1a248b8335a9a229803a298ef373e5c4990a48bb/crates/runtime/src/method.rs#L22


