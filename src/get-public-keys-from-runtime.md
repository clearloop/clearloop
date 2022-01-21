# get public keys from runtime

| Repository                           | Pull Request                      |
|--------------------------------------|-----------------------------------|
| [Chainsafe/filecoindot][filecoindot] | [ChainSafe/filecoindot#181][#181] |

```rust
//! filecoindot/src/ocw/mod.rs#66

fn bootstrap<T: Config>(_: T::BlockNumber, urls: &[&str]) -> Result<()> {
    // NOTE
    //
    // get public keys from keystore
    let all_public: Vec<Vec<u8>> = <FilecoindotId as frame_system::offchain::AppCrypto<
        <Sr25519Signature as Verify>::Signer,
        Sr25519Signature,
    >>::RuntimeAppPublic::all()
    .into_iter()
    .map(|key| key.encode())
    .collect();
    
    // ...
    
}
```

[ChainSafe/filecoindot#181][#181]

the common usages of these public keys al for the session of staking, here we can get
the keys in offchain worker for validating them to execute our offchain logic.


[filecoindot]: https://github.com/ChainSafe/filecoindot
[#181]: https://github.com/ChainSafe/filecoindot/pull/181
