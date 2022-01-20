# transactional

| Repository             | Pull Request               | Sample                                      |
|------------------------|----------------------------|---------------------------------------------|
| [ChainSafe/PINT][PINT] | [ChainSafe/PINT#294][#294] | [pallets/asset-index/src/lib.rs#L700][code] |

This sample basicly shows how to handle storage mutation in failing transactions.

```rust
// use `#[transaction]` to prevent commiting changes into storage in a failing transaction
//
// https://docs.substrate.io/rustdocs/latest/frame_support/attr.transactional.html
#[transactional]
pub fn complete_withdraw(origin: OriginFor<T>) -> DispatchResult {
	let caller = T::AdminOrigin::ensure_origin(origin.clone())?;
	let current_block = frame_system::Pallet::<T>::block_number();

    // use `try_mutate_exists` to prevent mutating storage in a failing transaction
    //
    // https://docs.substrate.io/rustdocs/latest/frame_support/storage/trait.StorageMap.html#tymethod.try_mutate_exists
	PendingWithdrawals::<T>::try_mutate_exists(&caller, |maybe_pending| -> DispatchResult {
		let pending = maybe_pending.take().ok_or(<Error<T>>::NoPendingWithdrawals)?;
		let still_pending: Vec<_> = pending
			.into_iter()
			.filter_map(|mut redemption| {
				if redemption.end_block >= current_block &&
					Self::do_complete_redemption(&caller, &mut redemption.assets)
				{
					Self::deposit_event(Event::WithdrawalCompleted(caller.clone(), redemption.assets));
					return None;
				}
				Some(redemption)
			})
			.collect();

		if !still_pending.is_empty() {
			*maybe_pending = Some(still_pending);
		}

		Ok(())
	})
}
```

there were some confusion about the design in `#294`, quite pain, but finally learned sth from that.


[PINT]: https://github.com/ChainSafe/PINT
[#294]: https://github.com/ChainSafe/PINT/pull/294
[code]: https://github.com/ChainSafe/PINT/blob/acb2900a8bf45044aa15c0dd22d2d947f168bf2f/pallets/asset-index/src/lib.rs#L700
