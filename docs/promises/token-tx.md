---
sidebar_position: 2
---

# Sending $NEAR

You might want to send tokens from a contract for many reasons.

* The contract uses something like the [Storage Standard](https://nomicon.io/Standards/StorageManagement.html) and needs to return deposits to users when they unregister.
* Users pay into the contract and the contract later pays these fees to the maintainers, redistributes them to users, or disburses them to some cause the users vote on.
* And more!

Blockchains give us programmable money, and the ability for a smart contract to send tokens lies at the heart of that ability.

NEAR makes this easy. Transferring NEAR tokens is the simplest transaction you can send from a smart contract. Here's all you need:

```rust
let amount: u128 = 1_000_000_000_000_000_000_000_000; // 1 $NEAR as yoctoNEAR
let account_id: AccountId = "example.near".parse().unwrap();

Promise::new(account_id).transfer(amount);
```

:::caution Prerelease!
The `AccountId` behavior described throughout this document is a feature of `near-sdk-rs` **v4**. The functionality is [still possible](https://docs.rs/near-sdk/3.1.0/near_sdk/json_types/struct.ValidAccountId.html) using v3, but if you want to use the cleaner v4 syntax, use this in your `Cargo.toml`:

```toml
[dependencies]
near-sdk = "4.0.0-pre.2"
```
:::

In the context of a full contract and function call, this could look like:

```rust
use near_sdk::{json_types::U128, near_bindgen, AccountId, Promise};

#[near_bindgen]
pub struct Contract {}

#[near_bindgen]
impl Contract {
    pub fn pay(amount: U128, to: AccountId) -> Promise {
        Promise::new(to).transfer(amount.0)
    }
}
```

Most of this is boilerplate you're probably familiar with by now – imports, setting up [`near_bindgen`](../contract-structure/near-bindgen.md), [borsh](../contract-interface/serialization-interface.md), etc. Some interesting details related to the transfer itself:

* `U128` with a capital `U`: The `pay` method defined here accepts JSON as input, and JSON numbers [cannot be larger than 2^53](https://tools.ietf.org/id/draft-ietf-json-rfc4627bis-09.html#rfc.section.6), which is about 9 followed by a mere 15 zeroes. Since the `transfer` method takes a number in [yocto](https://en.wikipedia.org/wiki/Yocto-)NEAR, it's likely to need numbers much larger than 2^53.

  When a function takes `U128` as input, it means that callers need to specify the number a a string. near-sdk-rs will then cast it to `U128` type, which wraps Rust's native [`u128`](https://doc.rust-lang.org/std/primitive.u128.html). The underlying `u128` can be retrieved with `.0` – used in `transfer(amount.0)`.

* `AccountId`: this will automatically check that the provided string is a well-formed NEAR account ID, and panic with a useful error if not.

* Returning `Promise`: This allows NEAR Explorer, near-cli, near-api-js, and other tooling to correctly determine if a whole chain of transactions is successful. If your function does not return `Promise`, tools like near-cli will return immediately after your function call. And then even if the `transfer` fails, your function call will be considered successful. You can see a before & after example of this behavior [here](https://github.com/near-examples/rust-high-level-cross-contract/pull/73#issuecomment-902849410).

Using near-cli, someone could invoke this function with a call like:

    near call $CONTRACT pay '{"amount": "1000000000000000000000000", "to": "example.near"}'

