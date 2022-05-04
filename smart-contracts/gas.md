---
description: What you need to know about gas as a Juno smart contract developer
---

# Gas and You

What you need to know about gas as a Juno smart contract developer.

## What is Gas

Gas is an abstract unit representing the computational cost of a transaction. 

It is used to ensure that all of the transactions within a block can finalize among the validator set within a reasonable amount of time, prevent abuse / spam, and create an incentive structure for validator operators (through gas fees).

`Gas fees = gas * gas-prices`

Each transaction has two gas related values:
* `GasWanted`: Amount of gas requested for transaction. It is provided by users when they generate the transaction
* `GasUsed`: Amount of gas actually consumed by transaction

</br>

## How Gas is Calculated

### Tendermint

Gas calculation is not done within tendermint.

</br>

### Cosmos SDK

Runtime gas calculation is done through the use of [GasMeters](https://docs.cosmos.network/master/basics/gas-fees.html#gas-meter). 

`ctx.GasMeter()` is used throughout the Cosmos SDK to track the current transaction's gas usage. 

Gas is consumed in a variety of places in the SDK but mostly happens when: 
* CRUDing [from the KVstore](https://github.com/cosmos/cosmos-sdk/blob/v0.42.10/store/types/gas.go#L198-L209)
* Charging for the total transaction size
* Pubkey signature verifications

This gas meter is also used by the Cosmos modules, accessed through the Context still, to charge gas for whatever the module devs think is relevant.

</br>

### Cosmwasm Module

The [cosmwasm module](https://github.com/CosmWasm/wasmd/tree/main/x/wasm#wasm-module) charges gas in [many different situations](https://github.com/CosmWasm/wasmd/blob/d5ef3ba2de3c9c87adb2d8826da35f4c5b07bc3c/x/wasm/keeper/gas_register.go#L13-L54). 

The cosmwasm go module also calls out to the [Cosmwasm wasmer VM](https://github.com/CosmWasm/cosmwasm/tree/v1.0.0-beta/packages/vm), which also charges for a [constant gas amount](https://github.com/CosmWasm/cosmwasm/blob/v1.0.0-beta/docs/GAS.md) per wasm instruction. The wasmer VM also charges for calls out to cosmos SDK and calls out to host functions.

I originally thought that this cost per [wasm operation](https://webassembly.github.io/spec/core/syntax/instructions.html) was going to be the largest part of the cost of the smart contract transactions, but from my limited digging into some dao-dao contracts it appears to be a pretty insignifcant percent of the overall gas costs. This makes me think that either the contracts in question were really computationally light weight, or this per wasm operation gas cost was mainly there to prevent people from really abusing things (like writing infinte loops or doing really CPU intensive work).

</br>

### Juno Specific Configuration
* Cosmwasm module [gas config overrides](https://github.com/CosmosContracts/juno/blob/main/app/wasm_config.go#L8-L11)

</br>

## How Gas is Used

### Tendermint

#### Max Gas Checks

Gas is mostly irrelevant to the tendermint layer, and [completely ignored during consensus](https://docs.tendermint.com/master/spec/abci/apps.html#gas):
```
Note that Tendermint does not currently enforce anything about Gas in the consensus, only the mempool. This means it does not guarantee that committed blocks satisfy these rules! It is the application's responsibility to return non-zero response codes when gas limits are exceeded.
```

However there is a tendermint [Consensus Param](https://github.com/tendermint/tendermint/blob/master/docs/architecture/adr-005-consensus-params.md) called [ConsensusParams.Block.MaxGas](https://docs.tendermint.com/master/spec/abci/apps.html#blockparams-maxgas) which when set does enforce that:
* `GasWanted` <= `ConsensusParams.Block.MaxGas` for [all txs in the mempool](https://github.com/tendermint/tendermint/blob/37287ead94aa010f2497a5df414c64b85e4861ce/node/setup.go#L160)
* (sum of `GasWanted` in a block) <= `ConsensusParams.Block.MaxGas` [when proposing a block](https://github.com/tendermint/tendermint/blob/37287ead94aa010f2497a5df414c64b85e4861ce/internal/mempool/mempool.go#L363-L367)

Tendermint only cares about `GasWanted`, Tendermint doesn't even use `GasUsed`. Which means limiting `GasUsed` <= `GasWanted` and `(sum of GasUsed in a block) <= MaxGas` is fully the responsibility of the Cosmos SDK Application.

#### Gas Fee based prioritization

Currently, CosmosSDK builds a block of transactions from the mempool in a FIFO ordering, but tendermint added support to allow for [prioritizing the transactions in the mempool](https://github.com/tendermint/tendermint/blob/f9e0f77af333f4ab7bfa1c0c303f7db47cec0c9e/docs/architecture/adr-067-mempool-refactor.md). So Cosmos SDK could allow for fee based priorization in the future, but for now I think there's no plan to move forward with fee based prioritization like in Etherium.

</br>

### Cosmos SDK

#### Max Gas Checks
Cosmos SDK uses [two GasMeters](https://docs.cosmos.network/master/basics/gas-fees.html#gas-meter): `ctx.GasMeter()` and `ctx.BlockGasMeter()` that track the actual, runtime gas usage. The former ensures that each transaction doesn't use more gas than its `GasWanted`, and the later ensures that the entire block's gas usage doesnt exceed the  `ConsensusParams.Block.MaxGas` we saw earlier.

#### Min Gas Checks
There is a Cosmos SDK local node config called `minimum-gas-prices`, that filters out transactions that have gas fees lower than the configured amount when creating a block to propose, and adding transactions to the mempool (TODO: Find the code for this to dig further).

It is worth noting that `minimum-gas-prices` is a config that each validator can set locally, and it is not agreed upon through governance. This means that validators [could actually include a bunch of 0 fee transactions when they are the proposer](https://docs.cosmos.network/master/core/baseapp.html#delivertx). There is [some discussion](https://github.com/cosmos/cosmos-sdk/discussions/8224) to try and make a min gas price that must be set a Consensus Param controlled through governance like `ConsensusParams.Block.MaxGas` (still allowing validators to set an amount higher than this absolute minimum). We need to investigate how worrisome this 0 fee spam could actually be when a rogue validator would only be up for proposal 150th of the time.

</br>

### Cosmwasm Module

#### Max Gas Checks

The cosmwasm module has a local node configuration called `query_gas_limit` which allows you to set a maximum gas limit for all [smart contract external queries](https://docs.cosmwasm.com/docs/1.0/architecture/query). Because those queries hit a single node, out of band of the block / transaction processing, they do not actually have an actual gas fee. So this limit is set to avoid DOSing RPC nodes with query calls.

</br>

### Juno Specific Configuration
* Tendermint's `ConsensusParams.Block.MaxGas` is [currently set to `100000000` for Juno](https://www.mintscan.io/juno/proposals/6)
* Juno's `AnteHandler` implementation is the default cosmosSDK auth module

</br>

## Security Concerns to Investigate
* Gas calculation is very complicated, there are surely loopholes in the gas metering / cost configuration that can be exploited:
  * Ex: at a very quick glance, [things that are currently configured for 0 gas costs looked sus](https://github.com/CosmWasm/wasmd/blob/d5ef3ba2de3c9c87adb2d8826da35f4c5b07bc3c/x/wasm/keeper/gas_register.go#L43-L48)
* As mentioned earlier, how much damage can a validator do by not setting `minimum-gas-prices` and flooding their block proposals with 0 fee transactions?
* Using more gas than you paid for:
  * Ex: https://github.com/CosmWasm/cosmwasm/issues/456#issuecomment-660304817
* Possible chain halts by exhausting all of the gas for a block using loopholes?
 * Is it possible to write a smart contract that just blocks / sleeps? Is that somehow blocked by the wasmer runtime? If so how does that happen.

</br>

## Future Work

* Fine Grained Gas Instrumentation for gas costs breakdown
* CI Gas benchmarking for the smart contracts
* Better ways to benchmark gas config constants and determine when they drift out of usefulness for each chain
