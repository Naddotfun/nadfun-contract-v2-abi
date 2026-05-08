# NadFun V2 Router Integration

`NadFunRouter` is the user-facing integration entry point for NadFun V2.

Use it for:

- token creation
- ERC20 quote buys and sells
- native MON buys and sells
- permit-based buys and sells
- exact-input and exact-output trades
- lifecycle-aware quotes that route to bonding curve before graduation and DEX after graduation

`NadFunRouter` is separate from the DEX aggregator document. The DEX document explains direct pair integration only.

## Contract Role

| Contract | ABI | Use For |
| --- | --- | --- |
| NadFunRouter | `abi/NadFunRouter.json` | create, buy, sell, native quote flows, permits, lifecycle-aware quotes |

## Testnet Addresses

```ts
export const NADFUN_V2_TESTNET_ROUTER = {
  nadFunRouter: "0x75588668999cA0557b78046b8a5E86b47b9234ec",
  wmon: "0x5a4E0bFDeF88C9032CB4d24338C5EB3d3870BfDd",
  lvmon: "0xBe3fa50514D9617ce645a02B34F595541AF02b6b",
  lvmonMinter: "0xFe5FE61E40433aF41383d8152f96093C236231f7",
} as const;
```

Mainnet: coming soon.

## Router Behavior

The router checks token lifecycle internally:

| Token State | Router Route |
| --- | --- |
| pre-graduation | bonding curve |
| post-graduation | DEX pair |

Use `isGraduated(token)` only when your UI needs to show lifecycle state.

```ts
const graduated = await publicClient.readContract({
  address: NADFUN_V2_TESTNET_ROUTER.nadFunRouter,
  abi: NadFunRouterAbi,
  functionName: "isGraduated",
  args: [token],
});
```

## Quote Helpers

Lifecycle-aware quotes:

```ts
const amountOut = await publicClient.readContract({
  address: NADFUN_V2_TESTNET_ROUTER.nadFunRouter,
  abi: NadFunRouterAbi,
  functionName: "getAmountOut",
  args: [token, amountIn, isBuy],
});

const amountIn = await publicClient.readContract({
  address: NADFUN_V2_TESTNET_ROUTER.nadFunRouter,
  abi: NadFunRouterAbi,
  functionName: "getAmountIn",
  args: [token, amountOut, isBuy],
});
```

`isBuy = true` means quote token in, launched token out.

`isBuy = false` means launched token in, quote token out.

For router integrations, use only `getAmountOut` and `getAmountIn` for user quotes. The router checks whether the token is pre-graduation or post-graduation and routes the quote accordingly.

Use the ABI parameter names directly:

| Function | Input Field | Output |
| --- | --- | --- |
| `getAmountOut(token, amountIn, isBuy)` | `amountIn` | `amountOut` |
| `getAmountIn(token, amountOut, isBuy)` | `amountOut` | `amountIn` |

## Token Creation

Create params:

| Field | Meaning |
| --- | --- |
| `name` | ERC20 name |
| `symbol` | ERC20 symbol |
| `tokenURI` | metadata URI, for example `https://storage.nadapp.net/...` |
| `quoteToken` | allowed quote token, such as WMON or LVMon |
| `creatorFeeRate` | creator fee in bps |
| `vaults` | creator-fee vault allocations |
| `salt` | deterministic token clone salt |
| `dexType` | DEX type; use `0` for NadFun V2 pair integration |
| `buyQuoteAmount` | quote amount used for the optional initial buy |
| `deadline` | Unix timestamp in seconds |

ERC20 quote create:

```ts
import NadFunRouterAbi from "./abi/NadFunRouter.json";

const hash = await walletClient.writeContract({
  address: NADFUN_V2_TESTNET_ROUTER.nadFunRouter,
  abi: NadFunRouterAbi,
  functionName: "create",
  args: [{
    name: "My Token",
    symbol: "MTK",
    tokenURI: "https://storage.nadapp.net/metadata/my-token.json",
    quoteToken,
    creatorFeeRate: 100,
    vaults,
    salt,
    dexType: 0,
    buyQuoteAmount,
    deadline,
  }],
  account,
});
```

Before `create`, approve the router to spend:

```text
quoteRequired = deployFee + buyQuoteAmount
```

Native MON create:

```ts
const quoteRequired = deployFee + buyQuoteAmount;

const hash = await walletClient.writeContract({
  address: NADFUN_V2_TESTNET_ROUTER.nadFunRouter,
  abi: NadFunRouterAbi,
  functionName: "createWithNative",
  args: [{
    name: "My Token",
    symbol: "MTK",
    tokenURI: "https://storage.nadapp.net/metadata/my-token.json",
    quoteToken,
    creatorFeeRate: 100,
    vaults,
    salt,
    dexType: 0,
    buyQuoteAmount,
    deadline,
  }],
  value: quoteRequired,
  account,
});
```

`createWithNative` supports native-backed quote tokens such as WMON and LVMon. Excess native is refunded.

## Vault Allocations

Use these testnet vault addresses when building `CreateParams.vaults`.

```ts
export const NADFUN_V2_TESTNET_VAULTS = {
  burnVault: "0xFA707fe7d2c2894bf0436c7B73947cBA9E888017",
  lpVault: "0x2acD9C75fe16c909237D9e6f080210D26c8c956D",
  creatorFeeVault: "0xfEB12B7698e296C57BBF9f0c9b38B3e908285A99",
  giftVault: "0xC112EB5C40FC9A22425300D232A31d00FF840ad0",
} as const;
```

Direct creator claim vault:

```ts
import { encodeAbiParameters } from "viem";

const vaults = [{
  vault: NADFUN_V2_TESTNET_VAULTS.creatorFeeVault,
  bps: 10_000,
  setupData: encodeAbiParameters(
    [{ type: "address" }],
    [creatorAddress],
  ),
}];
```

Gift vault:

```ts
const vaults = [{
  vault: NADFUN_V2_TESTNET_VAULTS.giftVault,
  bps: 10_000,
  setupData: encodeAbiParameters(
    [{
      type: "tuple",
      components: [
        { name: "platform", type: "uint8" },
        { name: "id", type: "string" },
      ],
    }],
    [{ platform: 1, id: "nadfun" }],
  ),
}];
```

`platform = 0` is GitHub. `platform = 1` is X.

## Exact-Input Trades

ERC20 quote buy:

```ts
const hash = await walletClient.writeContract({
  address: NADFUN_V2_TESTNET_ROUTER.nadFunRouter,
  abi: NadFunRouterAbi,
  functionName: "buy",
  args: [{
    amountIn,
    amountOutMin,
    token,
    to,
    deadline,
  }],
  account,
});
```

Native MON buy:

```ts
const hash = await walletClient.writeContract({
  address: NADFUN_V2_TESTNET_ROUTER.nadFunRouter,
  abi: NadFunRouterAbi,
  functionName: "buyWithNative",
  args: [{
    amountOutMin,
    token,
    to,
    deadline,
  }],
  value: amountIn,
  account,
});
```

ERC20 quote sell:

```ts
const hash = await walletClient.writeContract({
  address: NADFUN_V2_TESTNET_ROUTER.nadFunRouter,
  abi: NadFunRouterAbi,
  functionName: "sell",
  args: [{
    amountIn,
    amountOutMin,
    token,
    to,
    deadline,
  }],
  account,
});
```

Sell to native MON:

```ts
const hash = await walletClient.writeContract({
  address: NADFUN_V2_TESTNET_ROUTER.nadFunRouter,
  abi: NadFunRouterAbi,
  functionName: "sellToNative",
  args: [{
    amountIn,
    amountOutMin,
    token,
    to,
    deadline,
  }],
  account,
});
```

Approvals:

- `buy`: approve quote token to router.
- `sell`: approve launched token to router.
- `buyWithNative`: send native MON as `value`.
- `sellToNative`: approve launched token to router.

## Exact-Output Trades

Buy exact token output:

```ts
const hash = await walletClient.writeContract({
  address: NADFUN_V2_TESTNET_ROUTER.nadFunRouter,
  abi: NadFunRouterAbi,
  functionName: "exactOutBuy",
  args: [{
    amountInMax,
    amountOut,
    token,
    to,
    deadline,
  }],
  account,
});
```

Buy exact token output with native MON:

```ts
const hash = await walletClient.writeContract({
  address: NADFUN_V2_TESTNET_ROUTER.nadFunRouter,
  abi: NadFunRouterAbi,
  functionName: "exactOutBuyWithNative",
  args: [{
    amountOut,
    token,
    to,
    deadline,
  }],
  value: amountInMax,
  account,
});
```

Sell for exact quote output:

```ts
const hash = await walletClient.writeContract({
  address: NADFUN_V2_TESTNET_ROUTER.nadFunRouter,
  abi: NadFunRouterAbi,
  functionName: "exactOutSell",
  args: [{
    amountInMax,
    amountOut,
    token,
    to,
    deadline,
  }],
  account,
});
```

Sell for exact native MON output:

```ts
const hash = await walletClient.writeContract({
  address: NADFUN_V2_TESTNET_ROUTER.nadFunRouter,
  abi: NadFunRouterAbi,
  functionName: "exactOutSellToNative",
  args: [{
    amountInMax,
    amountOut,
    token,
    to,
    deadline,
  }],
  account,
});
```

Exact-output native flows use `amountInMax` as `msg.value` and refund unused input.

## Permit Flows

Permit variants skip a separate approval transaction when the token supports ERC-2612 permit.

| Function | Permit Token |
| --- | --- |
| `buyWithPermit` | quote token |
| `sellWithPermit` | launched token |
| `sellToNativeWithPermit` | launched token |

Each permit call includes `amountAllowance`, `v`, `r`, and `s`.

## Native Quote Notes

Native MON flows convert native input into the configured quote token.

| Quote Token | Native Handling |
| --- | --- |
| WMON | router wraps native MON |
| LVMon | router mints LVMon through the configured LVMon minter |

If the LVMon minter returns more LVMon than required, excess LVMon is refunded as LVMon. Excess native input is refunded as native MON.

## Events

Router events are user-action oriented.

| Event | Fields | Use For |
| --- | --- | --- |
| `Create` | `token`, `creator` | transaction-level create tracking |
| `Buy` | `buyer`, `token`, `amountIn`, `amountOut`, `graduated` | buy history across bonding curve and DEX phases |
| `Sell` | `seller`, `token`, `amountIn`, `amountOut`, `graduated` | sell history across bonding curve and DEX phases |

The `graduated` flag tells whether the trade executed through the DEX phase.

## Integration Checklist

- Use `getAmountOut` / `getAmountIn` for user quotes.
- Set `amountOutMin` on exact-input trades.
- Set `amountInMax` on exact-output trades.
- Set short `deadline` values.
- Approve the router before ERC20 quote buys or token sells.
- Use native functions only for native-backed quote tokens.
- Parse `BondingCurve.Create` when you need full token metadata and pair address.
