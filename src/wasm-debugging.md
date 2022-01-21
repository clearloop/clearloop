# wasm debugging

| Repository                                  | Pull Request                           |
|---------------------------------------------|----------------------------------------|
| [paritytech/cargo-contract][cargo-contract] | [paritytech/cargo-contract#131][#131]  |
| [paritytech/parity-wasm][parity-wasm]       | [paritytech/parity-wasm#300][#300]     |
| [paritytech/wasm-utils][wasm-utils]         | [paritytech/wasm-utils#146][#146]      |
| [patractlabs/wasmi][wasmi]                  | [patractlabs/wasmi#pulls][wasmi-pulls] |

This sample introduces what I did in [polkassembly#250][#250]

the wasm contract haven't have the ability to debug wasm traps at that time,
so what I did are distributed in `cargo-contract`, `wasmi`, `wasm-utils`, etc.

## 0. keep DWARF from rustc

```rust
//! src/cmd/build.rs

fn build_cargo_project(
    crate_metadata: &CrateMetadata,
    verbosity: Option<Verbosity>,
    unstable_flags: UnstableFlags,
    debug: bool,
) -> Result<()> {
    let mut flags =
        "-C link-arg=-z -C link-arg=stack-size=65536 -C link-arg=--import-memory".to_string();
    if debug {
        // NOTE:
        //
        // use opt-level=1 to keep more debug info, including DWARF of wasm
        // file to debug source with line number
        flags.push_str(" -C opt-level=1");
    }
    std::env::set_var("RUSTFLAGS", flags);
    
    // ...
}
```

[paritytech/cargo-contract#131][cargo-contract]


## 1. keep name section from striping

```rust
//! src/cmd/build.rs
fn strip_custom_sections(module: &mut Module, debug: bool) {
    module.sections_mut().retain(|section| match section {
        Section::Custom(_) => false,
        // NOTE
        //
        // https://webassembly.github.io/spec/core/appendix/custom.html#name-section
        // describe which function failed in our source code
        Section::Name(_) => debug,
        Section::Reloc(_) => false,
        _ => true,
    });
}
```

[paritytech/cargo-contract#131][cargo-contract]


## 2. keep debug info from `wasm-opt`

```rust
//! src/cmd/build.rs
fn optimize_wam(
    crate_metadata: &CrateMetadata,
    debug_info: bool,
) -> Result<(OptimizationResult, Option<PathBuf>)> {
    let codegen_config = binaryen::CodegenConfig {
        // execute -O3 optimization passes (spends potentially a lot of time optimizing)
        optimization_level: 3,
        // the default
        shrink_level: 1,
        // the default
        debug_info,
    };
    
    // ...
}
```

[paritytech/cargo-contract#131][cargo-contract]


## 3. store name section while building WASM module

this operation happens in `pallet-contracts`, which just drops
the name section again.

```rust
impl From<elements::Module> for ModuleScaffold {
	fn from(module: elements::Module) -> Self {
		
        // ...

		let mut other = Vec::new();
		let mut sections = module.into_sections();
		while let Some(section) = sections.pop() {
			match section {
			    element: element.unwrap_or_default(),
			    code: code.unwrap_or_default(),
			    data: data.unwrap_or_default(),
                // NOTE: name section belongs to custom section which is `other` here
			    other,
            }
        }
    }
}
```

[paritytech/parity-wasm#300][#300]


## 4. update function indices in name section while injecting gas meter

this also happens in `pallet-contracts`, they changes the function indecies
while injecting gas meter functions

```rust
pub fn inject_gas_counter<R: Rules>(
	module: elements::Module,
	rules: &R,
	gas_module_name: &str,
	process_names: ProcessNames,
)
	-> Result<elements::Module, elements::Module>
{
    
    // ...
    
    elements::Section::Name(section) => {
		match process_names {
			ProcessNames::No => {},
            ProcessNames::UnsafeYes => {
                if let Some(funcs) = section.functions_mut() {
                    let mut new_names = elements::IndexMap::<String>::default();
                    for (idx, func) in funcs.names().iter() {
                        if idx >= gas_func {
                            new_names.insert(idx + 1, String::from(func));
                        } else {
                            new_names.insert(idx, String::from(func));
                        }
                    }

                    *funcs.names_mut() = new_names;
                }
            }
        }
    }
}
```

[paritytech/wasm-utils#146][#146]


## 5. implement wasm backtrace to wasmi

this part is quite complex since `wasmi` doesn't have this design.


### 5.0. store function name in `FuncRef`

```rust
//! patractlabs/wasmi/src/func.rs#L22

// ...

pub struct FuncRef {
    
    // ...
    
    // Function name
    name: Option<String>,
}
```

[patractlabs/wasmi#1][#1]


### 5.1 set function info using name section while parsing WASM modules

```rust
//! patractlabs/wasmi/src/modules.rs#228

fn alloc_module<'i, I: Iterator<Item = &'i ExternVal>>(
    loaded_module: &Module,
    extern_vals: I,
) -> Result<ModuleRef, Error> {

    // ...

    // If has names section
    let mut has_name_section = true;
    let empty_index_map = IndexMap::with_capacity(0);
    let function_names = loaded_module.name_map().unwrap_or_else(|| {
        has_name_section = false;
        &empty_index_map
    });
    
    // ...
    
    for (index, (ty, body)) in Iterator::zip(
      funcs.into_iter(), bodies.into_iter()
    ).enumerate() {
    
        // ...
        
        if has_name_section {
            func_instance = func_instance.set_name(function_names.get(index as u32));
        }
        instance.push_func(func_instance);
    }
    
    // ...
}
```

[patractlabs/wasmi#1][#1]


### 5.2 make the interpreter support trace

```rust
//! patractlabs/wasmi/src/runner.rs#L1471

impl CallStack {
    /// Get the functions of current the stack
    pub fn trace(&self) -> Vec<Option<&String>> {
        self.buf.iter().map(|f| f.name()).collect::<Vec<_>>()
    }
}
```

[patractlabs/wasmi#1][#1]

### 5.3 gather debug infos when program breaks

```rust
//! patractlabs/wasmi/src/func.rs#163

pub fn invoke<E: Externals>(
    func: &FuncRef,
    args: &[RuntimeValue],
    externals: &mut E,
) -> Result<Option<RuntimeValue>, Trap> {
    
    // ...

    match *func.as_internal() {
        FuncInstanceInternal::Internal { .. } => {
            let mut interpreter = Interpreter::new(func, args, None)?;
            let res = interpreter.start_execution(externals);
            if res.is_err() {
                let mut stack = interpreter
                    .trace_stack()
                    .iter()
                    .filter_map(|n| n.as_ref())
                    .map(|n| rustc_demangle::demangle(n).to_string())
                    .collect::<Vec<_>>();
                stack.reverse();

                // Embed this info into the trap
                res.map_err(|e| e.set_wasm_trace(stack))
            } else {
                res
            }
        }
        
        // ...
        
    }
    
    // ...
}
```

[patractlabs/wasmi#1][#1]

[#250]: https://polkadot.polkassembly.io/post/250
[cargo-contract]: https://github.com/paritytech/cargo-contract
[#131]: https://github.com/paritytech/cargo-contract/pull/131
[wasmi]: a
[parity-wasm]: https://github.com/paritytech/parity-wasm
[#300]: https://github.com/paritytech/parity-wasm/pull/300
[wasm-utils]: https://github.com/paritytech/wasm-utils
[#146]: https://github.com/paritytech/wasm-utils/pull/146
[wasmi]: https://github.com/patractlabs/wasmi/tree/v0.9.1
[wasmi-pulls]: https://github.com/patractlabs/wasmi/pulls?q=is%3Apr+is%3Aclosed
[#1]: https://github.com/patractlabs/wasmi/pull/1/files
