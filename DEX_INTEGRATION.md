# NadFun V2 DEX Aggregator Integration

NadFun V2 DEX is Uniswap V2-compatible at the pair interface level, but it has a custom quote-token-denominated fee model.

Each meme-token pair has a configured quote token. In addition to the LP fee, swaps pay a creator/protocol fee in the quote token.

If the quote token is the input token, the extra fee is taken from quote input.

If the quote token is the output token, the user receives a net quote output, and the creator/protocol fee is grossed up from that net output.

Integrators must not use standard Uniswap V2 `getAmountOut` / `getAmountIn` formulas for meme-token pairs without applying this custom fee logic.

## ABIs

| Contract | ABI | Use For |
| --- | --- | --- |
| NadFunFactory | `abi/NadFunFactory.json` | standard V2-style factory pair lookup and `PairCreated` indexing |
| NadFunPair | `abi/NadFunPair.json` | standard V2-style pair reads, swaps, LP reads, and events; plus NadFun quote calls |
| FeeCollector | `abi/FeeCollector.json` | meme-token pair fee config and quote token |

## Testnet Addresses

```ts
export const NADFUN_V2_TESTNET_DEX = {
  nadFunFactory: "0x59C51c66B79c68F63d5446940CD13b6968788e36",
  feeCollector: "0x653cf4297fB3f7804173b8449950E20812DD6dC3",
} as const;
```

Mainnet: coming soon.

## Uniswap V2 Compatibility

Use the normal Uniswap V2 pair mental model for:

- `token0()` / `token1()` sorted token order
- `getReserves()` reserve ordering
- `Swap` and `Sync` event indexing
- push-token `swap(amount0Out, amount1Out, to, data)` execution
- LP token reads and LP supply

Only the differences below need special handling.

| Area | Uniswap V2 | NadFun V2 DEX |
| --- | --- | --- |
| LP fee | commonly 30 bps | `25` bps |
| Extra swap fee | none | meme-token pairs charge creator fee + DEX protocol fee in quote token |
| Pair type | one AMM type | meme-token pair or general pair |
| Quote math | reserve formula only | reserve formula + NadFun fee logic for meme-token pairs |
| Pair quote calls | not in canonical pair | `NadFunPair.getAmountOut` / `getAmountIn` are available |
| Flash callback | `uniswapV2Call` | `nadFunCall` |
| Minimum liquidity recipient | commonly `address(0)` | `0xdead` |
| `sync()` before LP exists | usually allowed | reverts while LP supply is zero |

## Pair Types

`NadFunPair` uses the same AMM mechanics for both pair types. The difference is whether `FeeCollector.getFeeConfig(pair)` exists.

| Pair Type | Fee Config | Fee Model |
| --- | --- | --- |
| Meme-token pair | `getFeeConfig(pair)` succeeds | LP fee + creator fee + DEX protocol fee in quote token |
| General pair | `getFeeConfig(pair)` reverts | LP fee only |

For DEX quote math, use:

```text
feeRate = creatorFeeRate + dexProtocolFeeRate
```

Do not include `curveProtocolFeeRate` in DEX quote math.

## Read Fee Config

Use `FeeCollector.getFeeConfig(pair)` to classify the pair and get quote-token fee inputs.

```ts
import FeeCollectorAbi from "./abi/FeeCollector.json";
import NadFunPairAbi from "./abi/NadFunPair.json";

const ZERO_ADDRESS = "0x0000000000000000000000000000000000000000";

async function getPairFeeInputs(publicClient: PublicClient, pair: `0x${string}`) {
  const feeCollector = await publicClient.readContract({
    address: pair,
    abi: NadFunPairAbi,
    functionName: "feeCollector",
  });

  try {
    const config = await publicClient.readContract({
      address: feeCollector,
      abi: FeeCollectorAbi,
      functionName: "getFeeConfig",
      args: [pair],
    });

    return {
      pairType: "memeToken" as const,
      baseToken: config.baseToken,
      quoteToken: config.quoteToken,
      creatorFeeRate: BigInt(config.creatorFeeRate),
      dexProtocolFeeRate: BigInt(config.dexProtocolFeeRate),
      feeRate: BigInt(config.creatorFeeRate) + BigInt(config.dexProtocolFeeRate),
    };
  } catch {
    return {
      pairType: "general" as const,
      quoteToken: ZERO_ADDRESS,
      feeRate: 0n,
    };
  }
}
```

## Contract Quote Calls

For the simplest integration, call the pair quote functions. These calls apply the correct pair type and fee logic internally.

Exact input:

```ts
const amountOut = await publicClient.readContract({
  address: pair,
  abi: NadFunPairAbi,
  functionName: "getAmountOut",
  args: [tokenIn, amountIn],
});
```

Exact output:

```ts
const amountIn = await publicClient.readContract({
  address: pair,
  abi: NadFunPairAbi,
  functionName: "getAmountIn",
  args: [tokenOut, amountOut],
});
```

For meme-token pairs:

| Intent | Exact Input | Exact Output |
| --- | --- | --- |
| Buy base token | `tokenIn = quoteToken` | `tokenOut = baseToken` |
| Sell base token | `tokenIn = baseToken` | `tokenOut = quoteToken` |

For exact-output sells, `amountOut` is the net quote amount the user wants to receive.

## Off-Chain Quote Math

Use this path when your aggregator already has current reserves and fee config.

Constants:

```ts
const BPS = 10_000n;
const LP_FEE_RATE = 25n;
```

### Meme-Token Pair Formula

Definitions:

```text
feeRate = creatorFeeRate + dexProtocolFeeRate
```

Buy exact input, quote token in -> base token out:

```text
tokenIn = quoteToken
reserveIn = reserve for quoteToken
reserveOut = reserve for baseToken
totalFeeRate = LP_FEE_RATE + feeRate

amountInWithFee = amountIn * (BPS - totalFeeRate)
amountOut = amountInWithFee * reserveOut / (reserveIn * BPS + amountInWithFee)
```

Buy exact output, desired base token out -> quote token in:

```text
tokenOut = baseToken
tokenIn = quoteToken
reserveIn = reserve for quoteToken
reserveOut = reserve for baseToken
totalFeeRate = LP_FEE_RATE + feeRate

amountIn = ceil(reserveIn * BPS * amountOut / ((BPS - totalFeeRate) * (reserveOut - amountOut)))
```

Sell exact input, base token in -> net quote token out:

```text
tokenIn = baseToken
reserveIn = reserve for baseToken
reserveOut = reserve for quoteToken

amountInWithLpFee = amountIn * (BPS - LP_FEE_RATE)
amountOutBeforeSwapFee = amountInWithLpFee * reserveOut / (reserveIn * BPS + amountInWithLpFee)
swapFee = ceil(amountOutBeforeSwapFee * feeRate / (BPS - LP_FEE_RATE))
amountOut = amountOutBeforeSwapFee - swapFee
```

Sell exact output, desired net quote token out -> base token in:

```text
tokenOut = quoteToken
tokenIn = baseToken
reserveIn = reserve for baseToken
reserveOut = reserve for quoteToken

amountOutBeforeSwapFee = ceil(amountOut * (BPS - LP_FEE_RATE) / (BPS - LP_FEE_RATE - feeRate))
amountIn = ceil(reserveIn * BPS * amountOutBeforeSwapFee / ((BPS - LP_FEE_RATE) * (reserveOut - amountOutBeforeSwapFee)))
```

### TypeScript Helpers

```ts
function ceilDiv(a: bigint, b: bigint): bigint {
  return (a + b - 1n) / b;
}

function sameAddress(a: string, b: string): boolean {
  return a.toLowerCase() === b.toLowerCase();
}

function getMemePairReserves(args: {
  quoteToken: string;
  token0: string;
  reserve0: bigint;
  reserve1: bigint;
}) {
  return sameAddress(args.quoteToken, args.token0)
    ? { reserveQuote: args.reserve0, reserveBase: args.reserve1 }
    : { reserveQuote: args.reserve1, reserveBase: args.reserve0 };
}

export function getMemeBuyAmountOut(args: {
  amountIn: bigint;
  reserveQuote: bigint;
  reserveBase: bigint;
  feeRate: bigint;
}): bigint {
  const { amountIn, reserveQuote, reserveBase, feeRate } = args;
  if (amountIn <= 0n) throw new Error("INSUFFICIENT_INPUT_AMOUNT");
  if (reserveQuote <= 0n || reserveBase <= 0n) throw new Error("INSUFFICIENT_LIQUIDITY");
  if (LP_FEE_RATE + feeRate >= BPS) throw new Error("INVALID_FEE_RATE");

  const totalFeeRate = LP_FEE_RATE + feeRate;
  const amountInWithFee = amountIn * (BPS - totalFeeRate);
  return (amountInWithFee * reserveBase) / (reserveQuote * BPS + amountInWithFee);
}

export function getMemeBuyAmountIn(args: {
  amountOut: bigint;
  reserveQuote: bigint;
  reserveBase: bigint;
  feeRate: bigint;
}): bigint {
  const { amountOut, reserveQuote, reserveBase, feeRate } = args;
  if (amountOut <= 0n) throw new Error("INSUFFICIENT_OUTPUT_AMOUNT");
  if (reserveQuote <= 0n || reserveBase <= 0n || amountOut >= reserveBase) {
    throw new Error("INSUFFICIENT_LIQUIDITY");
  }
  if (LP_FEE_RATE + feeRate >= BPS) throw new Error("INVALID_FEE_RATE");

  const totalFeeRate = LP_FEE_RATE + feeRate;
  return ceilDiv(
    reserveQuote * BPS * amountOut,
    (BPS - totalFeeRate) * (reserveBase - amountOut),
  );
}

export function getMemeSellAmountOut(args: {
  amountIn: bigint;
  reserveBase: bigint;
  reserveQuote: bigint;
  feeRate: bigint;
}): bigint {
  const { amountIn, reserveBase, reserveQuote, feeRate } = args;
  if (amountIn <= 0n) throw new Error("INSUFFICIENT_INPUT_AMOUNT");
  if (reserveBase <= 0n || reserveQuote <= 0n) throw new Error("INSUFFICIENT_LIQUIDITY");
  if (LP_FEE_RATE + feeRate >= BPS) throw new Error("INVALID_FEE_RATE");

  const amountInWithLpFee = amountIn * (BPS - LP_FEE_RATE);
  const amountOutBeforeSwapFee = (amountInWithLpFee * reserveQuote) / (reserveBase * BPS + amountInWithLpFee);
  if (feeRate === 0n) return amountOutBeforeSwapFee;

  const swapFee = ceilDiv(amountOutBeforeSwapFee * feeRate, BPS - LP_FEE_RATE);
  return amountOutBeforeSwapFee - swapFee;
}

export function getMemeSellAmountIn(args: {
  amountOut: bigint;
  reserveBase: bigint;
  reserveQuote: bigint;
  feeRate: bigint;
}): bigint {
  const { amountOut, reserveBase, reserveQuote, feeRate } = args;
  if (amountOut <= 0n) throw new Error("INSUFFICIENT_OUTPUT_AMOUNT");
  if (reserveBase <= 0n || reserveQuote <= 0n || amountOut >= reserveQuote) {
    throw new Error("INSUFFICIENT_LIQUIDITY");
  }
  if (LP_FEE_RATE + feeRate >= BPS) throw new Error("INVALID_FEE_RATE");

  const amountOutBeforeSwapFee = feeRate > 0n
    ? ceilDiv(amountOut * (BPS - LP_FEE_RATE), BPS - LP_FEE_RATE - feeRate)
    : amountOut;
  if (amountOutBeforeSwapFee >= reserveQuote) throw new Error("INSUFFICIENT_LIQUIDITY");

  return ceilDiv(
    reserveBase * BPS * amountOutBeforeSwapFee,
    (BPS - LP_FEE_RATE) * (reserveQuote - amountOutBeforeSwapFee),
  );
}
```

### General Pair Formula

General pairs do not have a configured quote token and do not charge creator/protocol fees. Use the same constant-product formula with NadFun's `25` bps LP fee:

```text
amountInAfterLpFee = amountIn * (BPS - LP_FEE_RATE)
amountOut = amountInAfterLpFee * reserveOut / (reserveIn * BPS + amountInAfterLpFee)
```

Exact output:

```text
amountIn = ceil(reserveIn * BPS * amountOut / ((BPS - LP_FEE_RATE) * (reserveOut - amountOut)))
```

## Events

Index `NadFunPair.Swap` and `NadFunPair.Sync`.

For meme-token pairs:

- quote token in, meme token out: buy
- meme token in, quote token out: sell

For general pairs, classify side using your aggregator's selected base/quote display convention.

## Aggregator Checklist

- Load `abi/NadFunFactory.json`, `abi/NadFunPair.json`, and `abi/FeeCollector.json`.
- Use standard V2-style factory and pair reads for discovery, reserves, and events.
- Classify the pair with `FeeCollector.getFeeConfig(pair)`.
- For meme-token pairs, apply quote-token creator/protocol fee logic.
- For general pairs, use LP-fee-only constant-product math with `25` bps.
- Do not use standard Uniswap V2 amount formulas for meme-token pairs.
- Use `NadFunPair.getAmountOut` / `getAmountIn` if you do not want to maintain fee config off-chain.
