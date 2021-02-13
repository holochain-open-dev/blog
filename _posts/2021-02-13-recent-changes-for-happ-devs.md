---
layout: post
title: Recent Changes for hApp Devs
tags: [holochain, rsm, holochain-rsm]
author: Guillem Córdoba
---

Here are some recent changes that will improve hApp development. Even though they may create some friction at first if you have to change already existing code, they are all steps in the right direction in the long run.

# nix-shell



# HDK

In the recent weeks, there have been a lot of breaking changes happening in the HDK. This is mainly an effort on the part of the holochain development team to introduce as many features and change possible before the release cycle for the HDK and holochain itself is stablished, and with it we arrive to a stabilization of the HDK API.

Here, we'll be detailing the main breaking changes that will affect you if you have some rsm zome code that needs updating to the latest version.

## Serialization

- **The macro `SerializedBytes` is no longer necessary with input/output structs, but now `Debug` is**

You'll need to add the `Debug` macro in all input/output structs, and the `SerializedBytes` is optional.

- **Any rust types that implement `Debug` can be returned**

You can now freely use `Vec`s, `Option`s, `String`s as input or output types for your functions.

- **Better output on serialization errors**

We now get useful messages whenever a serialization error occurs instead of the usual `Unknown error`.

### Example

Before the serialization changes:

```rust
#[derive(Clone, Serialize, Deserialize, SerializedBytes)]
pub struct SearchProfilesInput {
    nickname_prefix: String,
}
#[derive(Clone, Serialize, Deserialize, SerializedBytes)]
pub struct GetProfilesOutput(Vec<AgentProfile>);
#[hdk_extern]
pub fn search_profiles(
    search_profiles_input: SearchProfilesInput,
) -> ExternResult<GetProfilesOutput> {
    let agent_profiles = profile::search_profiles(search_profiles_input.nickname_prefix)?;

    Ok(GetProfilesOutput(agent_profiles))
}
```

After the serialization changes:

```rust
#[derive(Clone, Serialize, Deserialize, Debug)]
pub struct SearchProfilesInput {
    nickname_prefix: String,
}
#[hdk_extern]
pub fn search_profiles(
    search_profiles_input: SearchProfilesInput,
) -> ExternResult<Vec<AgentProfile>> {
    let agent_profiles = profile::search_profiles(search_profiles_input.nickname_prefix)?;

    Ok(agent_profiles)
}
```

## `ExternResult` has now `WasmError` as its result: one less nested struct

Before, if I wanted to return a string error, I had to return all these nested structs:

```rust
return Err(HdkError::Wasm(WasmError::Zome("Error Message".into())));
```

Now, `HdkError` no longer exists: now you can simply return a `WasmError::Guest`:

```rust
return Err(WasmError::Guest("Error Message".into()));
```

## `entry_def_index!` makes `query` easier

If you've written a `query` function call in RSM, you know it can get tedious, because of the entry definition index. Now with this new macro, we can just write `query` like this:

```rust
fn query_all_offers() -> ExternResult<Vec<Element>> {
    let offer_entry_type = EntryType::App(AppEntryType::new(
        entry_def_index!(Offer),
        zome_info()?.zome_id,
        EntryVisibility::Private,
    ));
    let filter = ChainQueryFilter::new()
        .entry_type(offer_entry_type)
        .include_entries(true);
    let query_result = query(filter)?;

    Ok(query_result.0)
}
```

## `EntryDefRegistration` trait

There is now the `EntryDefRegistration` trait that is automatically generated whenever we use `#[hdk_entry()]` to define an entry. This means that we can write a generic function that takes trait objects for that trait, and call the necessary methods on that:

```rust
#[hdk_entry(id = "profile")]
struct Post(String);

fn query_and_convert_entries<T: EntryDefRegistration>() -> ExternResult<Vec<T>> {
    let entry_def_id = T::entry_def_id();
  
    let filter = ChainQueryFilter::new()
        .entry_type(EntryType::App(AppEntryType::new(
            entry_def_id,
            zome_info!()?.zome_id,
            EntryVisibility::Private,
        )))
        .header_type(HeaderType::Create)
        .include_entries(true);
    let query_result: ElementVec = query!(filter)?;
    Ok(query_result.0)
}
```

## Better support for timestamps

Timestamps have been improved by a big margin: they are now easily convertible to and from `chronos` types, and also you can do easy math on them.

To see all changes and new goodies, check [here](https://github.com/holochain/holochain/pull/617/files#diff-0c3ccc5c3c05b5346a3ddd204bbf5251e48dd9700152c47ee70548f65879aec3L11).

## `sign_raw` and `verify_signature_raw`

The recently introduced `sign` and `verify_signature` have been joined by `sign_raw` and `verify_signature_raw`:

- Raw versions directly accept an array of bytes as its input
- "Normal" versions serializes the input struct before signing the bytes
 
# Debugging

There is a bunch of changes in the way that tracing is done at the conductor/wasm interface level. The biggest win we have as holochain devs is the ability to add the `WASM_LOG` to our tests and runs to control the output that we get. This allows us to turn off the logs that holochain itself outputs, and just concentrate on the logs that our application is producing.

The `WASM_LOG` variable can take the same values as the `RUST_LOG` one: `error`, `info`, `debug` and `trace`. The two variables can also be combined together to control the output of holochain.

Example use when running holochain:

```
WASM_LOG=debug holochain
```

Example use with the `package.json` of tryorama:

```json
  "scripts": {
    "test": "WASM_LOG=debug RUST_LOG=error RUST_BACKTRACE=1 TRYORAMA_HOLOCHAIN_PATH=\"holochain\" ts-node src/index.ts"
  },
```

With this, we now also get different levels of debugging output enabled in our zomes: 

```rust
#[hdk_extern]
fn debug(_: ()) -> ExternResult<()> {
    trace!("tracing {}", "works!");
    debug!("debug works");
    info!("info works");
    warn!("warn works");
    error!("error works");
    debug!(foo = "fields", bar = "work", "too");

    Ok(())
}
```