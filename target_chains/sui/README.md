# Pyth on Sui

### Contents

1. Background
2. How to Update and Consume Price Feeds
3. Examples
4. Contracts Registry (addresses)
5. Common Questions how How to Integrate with Pyth on Sui

# 1. Background

Pyth price feeds on Sui are uniquely represented in the global store as `PriceInfoObjects`. These objects have the `key` ability and serve as wrappers around the `PriceInfo` object, which in turn contains the price info: namely the `PriceFeed`, the arrival time of the latest price update, and the attestation time of the latest update.

`PriceInfoObject`s are central to Pyth on Sui, since they are in unique correspondence with each Pyth price feed and must be passed in to functions that update price feeds or which query info about price feeds, e.g.

- `update_single_price_feed`
- `update_single_price_feeds_if_fresh`
- `get_price`

# 2. How to Update and Consume Price Feeds

We demo how to update and then consume a price feed by building a Sui [programmable transaction](https://docs.sui.io/build/prog-trans-ts-sdk) off-chain, and then executing it to update a price feed and get an updated price.

As with other chains, one first obtains a batch price attestation VAA (of type `vector<u8>`) from a Pyth price service endpoint, which encodes update price information for a feed.

### 1. `wormhole::vaa::parse_and_verify`

Call `parse_and_verify` on the batch attestation VAA bytes to obtain a `VAA` hot potato object.

```Rust
public fun parse_and_verify(
      wormhole_state: &State,
      buf: vector<u8>, // price update VAA bytes
      the_clock: &Clock
  ): VAA
```

### 2. `pyth::pyth::update_single_price_feed`

Use the verified VAA and create a hot potato vector containing the latest price updates.

```Rust
public fun create_price_infos_hot_potato(
        pyth_state: &PythState,
        verified_vaas: vector<VAA>,
        clock: &Clock
): HotPotatoVector<PriceInfo>
```

### 3.`pyth::pyth::update_single_price_feed`

Use the hot potato price updates vector to update a price feed.

Note that conventional Pyth price IDs are found [here](https://pyth.network/developers/price-feed-ids#pyth-evm-mainnet).
However, instead of passing in a Pyth price feed ID to update the price feed (which is what is done on other chains), one must pass in a `PriceInfoObject` ID instead.

The `PriceInfoObject` IDs are distinct from Pyth price feed IDs, and are stored in a map on-chain (Pyth price feed ID => PriceInfoObject ID). We pulled this map into a local json file [here](./scripts/generated). The `PriceInfoObject` ID can also be queried on-chain by calling the `pyth::state::get_price_info_object_id` found in the Pyth package. See the common questions section below for more info.

```Rust
public fun update_single_price_feed(
    pyth_state: &PythState,
    price_updates: HotPotatoVector<PriceInfo>,
    price_info_object: &mut PriceInfoObject,
    fee: Coin<SUI>,
    clock: &Clock
): HotPotatoVector<PriceInfo>
```

### 4.`pyth::hot_potato_vector::destroy`

Drop the hot potato. (You must call this function to drop the potato).

```Rust
public fun destroy<T: copy + drop>(
    hot_potato_vector: HotPotatoVector<T>
)
```

### 5. `pyth::pyth::get_price`

Finally, get the price of the updated price feed in `PriceInfoObject` 🎉🎉🎉.

```Rust
public fun get_price(
      state: &PythState,
      price_info_object: &PriceInfoObject,
      clock: &Clock
): Price
```

# 3. Examples

See the `./scripts` folder for examples of programmable transactions for creating price feeds and updating price feeds.

- [Demo for updating a price feed](./scripts/pyth/update_price_feeds.ts)

To build and test the contracts, run the following

```
$ make test
$ make build
```

# 4. Contracts Registry

## Pyth on Testnet

- PYTH_PACKAGE_ID: [0x975e063f398f720af4f33ec06a927f14ea76ca24f7f8dd544aa62ab9d5d15f44](https://explorer.sui.io/object/0x975e063f398f720af4f33ec06a927f14ea76ca24f7f8dd544aa62ab9d5d15f44?network=testnet)
- PYTH_STATE_ID: [0xd8afde3a48b4ff7212bd6829a150f43f59043221200d63504d981f62bff2e27a](https://explorer.sui.io/object/0xd8afde3a48b4ff7212bd6829a150f43f59043221200d63504d981f62bff2e27a?network=testnet)
- PYTH_PRICE_FEED_ID_TO_PRICE_INFO_OBJECT_ID: [0xf8929174008c662266a1adde78e1e8e33016eb7ad37d379481e860b911e40ed5](https://explorer.sui.io/object/0xf8929174008c662266a1adde78e1e8e33016eb7ad37d379481e860b911e40ed5?network=https%3A%2F%2Ffullnode.testnet.sui.io%3A443)

## Wormhole on Testnet

- WORMHOLE_PACKAGE_ID: [0xcc029e2810f17f9f43f52262f40026a71fbdca40ed3803ad2884994361910b7e](https://explorer.sui.io/object/0xcc029e2810f17f9f43f52262f40026a71fbdca40ed3803ad2884994361910b7e?network=testnet)
- WORMHOLE_STATE_ID: [0xebba4cc4d614f7a7cdbe883acc76d1cc767922bc96778e7b68be0d15fce27c02](https://explorer.sui.io/object/0xebba4cc4d614f7a7cdbe883acc76d1cc767922bc96778e7b68be0d15fce27c02?network=testnet)

## Pyth on Mainnet

- PYTH_PACKAGE_ID: [0x00b53b0f4174108627fbee72e2498b58d6a2714cded53fac537034c220d26302](https://explorer.sui.io/object/0x00b53b0f4174108627fbee72e2498b58d6a2714cded53fac537034c220d26302?network=https%3A%2F%2Ffullnode.mainnet.sui.io%3A443)
- PYTH_STATE_ID: [0xf9ff3ef935ef6cdfb659a203bf2754cebeb63346e29114a535ea6f41315e5a3f](https://explorer.sui.io/object/0xf9ff3ef935ef6cdfb659a203bf2754cebeb63346e29114a535ea6f41315e5a3f?network=https%3A%2F%2Ffullnode.mainnet.sui.io%3A443)
- PYTH_PRICE_FEED_ID_TO_PRICE_INFO_OBJECT_ID: [0x14b4697477d24c30c8eecc31dd1bd49a3115a9fe0db6bd4fd570cf14640b79a0](https://explorer.sui.io/object/0x14b4697477d24c30c8eecc31dd1bd49a3115a9fe0db6bd4fd570cf14640b79a0?network=https%3A%2F%2Ffullnode.mainnet.sui.io%3A443)

## Wormhole on Mainnet

- WORMHOLE_PACKAGE_ID: [0x5306f64e312b581766351c07af79c72fcb1cd25147157fdc2f8ad76de9a3fb6a](https://explorer.sui.io/object/0x5306f64e312b581766351c07af79c72fcb1cd25147157fdc2f8ad76de9a3fb6a)
- WORMHOLE_STATE_ID: [0xaeab97f96cf9877fee2883315d459552b2b921edc16d7ceac6eab944dd88919c](https://explorer.sui.io/object/0xaeab97f96cf9877fee2883315d459552b2b921edc16d7ceac6eab944dd88919c)

# 5. Common Questions on How to Integrate with Pyth on Sui

## 1. What is up with the "sui rev"? (`09b2081498366df936abae26eea4b2d5cafb2788`). Why does it point to a specific commit hash instead of "main" or "devnet"?

Our Pyth `Move.toml` file contains the following dependencies. It depends on specific versions of the [Sui Framework](https://github.com/MystenLabs/sui) as well as [Wormhole](https://github.com/wormhole-foundation/wormhole). To make your Sui package compatible, you must also specify the following dependencies verbatim in your `Move.toml` file. We are locked in to this specific `rev` because our package depends on Wormhole, which [uses the `rev` `09b2081498366df936abae26eea4b2d5cafb2788`](https://github.com/wormhole-foundation/wormhole/blob/main/sui/wormhole/Move.mainnet.toml).

```
[dependencies.Sui]
git = "https://github.com/MystenLabs/sui.git"
subdir = "crates/sui-framework/packages/sui-framework"
rev = "09b2081498366df936abae26eea4b2d5cafb2788"

[dependencies.Wormhole]
git = "https://github.com/wormhole-foundation/wormhole.git"
subdir = "sui/wormhole"
rev = "d050ad1d67a5b7da9fb65030aad12ef5d774ccad"
```

## 2. How do I find the Sui Object ID of a PriceInfoObject for a Pyth Price Feed?

The mapping of Pyth price feed IDs to `PriceInfoObject` object IDs can be found [here](./scripts/generated/). (Note: this may go out of date as more price feeds are introduced and created over time).

This mapping is also stored on-chain, and can be queried on-chain using the getter function `pyth::state::get_price_info_object_id` defined in the Pyth package.

Also recall that the list of Pyth price feed IDs can be found [here](https://pyth.network/developers/price-feed-ids#pyth-evm-testnet).
