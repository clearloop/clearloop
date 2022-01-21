# function generator

| Repository               | Pull Request |
|--------------------------|--------------|
| [clearloop/ceres][ceres] |              |

this is just a show case of the function generator in ceres.


```rust
/// Parse host function
pub fn parse(attr: TokenStream, item: TokenStream) -> TokenStream {
    let ts = item;
    let input: HostFunction = parse_macro_input!(ts);

    // construct host trait
    let module = attr.to_string();
    let name_str = input.name.to_string();

    // construct struct
    let struct_ident = input.struct_ident(&module);

    // construct function
    let content = input.content;
    let mut attr_t = TokenStream2::new();
    for attr in input.attrs {
        attr.to_tokens(&mut attr_t);
    }

    // declare args
    let arg_len = &input.fields.len();
    let mut args = TokenStream2::new();
    for i in 0..*arg_len {
        (&mut args).extend(input.fields[i].declare(i))
    }

    // quote output
    let tks = quote! {
        #attr_t
        pub struct #struct_ident;

        impl Host for #struct_ident {
            fn module() -> &'static str {
                #module
            }

            fn name() -> &'static str {
                #name_str
            }

            fn wrap(sandbox: &mut Sandbox, args: &[Value]) -> Result<Option<Value>> {
                log::debug!("call {}::{}", #module, #name_str);

                if args.len() != #arg_len {
                    return Err(Error::WrongArugmentLength);
                }

                #args

                #content
            }
        }
    };

    tks.into()
}

```

[ceres/crates/derive/src/attr/host.rs][host]


which will convert 

```rust
#[host(seal0)]
pub fn block_number(out_ptr: u32, out_len_ptr: u32) -> Result<Option<Value>> {
    sandbox.write_sandbox_output(out_ptr, out_len_ptr, &sandbox.block_number())?;
    Ok(None)
}
```

into 

```rust
pub struct Seal0BlockNumber;

impl Host for Seal0BlockNumber {
    fn module() -> &'static str {
        "seal0"
    }
    fn name() -> &'static str {
        "block_number"
    }

    fn wrap(sandbox: &mut Sandbox, args: &[Value]) -> Result<Option<Value>> {
        if args.len() != 2usize {
            return Err(Error::WrongArugmentLength);
        }
        let out_ptr: u32 = args[0usize].into();
        let out_len_ptr: u32 = args[1usize].into();
        {
            sandbox.write_sandbox_output(out_ptr, out_len_ptr, &sandbox.block_number())?;
            Ok(None)
        }
    }
}
```

[ceres]: https://github.com/clearloop/ceres
[host]: https://github.com/clearloop/ceres/blob/master/crates/derive/src/attr/host.rs
