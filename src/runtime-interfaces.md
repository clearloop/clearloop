# runtime interfaces

| Repository                           | Pull Request                    | Code |
|--------------------------------------|---------------------------------|------|
| [ChainSafe/filecoindot][filecoindot] | [ChainSafe/filecoindot#89][#89] |      |

this sample shows how to use runtime interfaces to call `std` methods in pallets.

original learned this in [patractlabs/europa][europa] for zk-proof, worked on a forked branch
which is deleted now, helped willes implement this in `#89`.


## 0. write interfaces with `#[runtime_interface]`

```rust
//! filecoindot-io/src/lib.rs

use sp_runtime_interface::runtime_interface;
use sp_std::vec::Vec;

#[runtime_interface]
pub trait ForestProofVerify {
    fn verify_receipt(proof: Vec<Vec<u8>>, cid: Vec<u8>) -> Option<()> {
        use filecoindot_proofs::{ForestAmtAdaptedNode, ProofVerify, Verify};
        ProofVerify::verify_proof::<ForestAmtAdaptedNode<String>>(proof, cid).ok()
    }
    
    // ...
}
```

[ChainSafe/filecoindot/filecoindot-io/src/lib.rs][interface]


## 1. configure our interfaces in node service

```rust
//! node/src/service.rs

impl sc_executor::NativeExecutionDispatch for ExecutorDispatch {
    // ...
    
    #[cfg(not(feature = "runtime-benchmarks"))]
    type ExtendHostFunctions = (filecoindot_io::forest_proof_verify::HostFunctions);
    
    // ...
}
```

[ChainSafe/filecoindot/substrate-node-example/node/src/service.rs#L30][service]


## 2. call our interface in pallet

```rust
//! filecoindot/src/lib.rs

impl<T: Config> Pallet<T> {
    pub fn verify_receipt_inner(
        proof: Vec<Vec<u8>>,
        block_cid: BlockCid,
        cid: Vec<u8>,
    ) -> DispatchResult {
        filecoindot_io::forest_proof_verify::verify_receipt(proof, cid).ok_or(Error::<T>::VerificationError)?;
        Self::verified_block(&block_cid)
            .then(|| ())
            .ok_or(Error::<T>::VerificationError)?;
        Ok(())
    }
    
    // ...
}
```

[ChainSafe/filecoindot/filecoindot/src/lib.rs#L388][pallet]



[europa]: https://github.com/patractlabs/europa
[filecoindot]: https://github.com/ChainSafe/filecoindot
[#89]: https://github.com/ChainSafe/filecoindot/pull/89
[interface]: https://github.com/ChainSafe/filecoindot/blob/main/filecoindot-io/src/lib.rs
[service]: https://github.com/ChainSafe/filecoindot/blob/9c0cdcf271047da0b222a7e487a39bf25d08907c/substrate-node-example/node/src/service.rs#L30
[pallet]: https://github.com/ChainSafe/filecoindot/blob/9c0cdcf271047da0b222a7e487a39bf25d08907c/filecoindot/src/lib.rs#L388
