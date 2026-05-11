# NadFun V2 Integration

Integration-facing docs and ABI files for NadFun V2.

## Integration Docs

| Area | Document | Covers |
| --- | --- | --- |
| Router | [ROUTER_INTEGRATION.md](ROUTER_INTEGRATION.md) | user-facing create, buy, sell, native quote, permit, and lifecycle-aware quote flows |
| Bonding Curve | [BONDING_CURVE_INTEGRATION.md](BONDING_CURVE_INTEGRATION.md) | pre-graduation curve state, quote configs, fees, vault allocations, events, graduation |
| DEX | [DEX_INTEGRATION.md](DEX_INTEGRATION.md) | Uniswap V2 differences, meme-token vs general pair behavior, pair quote calls, off-chain quote math |

## ABI Files

| File | Use For |
| --- | --- |
| `abi/NadFunRouter.json` | Main integration entry point for create, buy, sell, native quote flows, and lifecycle-aware quotes |
| `abi/BondingCurve.json` | Bonding-curve state and lifecycle events |
| `abi/NadFunFactory.json` | DEX pair lookup and pair indexing |
| `abi/NadFunPair.json` | DEX reserves, swaps, LP token reads, and pair events |
| `abi/FeeCollector.json` | DEX meme-token pair fee config and pair classification |

## Aggregator Metadata

NadFun V2 is Uniswap V2-compatible at the pair interface level, but meme-token pairs require custom quote-token fee math. Do not submit or integrate NadFun V2 as a drop-in standard Uniswap V2 fork without the fee rules in [DEX_INTEGRATION.md](DEX_INTEGRATION.md).

```ts
export const NADFUN_V2_MAINNET_AGGREGATOR_METADATA = {
  name: "NadFun V2",
  chainName: "Monad Mainnet",
  chainId: 143,
  nativeToken: {
    symbol: "MON",
    decimals: 18,
  },
  wrappedNative: {
    symbol: "WMON",
    address: "0x3bd359C1119dA7Da1D913D1C4D2B7c461115433A",
    decimals: 18,
  },
  supportedQuoteTokens: [
    {
      symbol: "WMON",
      address: "0x3bd359C1119dA7Da1D913D1C4D2B7c461115433A",
      decimals: 18,
    },
    {
      symbol: "LVMON",
      address: "0x91b81bfbe3A747230F0529Aa28d8b2Bc898E6D56",
      decimals: 18,
    },
  ],
  nadFunRouterAddress: "0x8986C8fD44eb85294A725a7e61AF35E76bA26F91",
  nadFunFactoryAddress: "0xA25b13127e63ddae6d0b35570FF3D39dBD621001",
  feeCollectorAddress: "0xE1C8b73343f5A83EBe165BE90470d84B00e33022",
  bondingCurveAddress: "0x9f3832732923252A21044F21eE6bd87F09514ae4",
  deploymentBlocks: {
    bondingCurve: 73857231,
    nadFunRouter: 73857240,
    feeCollector: 73857254,
    nadFunFactory: 73857264,
  },
  lpFeeBps: 25,
  standardUniswapV2RouterAddress: null,
  standardUniswapV2InitCodeHash: null,
  webUrl: "coming soon",
  tokenlistUrl: "coming soon",
} as const;

export const NADFUN_V2_TESTNET_AGGREGATOR_METADATA = {
  name: "NadFun V2",
  chainName: "Monad Testnet",
  chainId: 10143,
  nativeToken: {
    symbol: "MON",
    decimals: 18,
  },
  wrappedNative: {
    symbol: "WMON",
    address: "0x5a4E0bFDeF88C9032CB4d24338C5EB3d3870BfDd",
    decimals: 18,
  },
  supportedQuoteTokens: [
    {
      symbol: "WMON",
      address: "0x5a4E0bFDeF88C9032CB4d24338C5EB3d3870BfDd",
      decimals: 18,
    },
    {
      symbol: "LVMON",
      address: "0xBe3fa50514D9617ce645a02B34F595541AF02b6b",
      decimals: 18,
    },
  ],
  nadFunRouterAddress: "0x75588668999cA0557b78046b8a5E86b47b9234ec",
  nadFunFactoryAddress: "0x59C51c66B79c68F63d5446940CD13b6968788e36",
  feeCollectorAddress: "0x653cf4297fB3f7804173b8449950E20812DD6dC3",
  bondingCurveAddress: "0x27063a38eC0D3281D354090EB92e669Ed1eB956C",
  deploymentBlocks: {
    bondingCurve: 30418615,
    nadFunRouter: 30418626,
    feeCollector: 30418637,
    nadFunFactory: 30418645,
  },
  lpFeeBps: 25,
  standardUniswapV2RouterAddress: null,
  standardUniswapV2InitCodeHash: null,
  webUrl: "coming soon",
  tokenlistUrl: "coming soon",
} as const;
```

## Network Status

| Network | Status |
| --- | --- |
| Monad Mainnet | Available |
| Monad Testnet | Available |
