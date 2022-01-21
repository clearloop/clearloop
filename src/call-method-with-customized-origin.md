# call method with customized origin

| Repository             | Pull Request               |
|------------------------|----------------------------|
| [ChainSafe/PINT][PINT] | [ChainSafe/PINT#104][#104] |


Here is an origin for committee member, which uses a customized origin,
for using this origin in benchmarks, we need to use `dispatch_bypass_filter`
to recognize our origin.


```rust
//! pallets/committee/src/types.rs#219

// ...

/// Ensure committee member
pub struct EnsureMember<T>(sp_std::marker::PhantomData<T>);

impl<
        O: Into<Result<RawOrigin<<T as frame_system::Config>::AccountId>, O>>
            + From<RawOrigin<<T as frame_system::Config>::AccountId>>
            + Clone,
        T: Config,
    > EnsureOrigin<O> for EnsureMember<T>
{
    type Success = <T as frame_system::Config>::AccountId;
    fn try_origin(o: O) -> Result<Self::Success, O> {
        let origin = o.clone().into()?;
        match origin {
            RawOrigin::Signed(i) => {
                if <Members<T>>::contains_key(i.clone()) {
                    Ok(i)
                } else {
                    Err(o)
                }
            }
            _ => Err(o),
        }
    }

    #[cfg(feature = "runtime-benchmarks")]
    fn successful_origin() -> O {
        O::from(RawOrigin::Signed(Default::default()))
    }
}

// ...
```

```rust
//! pallets/committee/src/mock.rs

// ...

impl pallet_committee::Config for Test {
    
    // ...
    
    type ProposalExecutionOrigin = EnsureMember<Self>;
    
    // ...
    
}

// ...

```

```rust
//! pallets/committee/src/benchmarking.rs#36

benchmarks! {
    propose {
        let origin = T::ProposalSubmissionOrigin::successful_origin();
        let proposal = submit_proposal::<T>(origin.clone());
        let call = <Call<T>>::propose(Box::new(<SystemCall<T>>::remark(vec![0; 0]).into()));
    }: {
        call.dispatch_bypass_filter(origin)?
    } verify {
        assert!(<Pallet<T>>::get_proposal(&proposal.hash()) == Some(proposal));
    }
}

```

[ChainSafe/PINT#104][#104]


[PINT]: https://github.com/ChainSafe/PINT
[#104]: https://github.com/ChainSafe/PINT/pull/104
