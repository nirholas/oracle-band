# OracleBand

**Refuses to let a pool settle at a price the wider market does not recognise.**

A production Uniswap v4 hook. It holds no funds and takes no fee for itself. No owner, no pause switch, no upgrade path.

- **Site:** https://oracle-band.pages.dev
- **Catalogue:** https://hookforge.pages.dev
- **Contract:** [`src/hooks/OracleBandHook.sol`](src/hooks/OracleBandHook.sol)
- **Licence:** MIT

## How it works

A thin pool can be walked anywhere. Buy through the last tick of liquidity and the pool will quote you a price no other venue would, and whatever reads that pool afterwards, a lending market, a vault, another pool's oracle, inherits the number. The pool is not wrong: it faithfully reports what it was paid.

It is simply alone. This hook gives the pool a second opinion. Before and after every swap it compares the pool price against a reference from an {IPriceOracle} and rejects the swap if the result lands outside a band around it: deviation = |poolPrice / referencePrice - 1|, rejected when deviation > maxDeviationBps The check runs on both sides of the swap, and the pair is what makes it hard to defeat.

The `beforeSwap` check refuses to trade from a price that is already outside the band, so a pool pushed out of line in one transaction cannot be traded against in the next. The `afterSwap` check refuses to leave the pool outside the band, which is what stops the walk in the first place. Neither check can be satisfied by splitting a large swap into small ones, because the constraint is on the resulting price rather than on the size of the trade.

Prices are compared in `sqrtPriceX96`, squared back through `FullMath` so the comparison is on the actual price ratio and not on an approximation of it that drifts as the deviation grows. Liquidity operations are untouched. A provider can always withdraw, including while the band is refusing swaps, which is the property that makes it safe to sit behind one.

The obvious objection: this makes the pool depend on an oracle, and oracles fail. So the hook treats failure as refusal rather than as permission. A feed older than `maxStaleness` halts swapping instead of waving it through, and a feed that reverts propagates rather than being caught.

A pool that would rather trade blind than not trade should not use this hook, and the fee-based hooks in this catalogue are the oracle-free alternative.

## Prior art

Oracle-deviation checks exist inside individual protocols, and Detox uses Pyth to detect MEV and redirect it. Both act after the fact, on a pool that has already printed the price. Enforcing the band as a precondition on both sides of the swap, so the out-of-band price is never written at all, is what is new here.

## Where it does not help

The pool inherits the oracle's liveness. If the feed stops, swapping stops, and on a chain where the feed updates on a deviation threshold rather than a heartbeat, a quiet market can look stale. Set maxStaleness against the feed's actual publication cadence, not against how fresh you would like it to be.

## Using it

Uniswap v4 removed `hookData` from `initialize`, so per-pool parameters arrive out of band. Fix them for a pool key whose pool does not exist yet, then initialize. Nobody can change them afterwards, including you.

```solidity
hook.configure(
    key,
    OracleBandHook.Config({
        oracle: /* address */ 0,
        maxDeviationBps: /* uint32 */ 0,
        maxStaleness: /* uint32 */ 0
    })
);

poolManager.initialize(key, startingSqrtPriceX96);
```


### Parameters

| Parameter | Type | Units |
| --- | --- | --- |
| `oracle` | `address` |  |
| `maxDeviationBps` | `uint32` | basis points (`10000` = 100%) |
| `maxStaleness` | `uint32` |  |

## What it reverts with

| Error | Meaning |
| --- | --- |
| `InvalidConfig()` | `maxDeviationBps` or `maxStaleness` was zero, or no oracle was given. |
| `OutsideBand(uint256,uint32)` | The pool price is outside the band around the reference. |
| `PoolAlreadyInitialized()` | The pool already exists, so its configuration is final. |
| `PoolNotConfigured()` | The pool was initialized without a configuration for this hook. |
| `ReferenceStale(uint256,uint32)` | The reference price is older than the pool tolerates, so the pool declines to trade. |

## The callbacks it claims

Uniswap v4 reads a hook's permissions from the low fourteen bits of its own address, which is why deploying one means mining a CREATE2 salt. This hook claims 3 of the fourteen:

- `afterInitialize`
- `beforeSwap`
- `afterSwap`

Mask: `0x10c0`, so every deployment of this hook has an address ending in those bits.

## It says what it is, on-chain

Every hook in this family implements `IHookMetadata`: four view functions that let an indexer, a wallet, a router or an agent identify a hook from its address alone, with no registry in the loop.

```bash
cast call $HOOK "hookName()(string)"    # OracleBand
cast call $HOOK "hookVersion()(string)" # 1.0.0
cast call $HOOK "specURI()(string)"     # the machine-readable manifest
cast call $HOOK "hookTags()(string[])"  # risk, oracle, manipulation-resistance, no-admin
```

The manifest this repository ships as [`hook.json`](hook.json) is what `specURI()` points at.

## Build and test

```bash
git clone --recurse-submodules https://github.com/nirholas/oracle-band
cd oracle-band
forge build
forge test
```

Foundry 1.7 or newer, Solidity 0.8.26, EVM version `cancun` (Uniswap v4 requires transient storage).

## Deploy

```bash
# Dry run: mines the salt and prints the address without sending anything.
forge script script/Deploy.s.sol --rpc-url $RPC_URL

# For real.
forge script script/Deploy.s.sol --rpc-url $RPC_URL --broadcast --verify
```

Needs `PRIVATE_KEY` in the environment and a funded deployer on the target chain. See [`docs/deploying.md`](docs/deploying.md).

## Status

**Unaudited.** Built to an audited shape, on OpenZeppelin's audited hook bases, and tested against a real `PoolManager`. No third party has reviewed it. Read "where it does not help" above before putting money behind it.

Not affiliated with Uniswap Labs.
