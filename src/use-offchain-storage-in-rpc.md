# use offchain storage in rpc

| Repository                           | Pull Request                    |
|--------------------------------------|---------------------------------|
| [Chainsafe/filecoindot][filecoindot] | [ChainSafe/filecoindot#60][#60] |

```rust
//! substrate-node-example/node/src/rpc.rs

// ...

pub struct FullDeps<C, P, S> {
    /// The offchain storage instance to use.
    pub storage: Option<Arc<RwLock<S>>>,
    /// The client instance to use.
    pub client: Arc<C>,
}

// ...


pub fn create_full<C, P, S>(deps: FullDeps<C, P, S>) -> jsonrpc_core::IoHandler<sc_rpc::Metadata>
where
    C: ProvideRuntimeApi<Block>,
    C: HeaderBackend<Block> + HeaderMetadata<Block, Error = BlockChainError> + 'static,
    C::Api: pallet_transaction_payment_rpc::TransactionPaymentRuntimeApi<Block, Balance>,
    C::Api: BlockBuilder<Block>,
    P: TransactionPool + 'static,
    // NOTE
    //
    // add `OffchainStorage` API
    S: OffchainStorage + 'static,
{
    use filecoindot_rpc::{Filecoindot, FilecoindotApi};
    use pallet_transaction_payment_rpc::{TransactionPayment, TransactionPaymentApi};
    use substrate_frame_rpc_system::{FullSystem, SystemApi};

    let mut io = jsonrpc_core::IoHandler::default();
    let FullDeps {
        client,
        deny_unsafe,
        pool,
        storage,
    } = deps;

    io.extend_with(SystemApi::to_delegate(FullSystem::new(

    // NOTE
    //
    // extend filecoindot rpc
    if let Some(storage) = storage {
        io.extend_with(FilecoindotApi::to_delegate(Filecoindot::new(storage)));
    }

    io
}
```

[ChainSafe/filecoindot#60][#60]


add offchain storage api and extend subtrate rpc.


```rust
//! substrate-node-example/node/src/service.rs

pub fn new_full(mut config: Configuration) -> Result<TaskManager, ServiceError> {

    // ...

    // NOTE
    //
    // this is quite tricky since we need to get the offchain storage from `backend`
    let storage = backend.offchain_storage().map(|s| Arc::new(RwLock::new(s)));
    
    let rpc_extensions_builder = {
        let client = client.clone();
        let pool = transaction_pool.clone();
        Box::new(move |deny_unsafe, _| {
            let deps = crate::rpc::FullDeps {
                client: client.clone(),
                pool: pool.clone(),
                deny_unsafe,
            };
            Ok(crate::rpc::create_full(deps))
        })
    };
    
    // ...
}

```

[ChainSafe/filecoindot#60][#60]

get offchain storage from backend.


[filecoindot]: https://github.com/ChainSafe/filecoindot
[#60]: https://github.com/ChainSafe/filecoindot/pull/60
