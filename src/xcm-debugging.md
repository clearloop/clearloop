# XCM debugging

| Repository             | Pull Request               |
|------------------------|----------------------------|
| [ChainSafe/PINT][PINT] | [ChainSafe/PINT#474][#474] |


this part is not a sample in strict, just shows what I did in `#294` for debugging
XCM transactions.


## 0. use `xcm-emulator` for integration tests

no code here, but if you want to write xcm tests in rust, `xcm-emulator` 
and `xcm-simulator` are the best choices.

- `xcm-simulator` for unit tests
- `xcm-emulator` for integration tests (which can use kusama runtime for debugging directly)


## 1. enable `xcm` in `RUST_LOG`

```sh
RUST_LOG=xcm cargo t
```

this is really important for debugging xcm transactions since every xcm operations written by
parity guys related to xcm emits logs with `xcm` target.


## 2. pull cumulus and polkadot locally and replace our deps with patch

this happens a lot since the provided xcm logs are useless most of the time.


## 3. debugging parachain's transaction

```rust
//! polkadot/xcm/pallet-xcm/src/lib.rs

// ...

fn on_response(
	origin: &MultiLocation,
    query_id: QueryId,
    response: Response,
    max_weight: Weight,
) -> Weight {
    // ...
    
    match call.dispatch(dispatch_origin) {
    	Ok(post_info) => {
    		let e = Event::Notified(query_id, pallet_index, call_index);
    		Self::deposit_event(e);
    		post_info.actual_weight
    	},
    	Err(error_and_info) => {
    		let e = Event::NotifyDispatchError(
    			query_id,
    			pallet_index,
    			call_index,
    		);
    		Self::deposit_event(e);
    		// Not much to do with the result as it is. It's up to the parachain to ensure that the
    		// message makes sense.
    		error_and_info.post_info.actual_weight
    	},
    }
    .unwrap_or(weight)
    
    // ...
}

// ...
```

this is some thing really crazy for debugging xcm since polkadot just throw our errors with

```
// Not much to do with the result as it is. It's up to the parachain to ensure that the
// message makes sense.
```

**add logs here to debug parachain's transactions.**


## 4. XCM version

> NOTE: instructions like `BuyExecution` have different palaces in `xcm::v1` and `xcm::v2`, 
> they don't have the ability to convert to each other, they'll make your xcm transaction failed.

```rust
//! PINT/runtime/integration-tests/src/ext.rs

pub fn kusama_ext() -> sp_io::TestExternalities {
  GenesisBuild::<Runtime>::assimilate_storage(
    &pallet_xcm::GenesisConfig { safe_xcm_version: Some(1) }, 
    &mut t
  ).unwrap();
}
```

[PINT/runtime/integration-tests/src/ext.rs#L67][xcm-version]

the default config of kusama configures the `safe_xcm_version` with `None`, which 
will navigate to `latest(2)` in their logic, you `xcm::v1` transaction will not pass 
the `XcmSender` and you'll never know what happens...

[polkadot/runtime/common/src/xcm_sender.rs#L27][send-xcm]

if you're using the XcmRouter from cumulus, debug it also.


## 5. balance decimals

> `1_000_000_000_000 (Balance)` means `1` KSM in kusama's code, this is well known for polkadotjs
> developers but really easy to ignore in the rust side, and you may take mb at least 2 days to 
> debug this issue if you're writing an integration tests between kusama and you DeFi parachain.

```rust
pub const INITIAL_BALANCE: Balance = 10_000_000_000_000;
```

[ChainSafe/PINT/runtime/integration-tests/src/prelude.rs#L19][balance]

[pallet-xcm]: https://github.com/paritytech/polkadot/blob/master/xcm/pallet-xcm/src/lib.rs#L1432
[PINT]: https://github.com/ChainSafe/PINT
[#474]: https://github.com/ChainSafe/PINT/pull/474
[xcm-version]: https://github.com/ChainSafe/PINT/blob/acb2900a8bf45044aa15c0dd22d2d947f168bf2f/runtime/integration-tests/src/ext.rs#L67
[send-xcm]: https://github.com/paritytech/polkadot/blob/cc24fb87b702db6510d08c689e442b6be384d798/runtime/common/src/xcm_sender.rs#L27
[balance]: https://github.com/ChainSafe/PINT/blob/acb2900a8bf45044aa15c0dd22d2d947f168bf2f/runtime/integration-tests/src/prelude.rs#L19
