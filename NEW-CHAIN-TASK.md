# New Chain Integration: Smart Order Router

## Overview

Add a new EVM chain to the Uniswap Smart Order Router. This repo consumes the SDK packages — ensure `sdks/` changes are done first and SDK patches are applied if needed.

## Prerequisites

Gather these values before starting:

```
CHAIN_ID=<numeric>            # e.g. 545
CHAIN_NAME=<name>             # e.g. FLOW_TESTNET (UPPER_SNAKE)
CHAIN_SLUG=<slug>             # e.g. flow-testnet (kebab-case)
NATIVE_SYMBOL=<symbol>        # e.g. FLOW
NATIVE_DECIMALS=<decimals>    # e.g. 18
WRAPPED_NATIVE_ADDRESS=<addr>
BLOCK_TIME_SECONDS=<number>   # e.g. 6
RPC_ENV_VAR=<name>            # e.g. JSON_RPC_PROVIDER_FLOW_TESTNET

# Protocol support flags:
HAS_V2=<true|false>
HAS_V3=<true|false>
HAS_V4=<true|false>
HAS_MIXED=<true|false>
IS_L2_WITH_L1_FEES=<true|false>

# Stablecoin(s):
USDC_ADDRESS=<addr>
USDC_DECIMALS=<number>        # e.g. 6
USDC_SYMBOL=<symbol>          # e.g. USDC or USDCf
```

## Steps

### 1. Add chain to all chain-level arrays and mappings

**File:** `src/util/chains.ts`

This is the largest file to modify. Add the chain to each of these:

```ts
// 1. SUPPORTED_CHAINS array:
ChainId.CHAIN_NAME,

// 2. V2_SUPPORTED array (if HAS_V2):
ChainId.CHAIN_NAME,

// 3. V4_SUPPORTED array (if HAS_V4):
ChainId.CHAIN_NAME,

// 4. MIXED_SUPPORTED array (if HAS_MIXED):
ChainId.CHAIN_NAME,

// 5. HAS_L1_FEE array (only if IS_L2_WITH_L1_FEES):
ChainId.CHAIN_NAME,

// 6. ID_TO_CHAIN_ID() switch — add case:
case CHAIN_ID:
  return ChainId.CHAIN_NAME

// 7. ChainName enum:
CHAIN_NAME = 'CHAIN_SLUG',

// 8. NativeCurrencyName enum (if new native):
NATIVE_NAME = 'NATIVE_SYMBOL',

// 9. NATIVE_NAMES_BY_ID:
[ChainId.CHAIN_NAME]: [
  'NATIVE_SYMBOL', 'Wrapped NATIVE_SYMBOL',
  '0xeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeee',
  '0xdeaddeaddeaddeaddeaddeaddeaddeaddead0000',
],

// 10. NATIVE_CURRENCY:
[ChainId.CHAIN_NAME]: NativeCurrencyName.NATIVE_NAME,

// 11. ID_TO_NETWORK_NAME() switch:
case ChainId.CHAIN_NAME:
  return ChainName.CHAIN_NAME

// 12. ID_TO_PROVIDER() switch:
case ChainId.CHAIN_NAME:
  return process.env.RPC_ENV_VAR!

// 13. WRAPPED_NATIVE_CURRENCY:
[ChainId.CHAIN_NAME]: new Token(
  ChainId.CHAIN_NAME,
  'WRAPPED_NATIVE_ADDRESS',
  NATIVE_DECIMALS,
  'WNATIVE',
  'Wrapped Native'
),

// 14. Custom NativeCurrency class (if not ETH-like):
class ChainNameNativeCurrency extends NativeCurrency {
  equals(other: Currency): boolean {
    return other.isNative && other.chainId === this.chainId
  }
  get wrapped(): Token {
    return WRAPPED_NATIVE_CURRENCY[ChainId.CHAIN_NAME]!
  }
  constructor() {
    super(ChainId.CHAIN_NAME, NATIVE_DECIMALS, 'NATIVE_SYMBOL', 'Native Name')
  }
}

// 15. nativeOnChain() — add guard function + conditional:
function isChainName(chainId: number): chainId is ChainId.CHAIN_NAME {
  return chainId === ChainId.CHAIN_NAME
}
// In nativeOnChain():
if (isChainName(chainId)) return new ChainNameNativeCurrency()
```

### 2. Add contract address mappings

**File:** `src/util/addresses.ts`

Add to each map (use `CHAIN_TO_ADDRESSES_MAP` from sdk-core or hardcode):

```ts
// V3_CORE_FACTORY_ADDRESSES
[ChainId.CHAIN_NAME]: CHAIN_TO_ADDRESSES_MAP[ChainId.CHAIN_NAME].v3CoreFactoryAddress,

// QUOTER_V2_ADDRESSES
[ChainId.CHAIN_NAME]: CHAIN_TO_ADDRESSES_MAP[ChainId.CHAIN_NAME].quoterAddress,

// NEW_QUOTER_V2_ADDRESSES (same as above, or separate if different)
[ChainId.CHAIN_NAME]: CHAIN_TO_ADDRESSES_MAP[ChainId.CHAIN_NAME].quoterAddress,

// PROTOCOL_V4_QUOTER_ADDRESSES (if HAS_V4)
[ChainId.CHAIN_NAME]: CHAIN_TO_ADDRESSES_MAP[ChainId.CHAIN_NAME].v4QuoterAddress!,

// UNISWAP_MULTICALL_ADDRESSES
[ChainId.CHAIN_NAME]: CHAIN_TO_ADDRESSES_MAP[ChainId.CHAIN_NAME].multicallAddress,

// STATE_VIEW_ADDRESSES (if HAS_V4)
[ChainId.CHAIN_NAME]: CHAIN_TO_ADDRESSES_MAP[ChainId.CHAIN_NAME].v4StateView!,

// WETH9 object
[ChainId.CHAIN_NAME]: new Token(
  ChainId.CHAIN_NAME,
  'WRAPPED_NATIVE_ADDRESS',
  NATIVE_DECIMALS,
  'WNATIVE',
  'Wrapped Native'
),
```

### 3. Configure gas costs

**File:** `src/routers/alpha-router/gas-models/gas-costs.ts`

Add chain to each function. Use similar chain as baseline:

```ts
// BASE_SWAP_COST — return BigNumber.from(2000) for L2-like, 135000 for mainnet-like
case ChainId.CHAIN_NAME:
  return BigNumber.from(2000)

// COST_PER_INIT_TICK — typically 31000 for L2s
case ChainId.CHAIN_NAME:
  return BigNumber.from(31000)

// COST_PER_HOP — typically 80000
case ChainId.CHAIN_NAME:
  return BigNumber.from(80000)
```

Also add stablecoins to `usdGasTokensByChain`:

```ts
[ChainId.CHAIN_NAME]: [USDC_CHAIN_NAME],
```

### 4. Configure cache block TTL

**File:** `src/util/defaultBlocksToLive.ts`

```ts
// Formula: 3600 / BLOCK_TIME_SECONDS (= 1 hour of blocks)
[ChainId.CHAIN_NAME]: Math.ceil(3600 / BLOCK_TIME_SECONDS),
```

### 5. Configure routing parameters

**File:** `src/routers/alpha-router/config.ts`

Add chain to `DEFAULT_ROUTING_CONFIG_BY_CHAIN()`. Either join an existing case group (e.g. Optimism-like L2 config) or create a custom config:

```ts
case ChainId.CHAIN_NAME:
  // Add to existing L2-like case block, or create new config
```

### 6. Define token constants

**File:** `src/providers/token-provider.ts`

```ts
export const USDC_CHAIN_NAME = new Token(
  ChainId.CHAIN_NAME,
  'USDC_ADDRESS',
  USDC_DECIMALS,
  'USDC_SYMBOL',
  'USD Coin'
)

// In USDC_ON() function — add case:
case ChainId.CHAIN_NAME:
  return USDC_CHAIN_NAME
```

### 7. Configure base tokens for routing (4 files)

Base tokens are used in multiple places for pool discovery and routing. Add entries in **all four** files:

**File:** `src/routers/legacy-router/bases.ts`

```ts
// In BASES_TO_CHECK_TRADES_AGAINST:
[ChainId.CHAIN_NAME]: [
  WRAPPED_NATIVE_CURRENCY[ChainId.CHAIN_NAME]!,
  USDC_CHAIN_NAME,
],
```

**File:** `src/routers/alpha-router/functions/get-candidate-pools.ts`

```ts
// In baseTokensByChain (import the stablecoin at the top):
[ChainId.CHAIN_NAME]: [
  USDC_CHAIN_NAME,
  WRAPPED_NATIVE_CURRENCY[ChainId.CHAIN_NAME]!,
],
```

**File:** `src/providers/v2/static-subgraph-provider.ts`

```ts
// In BASES_TO_CHECK_TRADES_AGAINST (import the stablecoin at the top):
[ChainId.CHAIN_NAME]: [
  WRAPPED_NATIVE_CURRENCY[ChainId.CHAIN_NAME]!,
  USDC_CHAIN_NAME,
],
```

**File:** `src/providers/v3/static-subgraph-provider.ts`

```ts
// In BASES_TO_CHECK_TRADES_AGAINST (import the stablecoin at the top):
[ChainId.CHAIN_NAME]: [
  WRAPPED_NATIVE_CURRENCY[ChainId.CHAIN_NAME]!,
  USDC_CHAIN_NAME,
],
```

### 8. Configure cache seed tokens

**File:** `src/providers/caching-token-provider.ts`

```ts
// In CACHE_SEED_TOKENS:
[ChainId.CHAIN_NAME]: {
  USDC: USDC_CHAIN_NAME,
  WNAT: WRAPPED_NATIVE_CURRENCY[ChainId.CHAIN_NAME],
},
```

### 9. Configure caching subgraph provider base tokens

**File:** `src/providers/caching-subgraph-provider.ts`

```ts
// In the base tokens map (import stablecoin at the top):
[ChainId.CHAIN_NAME]: [
  nativeOnChain(ChainId.CHAIN_NAME),
  WRAPPED_NATIVE_CURRENCY[ChainId.CHAIN_NAME]!,
  USDC_CHAIN_NAME,
],
```

### 10. Add USD gas tokens

**File:** `src/routers/alpha-router/gas-models/gas-model.ts`

```ts
// In usdGasTokensByChain (import the stablecoin at the top):
[ChainId.CHAIN_NAME]: [USDC_CHAIN_NAME],
```

### 11. Configure OnChainQuoteProvider (optional)

**File:** `src/routers/alpha-router/alpha-router.ts`

Add a case for your chain in the `OnChainQuoteProvider` constructor switch. Most chains can join the default L2-like case:

```ts
case ChainId.CHAIN_NAME:
  this.onChainQuoteProvider = new OnChainQuoteProvider(
    chainId,
    provider,
    this.multicall2Provider,
    // ... standard params
  )
```

### 12. Configure Tenderly simulation (optional)

**File:** `src/providers/tenderly-simulation-provider.ts`

Add the Tenderly node gateway URL if Tenderly supports your chain:

```ts
case ChainId.CHAIN_NAME:
  return 'https://CHAIN_SLUG.gateway.tenderly.co/'
```

### 13. Add L2 fee chain config (only if IS_L2_WITH_L1_FEES)

**File:** `src/util/l2FeeChains.ts`

```ts
// Add to opStackChains array:
ChainId.CHAIN_NAME,
```

### 14. Build and verify

```bash
cd smart-order-router
yarn build
```

## Files Changed (summary)

| File | Change |
|------|--------|
| `src/util/chains.ts` | Chain arrays, enums, native currency, provider mapping |
| `src/util/addresses.ts` | All address maps (V3 + V4 quoter, state view, multicall, WETH9) |
| `src/routers/alpha-router/gas-models/gas-costs.ts` | Gas cost functions (BASE_SWAP_COST, COST_PER_INIT_TICK, COST_PER_HOP) |
| `src/routers/alpha-router/gas-models/gas-model.ts` | `usdGasTokensByChain` entry |
| `src/util/defaultBlocksToLive.ts` | Cache TTL |
| `src/routers/alpha-router/config.ts` | Routing config |
| `src/routers/alpha-router/alpha-router.ts` | OnChainQuoteProvider case (optional) |
| `src/routers/alpha-router/functions/get-candidate-pools.ts` | Base tokens for pool candidate selection |
| `src/providers/token-provider.ts` | Stablecoin tokens + USDC_ON() |
| `src/providers/caching-token-provider.ts` | CACHE_SEED_TOKENS entry |
| `src/providers/caching-subgraph-provider.ts` | Base tokens for subgraph caching |
| `src/providers/v2/static-subgraph-provider.ts` | Base tokens for V2 static pools |
| `src/providers/v3/static-subgraph-provider.ts` | Base tokens for V3 static pools |
| `src/providers/tenderly-simulation-provider.ts` | Tenderly gateway URL (optional) |
| `src/routers/legacy-router/bases.ts` | Base trade tokens |
| `src/util/l2FeeChains.ts` | L1 fee handling (if L2) |

## Validation

- `yarn build` passes
- Chain is in SUPPORTED_CHAINS, appropriate protocol arrays
- `nativeOnChain(CHAIN_ID)` returns correct native currency
- `WRAPPED_NATIVE_CURRENCY[CHAIN_ID]` returns wrapped token
- Gas costs are reasonable for the chain type
- At least one stablecoin in `usdGasTokensByChain`

## Note on SDK Patches

If this repo cannot update its `@uniswap/sdk-core` dependency directly, apply a patch using `patch-package` to add the new `ChainId` enum value. Check `patches/` directory for examples.
