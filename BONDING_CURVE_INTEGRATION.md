# NadFun V2 Bonding Curve Integration

This document is for indexers, analytics, backend services, and advanced integrations that need to understand NadFun's pre-graduation bonding-curve state.

Use [ROUTER_INTEGRATION.md](ROUTER_INTEGRATION.md) for user-facing create, buy, sell, native quote, permit, and slippage-protected flows.

## Contract Role

| Contract | ABI | Use For |
| --- | --- | --- |
| BondingCurve | `abi/BondingCurve.json` | pre-graduation curve state, quote math, lifecycle events, graduation indexing |

The direct `BondingCurve.buy` and `BondingCurve.sell` functions are low-level balance-delta entrypoints. They do not take user slippage or deadline params. User-facing integrations should call `NadFunRouter`.

## Testnet Addresses

```ts
export const NADFUN_V2_TESTNET_BONDING = {
  bondingCurve: "0x27063a38eC0D3281D354090EB92e669Ed1eB956C",
} as const;
```

Mainnet: coming soon.

## Quote Token Configs

The current Monad testnet WMON and LVMon quote configs use the same curve parameters.

```ts
export const NADFUN_V2_TESTNET_QUOTE_CONFIGS = {
  wmon: {
    quoteToken: "0x5a4E0bFDeF88C9032CB4d24338C5EB3d3870BfDd",
    virtualReserve: 70000000000000000000000n,
    virtualTokenReserve: 1060569000000000000000000000n,
    minTokenReserve: 251660440677966101694915255n,
    deployFee: 10000000000000000000n,
    graduateFee: 1000000000000000000000n,
    curveProtocolFeeRate: 100,
  },
  lvmon: {
    quoteToken: "0xBe3fa50514D9617ce645a02B34F595541AF02b6b",
    lvmonMinter: "0xFe5FE61E40433aF41383d8152f96093C236231f7",
    virtualReserve: 70000000000000000000000n,
    virtualTokenReserve: 1060569000000000000000000000n,
    minTokenReserve: 251660440677966101694915255n,
    deployFee: 10000000000000000000n,
    graduateFee: 1000000000000000000000n,
    curveProtocolFeeRate: 100,
  },
} as const;
```

Human-readable values, assuming 18 decimals:

| Field | Value |
| --- | --- |
| `virtualReserve` | 70,000 quote token |
| `virtualTokenReserve` | 1,060,569,000 launch token |
| `minTokenReserve` | 251,660,440.677966101694915255 launch token |
| `deployFee` | 10 quote token |
| `graduateFee` | 1,000 quote token |
| `curveProtocolFeeRate` | 100 bps, 1% |

## Current Testnet Fee Config

NadFun V2 bonding-curve fee rates are basis points, where `10000` means 100%.

```ts
export const NADFUN_V2_TESTNET_BONDING_FEES = {
  defaultCreatorFeeRate: 100,
  curveProtocolFeeRate: 100,
  creatorFeeRates: [100, 300, 500],
  snipingPenaltyTable: [8000, 4000, 2000, 1500, 1000, 1000, 500],
} as const;
```

| Config | Current Testnet Value | Meaning |
| --- | --- | --- |
| `defaultCreatorFeeRate` | `100` | 1% creator fee used by default integrations |
| `curveProtocolFeeRate` | `100` | 1% protocol fee on bonding-curve buys and sells |
| `creatorFeeRates` | `100`, `300`, `500` | values accepted when creating a token |
| `snipingPenaltyTable` | `8000`, `4000`, `2000`, `1500`, `1000`, `1000`, `500` | extra buy-side fee during the first blocks after creation |

Sniping penalty lookup uses `block.number - createdAtBlock` as the table index.

| Blocks Since Creation | Sniping Penalty |
| --- | --- |
| `0` | 8000 bps, 80% |
| `1` | 4000 bps, 40% |
| `2` | 2000 bps, 20% |
| `3` | 1500 bps, 15% |
| `4` | 1000 bps, 10% |
| `5` | 1000 bps, 10% |
| `6` | 500 bps, 5% |
| `>= 7` | 0 bps |

The initial buy inside token creation does not apply sniping penalty. Normal buys after creation apply the current sniping penalty, plus creator and protocol fees. Sells do not apply sniping penalty.

### Quote Freshness

Pre-graduation buy quotes are block-sensitive while sniping penalty is active. Within the first 7 blocks after `createdAtBlock`, the buy-side fee can change every block.

Recommended:

- Do not cache pre-graduation buy quotes during the first 7 blocks.
- Re-read `getAmountOut` / `getAmountIn` immediately before transaction construction.
- Use short deadlines and tight `amountOutMin` / `amountInMax` bounds.
- Sells and post-penalty buys are not block-sensitive in the same way.

## Lifecycle

1. Token creation initializes a curve with virtual reserves from the selected quote config.
2. Optional initial buy runs against the curve during creation.
3. Before graduation, buys and sells update virtual reserves and emit curve events.
4. The curve graduates when `virtualTokenReserve == minTokenReserve`.
5. On graduation, the protocol pays the graduate fee, sends remaining quote/token liquidity to the LP manager, seeds the DEX pair, and emits `Graduate`.

## Curve State

Read curve state with `getCurve(token)`.

```ts
import BondingCurveAbi from "./abi/BondingCurve.json";

const curve = await publicClient.readContract({
  address: NADFUN_V2_TESTNET_BONDING.bondingCurve,
  abi: BondingCurveAbi,
  functionName: "getCurve",
  args: [token],
});
```

Important fields:

| Field | Meaning |
| --- | --- |
| `token` | launched token address; zero address means no curve |
| `creator` | creator recorded at token creation |
| `quoteToken` | quote token used for the curve |
| `virtualQuoteReserve` | quote-side virtual reserve |
| `virtualTokenReserve` | token-side virtual reserve |
| `k` | constant product invariant |
| `minTokenReserve` | graduation boundary |
| `createdAtBlock` | block used for sniping-penalty lookup |
| `graduated` | true after DEX liquidity handoff |
| `creatorFeeRate` | creator fee in bps |
| `dexType` | DEX type recorded for post-graduation pair |
| `pair` | NadFunPair address |
| `graduateFee` | quote amount paid on graduation |

## Graduation Race Conditions

A token can graduate between quote time and execution time. Direct integrations against `BondingCurve` (or against the post-graduation `NadFunPair`) must handle the lifecycle boundary themselves.

- For user-facing trades, prefer `NadFunRouter`. It checks lifecycle internally and routes the quote and execution to the correct venue.
- For direct integrators, check `getCurve(token).graduated` (or `NadFunRouter.isGraduated(token)`) close to execution time.
- Treat pre-graduation quotes near the `minTokenReserve` boundary as short-lived; the next buy can graduate the curve.
- Re-simulate before sending the transaction if the quote is more than a few blocks old.

`BondingCurve.getAmountOut` / `getAmountIn` revert after graduation; `NadFunPair.getAmountOut` / `getAmountIn` revert before graduation. Catch reverts and re-quote on the other venue if needed.

## Quote Helpers

`BondingCurve.getAmountOut` and `getAmountIn` are pre-graduation only. They revert after graduation.

```ts
const amountOut = await publicClient.readContract({
  address: NADFUN_V2_TESTNET_BONDING.bondingCurve,
  abi: BondingCurveAbi,
  functionName: "getAmountOut",
  args: [token, amountIn, true],
});

const amountIn = await publicClient.readContract({
  address: NADFUN_V2_TESTNET_BONDING.bondingCurve,
  abi: BondingCurveAbi,
  functionName: "getAmountIn",
  args: [token, amountOut, true],
});
```

`isBuy = true` means quote token in, launched token out.

`isBuy = false` means launched token in, quote token out.

## Curve Math

The V2 bonding curve uses constant product math:

```text
k = virtualQuoteReserve * virtualTokenReserve
```

Underlying amount formulas:

```text
amountOut = reserveOut - ceil(k / (reserveIn + amountIn))
amountIn = ceil(k / (reserveOut - amountOut)) - reserveIn
```

Buy path, `isBuy = true`:

```text
snipingFeeRate = getSnipingPenalty(token)
protocolFeeRate = curveProtocolFeeRate
totalFeeRate = snipingFeeRate + protocolFeeRate + creatorFeeRate
quoteInAfterFees = amountIn - ceil(amountIn * totalFeeRate / BPS)
tokenOut = virtualTokenReserve - ceil(k / (virtualQuoteReserve + quoteInAfterFees))
tokenOut is capped at virtualTokenReserve - minTokenReserve
```

Sell path, `isBuy = false`:

```text
protocolFeeRate = curveProtocolFeeRate
totalFeeRate = protocolFeeRate + creatorFeeRate
quoteOutBeforeFees = virtualQuoteReserve - ceil(k / (virtualTokenReserve + tokenIn))
quoteOutAfterFees = quoteOutBeforeFees - ceil(quoteOutBeforeFees * totalFeeRate / BPS)
```

The initial buy during token creation does not apply the sniping penalty. Normal post-create buys can include a sniping penalty based on `createdAtBlock`.

## Fees

Bonding-curve fees are quote-token-denominated.

| Fee | Applies To | Current Testnet Value | Source |
| --- | --- | --- | --- |
| deploy fee | token creation | 10 quote token | quote config |
| graduate fee | graduation | 1,000 quote token | quote config |
| curve protocol fee | pre-graduation buys and sells | 100 bps, 1% | quote config |
| creator fee | pre-graduation buys and sells | default 100 bps, 1% | create param |
| sniping penalty | normal buys during early blocks | 8000 -> 4000 -> 2000 -> 1500 -> 1000 -> 1000 -> 500 bps, then 0 | protocol sniping schedule |

Read the live sniping penalty:

```ts
const penaltyBps = await publicClient.readContract({
  address: NADFUN_V2_TESTNET_BONDING.bondingCurve,
  abi: BondingCurveAbi,
  functionName: "getSnipingPenalty",
  args: [token],
});
```

## Events

Index these events for bonding-curve state.

| Event | Use For |
| --- | --- |
| `Create` | token discovery, pair discovery, metadata, initial curve parameters |
| `Buy` | pre-graduation buy feed |
| `Sell` | pre-graduation sell feed |
| `Sync` | real and virtual reserve tracking |
| `SnipingPenalty` | early-buy penalty accounting |
| `Graduate` | DEX liquidity handoff |

Example `Create` watcher:

```ts
publicClient.watchContractEvent({
  address: NADFUN_V2_TESTNET_BONDING.bondingCurve,
  abi: BondingCurveAbi,
  eventName: "Create",
  onLogs: logs => {
    for (const log of logs) {
      const {
        creator,
        token,
        pair,
        quoteToken,
        name,
        symbol,
        tokenURI,
        virtualQuoteReserve,
        virtualTokenReserve,
        minTokenReserve,
      } = log.args;

      console.log({
        creator,
        token,
        pair,
        quoteToken,
        name,
        symbol,
        tokenURI,
        virtualQuoteReserve,
        virtualTokenReserve,
        minTokenReserve,
      });
    }
  },
});
```

## Vault Allocations

Vault allocations are configured at creation time and control how creator fees are distributed.

```ts
export const NADFUN_V2_TESTNET_VAULTS = {
  burnVault: "0xFA707fe7d2c2894bf0436c7B73947cBA9E888017",
  lpVault: "0x2acD9C75fe16c909237D9e6f080210D26c8c956D",
  creatorFeeVault: "0xfEB12B7698e296C57BBF9f0c9b38B3e908285A99",
  giftVault: "0xC112EB5C40FC9A22425300D232A31d00FF840ad0",
} as const;
```

Rules:

- `vaults.length` must be between 1 and 5.
- `bps` total must equal `10000`.
- each `bps` must be greater than 0.
- duplicate vault addresses are rejected.
- each vault must be active in the protocol vault registry.

Vaults:

| Vault | Behavior | `setupData` |
| --- | --- | --- |
| `burnVault` | uses creator fees for buyback and burn | `"0x"` |
| `lpVault` | uses creator fees for liquidity and LP burn | `"0x"` |
| `creatorFeeVault` | lets creator claim accumulated quote token | `abi.encode(address recipient)` |
| `giftVault` | makes fees claimable by GitHub/X identity | `abi.encode(GiftTarget{platform, id})` |
