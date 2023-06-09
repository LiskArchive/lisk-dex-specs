```
LIP: <LIP number>
Title: Define swap interaction for DEX module
Author: Iker Alustiza <iker@lightcurve.io>
        Sergey Shemyakov <sergey.shemyakov@lightcurve.io>
Type: Standards Track
Created: <YYYY-MM-DD>
Updated: <YYYY-MM-DD>
Required: LIP "Introduce DEX module", LIP 40, LIP 51
```

## Abstract

In this LIP, we introduce the swap interaction for the DEX module and specify the commands and internal functions that define it.
Together with LIP "Introduce DEX module" and LIP "LPs", this LIP completely specifies the DEX module.
The DEX module defines a decentralized exchange based on an automated market maker with a concentrated liquidity algorithm.

## Copyright

This LIP is licensed under the [Creative Commons Zero 1.0 Universal](https://creativecommons.org/publicdomain/zero/1.0/).

## Motivation

For motivation, refer to the Motivation Section of the ["Introduce DEX Module" LIP][dexmodulelip].

## Rationale

### Types of Swap Commands

The on-chain logic of DEX module supports three different types of swaps: swap with exact input, swap with exact output and swap with price limit.

The state of a pool changes every time when liquidity is added or removed, or when tokens are swapped in the pool. The exact amount of tokens that a trader obtains in exchange for a given amount of tokens depends on the state of DEX, so it is impossible to know precisely at the moment of signing a swap transaction. Swapping with exact input or output addresses this problem: trader can specify the precise amount of tokens that they want to swap, or that they want to get after a swap, but not both.

Swap with price limit supports the task of arbitrage: a trader can swap tokens in a pool until the price in the pool is equal to the market price.

### Slippage Protection

For a given swap, slippage denotes how much the internal price is changed after the swap (measured in the percents of the initial price). Slippage estimates the difference between the price before the swap and the actual price of the swap (as the ratio of amounts of input and output tokens). Thus big slippage corresponds to more expensive trades for a trader.

All swap commands offer protection against too high slippage. When requesting a swap with exact input/ output or with price limit, a trader can specify the minimal amount of tokens to obtain/ maximal amount of tokens to pay. Moreover, a swap with price limit can be used for slippage protection: here a trader can directly specify the price limit not to be crossed during the swap.

### Fees and incentives

Trader pays certain part of the swap amount as fees, depending on the fee tier of the pool. Fees are paid in both input and output tokens, for multihop swaps the fees are also paid in all intermediate tokens. Fees are split between liquidity providers, additionally all LSK fees are shared with the exchange sidechain validators. The fee parts for different actors are given as the constants of the DEX module.

The fees and incentives define the economics of the exchange. They motivate liquidity providers to invest into pools and stimulate validators.

## Specification

In this LIP, we specify three swap commands and the corresponding internal functions of the DEX module. The DEX module has module name `MODULE_NAME_DEX`.

### Constants and Types

This LIP uses constants and types defined in the ["Introduce DEX Module" LIP][dexmodulelip]. The table below defines types specific for this LIP:

|Name            | Type              | Validation                           | Description                           |
|----------------|-------------------|--------------------------------------|---------------------------------------|
| TickID         | `bytes`           | must have length `NUM_BYTES_TICK_ID` | Identifies tick objects in Lisk DEX   |
| PoolsGraph     | `tuple[Set[TokenID], Set[PoolID]]` | None                | Represents a pools graph from the DEX router as a set of vertices and a set of edges |

 The table below specifies event-relevant constants:

| Name                              | Type     | Value | Description                                                                                                               |
| --------------------------------- | -------- | ----- | ------------------------------------------------------------------------------------------------------------------------- |
| `SWAP_FAILED_INVALID_ROUTE`       | `uint32` | 0     | Return code for the failed swap event in case of invalid swap route                                                       |
| `SWAP_FAILED_TOO_MANY_TICKS`      | `uint32` | 1     | Return code for the failed swap event in case of crossing too many ticks                                                  |
| `SWAP_FAILED_NOT_EXACT_AMOUNT`    | `uint32` | 2     | Return code for the failed swap event in case of failing to swap exact amount in or out                                                    |
| `SWAP_FAILED_NOT_ENOUGH`          | `uint32` | 3     | Return code for the failed swap event in case of insufficient amount of output tokens or excessive amount of input tokens |
| `SWAP_FAILED_INVALID_LIMIT_PRICE` | `uint32` | 4     | Return code for the failed swap event in case of invalid limit price value for the swap with price limit command          |
| `FEE_TIER_PARTITION`              | `uint32` |1000000| The inverse of the fee tier precision                                                                                     |

#### Logic from Other Modules

Calling a function `fct` implemented in another module `module` is represented by `module.fct(required inputs)`.

### Events

#### Swapped

This event is emitted when a swap is performed successfully. The name of the event is `EVENT_NAME_SWAPPED`.

##### Topics

- `senderAddress`: the address requesting a swap.

##### Data

```java
swappedEventDataSchema = {
    "type": "object",
    "required": [
        "senderAddress",
        "priceBefore",
        "priceAfter",
        "tokenIdIn",
        "amountIn",
        "tokenIdOut"
        "amountOut",
    ],
    "properties": {
        "senderAddress": {
            "dataType": "bytes",
            "fieldNumber": 1
        },
        "priceBefore": {
            "dataType": "bytes",
            "maxLength": MAX_NUM_BYTES_Q96,
            "fieldNumber": 2
        },
        "priceAfter": {
            "dataType": "bytes",
            "maxLength": MAX_NUM_BYTES_Q96,
            "fieldNumber": 3
        },
        "tokenIdIn": {
            "dataType": "bytes",
            "fieldNumber": 4
        },
        "amountIn": {
            "dataType": "uint64",
            "fieldNumber": 5
        },
        "tokenIdOut": {
            "dataType": "bytes",
            "fieldNumber": 6
        },
        "amountOut": {
            "dataType": "uint64",
            "fieldNumber": 7
        }
    }
}
```

#### SwapFailed

This event is emitted when a swap fails. The name of the event is `EVENT_NAME_SWAP_FAILED`.

##### Topics

- `senderAddress`: the address requesting a swap.

##### Data

```java
swapFailedEventDataSchema = {
    "type": "object",
    "required": [
        "senderAddress",
        "tokenIdIn",
        "tokenIdOut",
        "reason"
    ],
    "properties": {
        "senderAddress": {
            "dataType": "bytes",
            "fieldNumber": 1
        },
        "tokenIdIn": {
            "dataType": "bytes",
            "fieldNumber": 2
        },
        "tokenIdOut": {
            "dataType": "bytes",
            "fieldNumber": 3
        },
        "reason": {
            "dataType": "uint32",
            "fieldNumber": 4
        }
    }
}
```

Here the `"reason"` property specifies the reason why the swap failed and can have values from the set `{SWAP_FAILED_INVALID_ROUTE, SWAP_FAILED_TOO_MANY_TICKS, SWAP_FAILED_NOT_ENOUGH}`.

### Commands

#### swap exact input command

The swap exact input command swaps the specifies input amount of a token, from the account of the user sending the command, for an output amount of another token to a receiving account. The command parameters also specify the route of the swap from initial pool ID to final pool ID, a minimum output amount and a deadline for the swap.

##### Parameters

```java
swapExactInCommandSchema = {
    "type": "object",
    "required": [
        "tokenIdIn",
        "amountTokenIn",
        "tokenIdOut",
        "minAmountTokenOut",
        "swapRoute",
        "maxTimestampValid"
        ],
    "properties": {
        "tokenIdIn": {
            "dataType": "bytes",
            "length": NUM_BYTES_TOKEN_ID,
            "fieldNumber": 1
        },
        "amountTokenIn": {
            "dataType": "uint64",
            "fieldNumber": 2
        },
        "tokenIdOut": {
            "dataType": "bytes",
            "length": NUM_BYTES_TOKEN_ID,
            "fieldNumber": 3
        },
        "minAmountTokenOut": {
            "dataType": "uint64",
            "fieldNumber": 4
        },
        "swapRoute": {
            "type": "array",
            "fieldNumber": 5,
            "items": {
                "dataType": "bytes",
                "length": NUM_BYTES_POOL_ID
            }
        },
        "maxTimestampValid": {
            "dataType": "uint32",
            "fieldNumber": 6
        }
    }
}

```

- `tokenIdIn`: The token ID of the swap input token.
- `amountTokenIn`: The amount of the swap input token to be swapped from the account of the sender.
- `tokenIdOut`: The token ID of the swap output token.
- `minAmountTokenOut`: The minimal amount of the tokens to be received by the sender for the swap to be valid.
- `swapRoute`: An array of pool IDs specifying the route for the swap, it can have a maximum length of `MAX_HOPS_SWAP` and it cannot be empty.
- `maxTimestampValid`: A `uint32` integer with the maximal timestamp of a block where a transaction with this command can be included.

##### Verification

```python
def verify(trs: Transaction) -> None:
    if trs.params does not satisfy swapExactInCommandSchema:
        raise Exception()
    if trs.params.tokenIdIn == trs.params.tokenIdOut:
        raise Exception()
    if trs.params.swapRoute is empty or length(trs.params.swapRoute) > MAX_HOPS_SWAP:
        raise Exception()
    firstPool = trs.params.swapRoute[0]
    lastPool = trs.params.swapRoute[-1]
    if getToken0Id(firstPool) != trs.params.tokenIdIn and getToken1Id(firstPool) != trs.params.tokenIdIn:
        raise Exception()
    if getToken0Id(lastPool) != trs.params.tokenIdOut and getToken1Id(lastPool) != trs.params.tokenIdOut:
        raise Exception()
    if trs.params.maxTimestampValid < lastBlockheader.timestamp:
        raise Exception()
```

##### Execution

Processing a transaction trs with module name `MODULE_NAME_DEX` and command name `COMMAND_SWAP_EXACT_INPUT` implies the following logic:

```python
def execute(trs: Transaction) -> None:
    senderAddress = SHA256(trs.senderPublicKey)[:NUM_BYTES_ADDRESS]
    tokenIdIn = trs.params.tokenIdIn
    tokenIdOut = trs.params.tokenIdOut
    tokens = [{id: tokenIdIn, amount: trs.params.amountTokenIn}]
    swapRoute = trs.params.swapRoute
    currentHeight = height of the block containing trs
    try:
        priceBefore = computeCurrentPrice(tokenIdIn, tokenIdOut, swapRoute)
    except:
        raiseSwapException(SWAP_FAILED_INVALID_ROUTE, tokenIdIn,
                            tokenIdOut, senderAddress)
    fees = []
    numCrossedTicks = 0

    # swap along all the pools in swapRoute
    for poolId in swapRoute:
        currentTokenIn = tokens[-1]
        if getToken0Id(poolId) == currentTokenIn.id:
            zeroToOne = True
            IdOut = getToken1Id(poolId)
        else if getToken1Id(poolId) == currentTokenIn.id:
            zeroToOne = False
            IdOut = getToken0Id(poolId)

        # if zeroToOne then price decreases after the swap, otherwise it increases
        sqrtLimitPrice = zeroToOne ? MIN_SQRT_RATIO : MAX_SQRT_RATIO
        try:
            (amountIn, amountOut, feesIn, feesOut, numCrossedTicks) = swap(poolId, zeroToOne, sqrtLimitPrice, currentTokenIn.amount, True, numCrossedTicks, currentHeight)
        except ExceptionSwapCrossedTooManyTicks:    # specific exception class for crossing too many ticks
            # crossed too many ticks
            raiseSwapException(SWAP_FAILED_TOO_MANY_TICKS, tokenIdIn,
                                tokenIdOut, senderAddress)
        if amountIn != currentTokenIn.amount:
            # the pool is depleted before swapping in the full amount
            raiseSwapException(SWAP_FAILED_NOT_EXACT_AMOUNT, tokenIdIn,
                                tokenIdOut, senderAddress)
        tokens.append({id:IdOut, amount: amountOut})
        fees.append({in: feesIn, out: feesOut})

    # check that amount out is at least the minimum required amount
    if  tokens[-1].amount <  trs.params.minAmountTokenOut:
        raiseSwapException(SWAP_FAILED_NOT_ENOUGH, tokenIdIn,
                            tokenIdOut, senderAddress)

    else:
        # make the corresponding state updates
        # transfer and lock all the tokens involved in the multihop swap  
        priceAfter = computeCurrentPrice(tokenIdIn, tokenIdOut, swapRoute)
        transferToPool(senderAddress, swapRoute[0], tokenIdIn, trs.params.amountTokenIn)
        transferFeesFromPool(fees[0].in, tokenIdIn, swapRoute[0])
        for i in range(1, length(swapRoute)):
            # pay output fees, transfer to the next pool, pay input fees in the next pool
            transferFeesFromPool(fees[i-1].out, tokens[i].id, swapRoute[i-1])
            transferPoolToPool(swapRoute[i-1], swapRoute[i], tokens[i].id, tokens[i].amount)
            transferFeesFromPool(fees[i].in, tokens[i].id, swapRoute[i])
        transferFeesFromPool(fees[-1].out, tokenIdOut, swapRoute[-1])
        transferFromPool(swapRoute[-1], senderAddress, tokenIdOut, tokens[-1].amount)

        emitEvent(
            module = MODULE_NAME_DEX,
            name = EVENT_NAME_SWAPPED,
            data = {
                "senderAddress": senderAddress,
                "priceBefore": q96ToBytes(priceBefore),
                "priceAfter": q96ToBytes(priceAfter),
                "tokenIdIn": tokenIdIn,
                "amountIn": trs.params.amountTokenIn,
                "tokenIdOut": tokenIdOut,
                "amountOut": tokens[-1].amount
            },
            topics = [senderAddress]
        )  
```

where the functions `getToken0Id`, `getToken1Id`, `transferToPool`, `transferPoolToPool`, and `transferFromPool` are defined in the [Introduce DEX module LIP][dexmodulelip].

#### swap exact output command

The swap exact output command swaps an input amount of a token, from the account of the user sending the command, for the specified output amount of another token to a receiving account. The command also specifies the route of the swap from initial pool ID to final pool ID, a maximum input amount and a deadline for the swap.

##### Parameters

```java
swapExactOutCommandSchema = {
    "type": "object",
    "required": [
        "tokenIdIn",
        "maxAmountTokenIn",
        "tokenIdOut",
        "amountTokenOut",
        "swapRoute",
        "maxTimestampValid"
    ],
    "properties": {
        "tokenIdIn": {
            "dataType": "bytes",
            "length": NUM_BYTES_TOKEN_ID,
            "fieldNumber": 1
        },
        "maxAmountTokenIn": {
            "dataType": "uint64",
            "fieldNumber": 2
        },
        "tokenIdOut": {
            "dataType": "bytes",
            "length": NUM_BYTES_TOKEN_ID,
            "fieldNumber": 3
        },
        "amountTokenOut": {
            "dataType": "uint64",
            "fieldNumber": 4
        }
        "swapRoute": {
            "type": "array",
            "fieldNumber": 5,
            "items": {
                "dataType": "bytes",
                "length": NUM_BYTES_POOL_ID
            }
        },
        "maxTimestampValid": {
            "dataType": "uint32",
            "fieldNumber": 6
        }
    }
}

```

- `tokenIdIn`: The token ID of the swap input token.
- `maxAmountTokenIn`: The maximal amount of the swap input token to be swapped from the account of the sender for the swap to be valid.
- `tokenIdOut`: The token ID of the swap output token.
- `amountTokenOut`: The amount of the output tokens to be received by the sender.
- `swapRoute`: An array of pool IDs specifying the route for the swap, it can have a maximum length of `MAX_HOPS_SWAP` and it cannot be empty.
- `maxTimestampValid`: A `uint32` integer with the maximal timestamp of a block where a transaction with this command can be included.

##### Verification

```python
def verify(trs: Transaction) -> None:
    if trs.params does not satisfy swapExactOutCommandSchema:
        raise Exception()
    if trs.params.tokenIdIn == trs.params.tokenIdOut:
        raise Exception()
    if trs.params.swapRoute is empty or length(trs.params.swapRoute) > MAX_HOPS_SWAP:
        raise Exception()
    firstPool = trs.params.swapRoute[0]
    lastPool = trs.params.swapRoute[-1]
    if getToken0Id(firstPool) != trs.params.tokenIdIn and getToken1Id(firstPool) != trs.params.tokenIdIn:
        raise Exception()
    if getToken0Id(lastPool) != trs.params.tokenIdOut and getToken1Id(lastPool) != trs.params.tokenIdOut:
        raise Exception()
    if trs.params.maxTimestampValid < lastBlockheader.timestamp:
        raise Exception()
```

##### Execution

Processing a transaction trs with module name `MODULE_NAME_DEX` and command name `COMMAND_SWAP_EXACT_OUTPUT` implies the following logic:

```python
def execute(trs: Transaction) -> None:
    senderAddress = SHA256(trs.senderPublicKey)[:NUM_BYTES_ADDRESS]
    tokenIdIn = trs.params.tokenIdIn
    tokenIdOut = trs.params.tokenIdOut
    tokens = [{id: tokenIdOut, amount: trs.params.amountTokenOut}]
    inverseSwapRoute = reverse trs.params.swapRoute
    currentHeight = height of the block containing trs
    try:
        priceBefore = computeCurrentPrice(tokenIdIn, tokenIdOut, trs.params.swapRoute)
    except:
        raiseSwapException(SWAP_FAILED_INVALID_ROUTE, tokenIdIn,
                            tokenIdOut, senderAddress)
    fees = []
    numCrossedTicks = 0

    # swap along all the pools in inverseSwapRoute
    for poolId in inverseSwapRoute:
        currentTokenOut = tokens[-1]
        if getToken1Id(poolId) == currentTokenOut.id:
            zeroToOne = True
            IdIn = getToken0Id(poolId)
        else if getToken0Id(poolId) == currentTokenOut.id:
            zeroToOne = False
            IdIn = getToken1Id(poolId)

        # if zeroToOne then price decreases after the swap, otherwise it increases
        sqrtLimitPrice = zeroToOne ? MIN_SQRT_RATIO : MAX_SQRT_RATIO
        try:
            (amountIn, amountOut, feesIn, feesOut, numCrossedTicks) = swap(poolId, zeroToOne, sqrtLimitPrice, currentTokenOut.amount, False, numCrossedTicks, currentHeight)
        except ExceptionSwapCrossedTooManyTicks:    # specific exception class for crossing too many ticks
            # crossed too many ticks
            raiseSwapException(SWAP_FAILED_TOO_MANY_TICKS, tokenIdIn,
                                tokenIdOut, senderAddress)
        if amountOut != currentTokenOut.amount:
            # the pool is depleted before swapping out the full amount
            raiseSwapException(SWAP_FAILED_NOT_EXACT_AMOUNT, tokenIdIn,
                                tokenIdOut, senderAddres)
        tokens.append({id:IdIn, amount: amountIn})
        fees.append({in: feesIn, out: feesOut})

    # check that amount out is at least the minimum required amount
    if  tokens[-1].amount >  trs.params.maxAmountTokenIn:
        raiseSwapException(SWAP_FAILED_NOT_ENOUGH, tokenIdIn,
                            tokenIdOut, senderAddress)
    else:
        # make the corresponding state updates
        # transfer and lock all the tokens involved in the multihop swap
        priceAfter = computeCurrentPrice(tokenIdIn, tokenIdOut, trs.params.swapRoute)
        transferFromPool(inverseSwapRoute[0], senderAddress, tokenIdOut, tokens[0].amount)
        transferFeesFromPool(fees[0].out, tokenIdOut, inverseSwapRoute[0])
        for i in range(1, length(inverseSwapRoute)):
            # pay input fees, transfer to the next pool, pay output fees in the next pool
            transferFeesFromPool(fees[i-1].in, tokens[i].id, inverseSwapRoute[i-1])
            transferPoolToPool(inverseSwapRoute[i-1], inverseSwapRoute[i], tokens[i].id, tokens[i].amount)
            transferFeesFromPool(fees[i].out, tokens[i].id, inverseSwapRoute[i])
        transferFeesFromPool(fees[-1].in, tokenIdIn, inverseSwapRoute[-1])
        transferToPool(senderAddress, inverseSwapRoute[-1], tokenIdIn, tokens[-1].amount)

        emitEvent(
            module = MODULE_NAME_DEX,
            name = EVENT_NAME_SWAPPED,
            data = {
                "senderAddress": senderAddress,
                "priceBefore": q96ToBytes(priceBefore),
                "priceAfter": q96ToBytes(priceAfter),
                "tokenIdIn": tokenIdIn,
                "amountIn": tokens[-1].amount,
                "tokenIdOut": tokenIdOut,
                "amountOut": trs.params.amountTokenOut
            },
            topics = [senderAddress]
        )

```

where the functions `getToken0Id`, `getToken1Id`, `transferToPool`, `transferPoolToPool`, and `transferFromPool` are defined in the [Introduce DEX module LIP][dexmodulelip].

#### swap with price limit command

The swap with price limit command performs a swap in a single pool with price change at most up to the specified target price.
The command transfers at most the specified input amount from the account of the user sending the command to the pool and transfers at least the specified output amount to the specified output account.
The command also specifies a deadline for the swap.

##### Parameters

```java
swapWithPriceLimitCommandSchema = {
    "type": "object",
    "required": [
        "tokenIdIn",
        "maxAmountTokenIn",
        "tokenIdOut",
        "minAmountTokenOut",
        "poolId",
        "maxTimestampValid",
        "sqrtLimitPrice"
    ],
    "properties": {
        "tokenIdIn": {
            "dataType": "bytes",
            "length": NUM_BYTES_TOKEN_ID,
            "fieldNumber": 1
        },
        "maxAmountTokenIn": {
            "dataType": "uint64",
            "fieldNumber": 2
        },
        "tokenIdOut": {
            "dataType": "bytes",
            "length": NUM_BYTES_TOKEN_ID,
            "fieldNumber": 3
        },
        "minAmountTokenOut": {
            "dataType": "uint64",
            "fieldNumber": 4
        },
        "poolId": {
            "dataType": "bytes",
            "length": NUM_BYTES_POOL_ID,
            "fieldNumber": 5
        },
        "maxTimestampValid": {
            "dataType": "uint32",
            "fieldNumber": 6
        },
        "sqrtLimitPrice": {
            "dataType": "bytes",
            "maxLength": MAX_NUM_BYTES_Q96,
            "fieldNumber": 7
        }
    }
}
```

- `tokenIdIn`: The token ID of the swap input token.
- `maxAmountTokenIn`: The maximal amount of the swap input token to be swapped from the account of the sender.
- `tokenIdOut`: The token ID of the swap output token.
- `minAmountTokenOut`: The minimal amount of the tokens to be received by the sender for the swap to be valid.
- `poolId`: A byte array with the pool ID specifying the pool for the swap.
- `maxTimestampValid`: A `uint32` integer with the maximal timestamp of a block where a transaction with this command can be included.
- `sqrtLimitPrice`: A byte array with the sqrt limit price, in `Q96` format, not to be crossed by the swap.

##### Verification

```python
def verify(trs: Transaction) -> None:
    if trs.params does not satisfy swapWithPriceLimitCommandSchema:
        raise Exception()
    if trs.params.tokenIdIn == trs.params.tokenIdOut:
        raise Exception()
    poolId = trs.params.poolId
    if poolId not in pools:
        raise Exception()
    if getToken0Id(poolId) != trs.params.tokenIdIn and getToken1Id(poolId) != trs.params.tokenIdIn:
        raise Exception()
    if getToken0Id(poolId) != trs.params.tokenIdOut and getToken1Id(poolId) != trs.params.tokenIdOut:
        raise Exception()
    if trs.params.maxTimestampValid < lastBlockheader.timestamp:
        raise Exception()
```

##### Execution

Processing a transaction trs with module name `MODULE_NAME_DEX` and command name `COMMAND_SWAP_WITH_PRICE_LIMIT` implies the following logic:

```python
def execute(trs: Transaction) -> None:
    senderAddress = SHA256(trs.senderPublicKey)[:NUM_BYTES_ADDRESS]
    tokenIdIn = trs.params.tokenIdIn
    tokenIdOut = trs.params.tokenIdOut
    amountTokenIn = trs.params.maxAmountTokenIn
    poolId = trs.params.poolId
    sqrtLimitPrice = bytesToQ96(trs.params.sqrtLimitPrice)
    currentHeight = height of the block containing trs
    try:
        priceBefore = computeCurrentPrice(tokenIdIn, tokenIdOut, [poolId])
    except:
        raiseSwapException(SWAP_FAILED_INVALID_ROUTE, tokenIdIn,
                            tokenIdOut, senderAddress)


    if sqrtLimitPrice < MIN_SQRT_RATIO or sqrtLimitPrice > MAX_SQRT_RATIO:
        raiseSwapException(SWAP_FAILED_INVALID_LIMIT_PRICE, tokenIdIn,
                                tokenIdOut, senderAddress)

    if getToken0Id(poolId) == tokenIdIn:
        zeroToOne = True
    else: # getToken1Id(poolId) == tokenIn.id:
        zeroToOne = False

    try:
        (amountIn, amountOut, feesIn, feesOut, numCrossedTicks) = swap(poolId, zeroToOne, sqrtLimitPrice, amountTokenIn, True, 0, currentHeight)
        # the variable numCrossedTicks is not needed here and could be discarded
    except ExceptionSwapCrossedTooManyTicks:    # specific exception class for crossing too many ticks
        # crossed too many ticks
        raiseSwapException(SWAP_FAILED_TOO_MANY_TICKS, tokenIdIn,
                            tokenIdOut, senderAddress)

    # check that amount out is at least the minimum required amount
    if  amountOut <  trs.params.minAmountTokenOut:
        raiseSwapException(SWAP_FAILED_NOT_ENOUGH, tokenIdIn,
                            tokenIdOut, senderAddress)
    else:
        priceAfter = computeCurrentPrice(tokenIdIn, tokenIdOut, [poolId])
        # make the corresponding state updates
        # transfer and lock all the tokens involved in the multihop swap
        transferToPool(senderAddress, poolId, tokenIdIn, amountIn)
        transferFeesFromPool(feesIn, tokenIdIn, poolId)
        transferFromPool(poolId, senderAddress, tokenIdOut, amountOut)
        transferFeesFromPool(feesOut, tokenIdOut, poolId)

        emitEvent(
            module = MODULE_NAME_DEX,
            name = EVENT_NAME_SWAPPED,
            data = {
                "senderAddress": senderAddress,
                "priceBefore": q96ToBytes(priceBefore),
                "priceAfter": q96ToBytes(priceAfter),
                "tokenIdIn": tokenIdIn,
                "amountIn": amountIn,
                "tokenIdOut": tokenIdOut,
                "amountOut": amountOut
            },
            topics = [senderAddress]
        )
```

where the functions `getToken0Id`, `getToken1Id`, `transferToPool`, and `transferFromPool` are defined in the [Introduce DEX module LIP][dexmodulelip].

### Internal Functions

In this section we specify the internal functions that are part of the processing logic in the swap command.

#### swap

This function computes a complete swap within a single pool. It crosses as many price ticks as required as long as the total number of crossed ticks is not more than `MAX_NUMBER_CROSSED_TICKS`.
This function calls the `swapWithin` function (see below) to calculate the amounts and price update within a tick range.
Unlike the [swap function][swapuniswap] of Uniswap v3, this function does not send the output amount to an address, this will be done outside this function by calling the corresponding token module exposed functions.

##### Parameters

- `poolId`: The ID of the pool in which the swap is performed.
- `zeroToOne`: A boolean with the direction of the swap, is `True` if `token0` is the input token, `token1` is the input token otherwise.
- `sqrtLimitPrice`: A Q96 number with the price limit not to be crossed by the swap.
- `amountSpecified`: The exact amount of tokens for the swap. It is the input amount if `exactInput == True` and is the output amount otherwise.
- `exactInput`: A boolean indicating whether a swap is performed with exact input or with exact output.
- `numCrossedTicks`: The total number of ticks crossed before swapping in the given pool.
- `currentHeight`: An integer with the height of the block when the swap is performed, is needed for correct tick crossing.

##### Returns

- `amountIn`: The total amount of the input token to be swapped in, including all the fees.
- `amountOut`: The total amount of the output token to be swapped out after all the fees are paid.
- `totalFeesIn`: The amount of fees to pay in the swap input tokens.
- `totalFeesOut`: The amount of fees to pay in the swap output tokens.
- `numCrossedTicks`: The total number of ticks crossed after swapping in the given pool.

##### Execution

```python
def swap(poolId: PoolID,
    zeroToOne: bool,
    sqrtLimitPrice: Q96,
    amountSpecified: uint64,
    exactInput: bool,
    numCrossedTicks: uint32,
    currentHeight: uint32
    ) -> tuple[uint64, uint64, uint64, uint64, uint32]:

    feeTier = getFeeTier(poolId)
    poolSqrtPriceQ96 = bytesToQ96(pools[poolId].sqrtPrice)

    # check if the current price is not already beyond the limit price
    # if zeroToOne then price decreases after the swap, otherwise it increases
    if (zeroToOne and sqrtLimitPrice >= poolSqrtPriceQ96) or
        (not zeroToOne and sqrtLimitPrice <= poolSqrtPriceQ96):
        return (0,0,0,0,0)

    amountRemaining = amountSpecified
    amountTotalIn = 0
    amountTotalOut = 0
    totalFeesIn = 0
    totalFeesOut = 0
    if zeroToOne:
        tokenIn = getToken0Id(poolId)
        tokenOut = getToken1Id(poolId)
    else:
        tokenIn = getToken1Id(poolId)
        tokenOut = getToken0Id(poolId)

    # loop like Figure 4 in uniswap v3 whitepaper until amount is exhausted, price limit is reached, or max number of ticks crossed
    # we use != for prices because otherwise we need to account for approaching limit price from above and from below
    while(amountRemaining != 0 and poolSqrtPriceQ96 != sqrtLimitPrice):
        if numCrossedTicks > MAX_NUMBER_CROSSED_TICKS:
            raise ExceptionSwapCrossedTooManyTicks() # specific exception class for crossing too many ticks

        currentTick = priceToTick(poolSqrtPriceQ96)
        # get the price for the next tick
        if zeroToOne:
            nextTick = value of next initialized tick < currentTick
        else:
            nextTick = value of next initialized tick > currentTick

        if nextTick does not exist:
            # reached the end of liquidity
            break

        sqrtNextTickPriceQ96 = tickToPrice(nextTick)

        if zeroToOne and ticks(poolId, currentTick) exists and poolSqrtPriceQ96 ==  tickToPrice(currentTick):
            # need to cross the current tick from right to left
            crossTick(poolId + tickToBytes(currentTick), False, currentHeight)
            numCrossedTicks += 1

        if pools[poolId].liquidity == 0:
            # jump to the next initialized tick, hope that it has liquidity
            poolSqrtPriceQ96 = sqrtNextTickPriceQ96
            if not zeroToOne:
                crossTick(poolId + tickToBytes(nextTick), True, currentHeight)
                numCrossedTicks += 1
            continue

        # case of non-zero pool liquidity at the current price
        if (zeroToOne and sqrtNextTickPriceQ96 < sqrtLimitPrice) or
                (not zeroToOne and sqrtNextTickPriceQ96 > sqrtLimitPrice):
            sqrtTargetPrice = sqrtLimitPrice
        else:
            sqrtTargetPrice = sqrtNextTickPriceQ96

        # compute the remaining amount to swap in or to get out of the swap
        feeCoeffAmountAfter = div_96(Q96(feeTier/2), Q96(FEE_TIER_PARTITION - feeTier/2))
        feeCoeffAmountBefore = div_96(Q96(feeTier/2), Q96(FEE_TIER_PARTITION))
        if exactInput:
            # The amount remaining includes input fees, subtract them
            firstInFee = roundUp_96(mul_96(Q96(amountRemaining), feeCoeffAmountBefore))
            amountRemainingTemp = amountRemaining - firstInFee
        else:
            # The amount remaining does not include output fees, add them
            firstOutFee = roundUp_96(mul_96(Q96(amountRemaining),feeCoeffAmountAfter))
            amountRemainingTemp = amountRemaining + firstOutFee

        # compute the swap within price tick or price limit reached
        (poolSqrtPriceQ96, amountIn, amountOut) = swapWithin(poolSqrtPriceQ96, sqrtTargetPrice, pools[poolId].liquidity, amountRemainingTemp, exactInput)
        if poolSqrtPriceQ96 != sqrtTargetPrice:
            # swapped through all the remaining amount,
            # fee should be exactly as estimated before the swap to avoid dust leftovers
            if exactInput:
                feeIn = amountRemaining - amountIn
                feeOut = roundUp_96(mul_96(Q96(amountOut), feeCoeffAmountBefore))
            else:
                feeIn = roundUp_96(mul_96(Q96(amountIn), feeCoeffAmountAfter))
                feeOut = amountOut - amountRemaining
        else:
            # feeIn is feeTier/FEE_TIER_PARTITION fraction of the feeIn + amountIn
            # feeOut is feeTier/FEE_TIER_PARTITION of amountOut
            feeIn = roundUp_96(mul_96(Q96(amountIn), feeCoeffAmountAfter))
            feeOut = roundUp_96(mul_96(Q96(amountOut), feeCoeffAmountBefore))
        # update the remaining amount after a swap within a tick
        if exactInput:
            amountRemaining -= (amountIn + feeIn)
        else:
            amountRemaining -= (amountOut - feeOut)

        amountTotalOut += amountOut - feeOut
        amountTotalIn += amountIn + feeIn

        totalFeesIn += feeIn
        totalFeesOut += feeOut

        # compute liquidity providers fees
        validatorFeePartIn = tokenIn == TOKEN_ID_LSK ? VALIDATORS_LSK_INCENTIVE_PART : 0
        validatorFeePartOut = tokenOut == TOKEN_ID_LSK ? VALIDATORS_LSK_INCENTIVE_PART : 0
        liquidityFeeInQ96 = muldiv_96(Q96(feeIn), Q96(FEE_TIER_PARTITION - validatorFeePartIn), Q96(FEE_TIER_PARTITION))
        liquidityFeeOutQ96 = muldiv_96(Q96(feeOut), Q96(FEE_TIER_PARTITION - validatorFeePartOut), Q96(FEE_TIER_PARTITION))

        # update liquidity provider fees
        liquidityFee0Q96 = zeroToOne ? liquidityFeeInQ96 : liquidityFeeOutQ96
        liquidityFee1Q96 = zeroToOne ? liquidityFeeOutQ96 : liquidityFeeInQ96
        globalFeesQ96_0 = div_96(liquidityFee0Q96, Q96(pools[poolId].liquidity))
        globalFeesQ96_1 = div_96(liquidityFee1Q96, Q96(pools[poolId].liquidity))
        feeGrowthGlobal0Q96_0 = bytesToQ96(pools[poolId].feeGrowthGlobal0)
        pools[poolId].feeGrowthGlobal0 = q96ToBytes(add_96(feeGrowthGlobal0Q96_0, globalFeesQ96_0))
        feeGrowthGlobal1Q96_1 = bytesToQ96(pools[poolId].feeGrowthGlobal1)
        pools[poolId].feeGrowthGlobal1 = q96ToBytes(add_96(feeGrowthGlobal1Q96_1, globalFeesQ96_1))

        # if sqrtNextTickPriceQ96 was reached by increasing the price,
        # cross the next tick from left to right
        if poolSqrtPriceQ96 == sqrtNextTickPriceQ96 and not zeroToOne:
            crossTick(poolId + tickToBytes(nextTick), True, currentHeight)
            numCrossedTicks += 1

    # update the pool's state with the correct sqrtPrice
    pools[poolId].sqrtPrice = q96ToBytes(poolSqrtPriceQ96)

    return (amountTotalIn, amountTotalOut, totalFeesIn, totalFeesOut, numCrossedTicks)
```

where the functions `bytesToQ96`, `q96ToBytes`, `getFeeTier`, `priceToTick`, `tickToPrice` and all operations on Q96 numbers are defined in the [Introduce DEX module LIP][dexmodulelip].

#### swapWithin

The function computes and returns the result of swapping a given amount within a single tick.
It is an internal function of the DEX module, to be called inside the general swap loop algorithm for a given pool.
It is a pure function and it does not update the state when executed.

##### Parameters

- `sqrtCurrentPrice`: A Q96 number with the pool price before the swap within a single tick.
- `sqrtTargetPrice`: A Q96 number with the price limit not to be crossed by the swap within a single tick.
- `liquidity`: The liquidity amount at the given tick.
- `amountRemaining`: The amount of tokens to swap. It is the input amount if `exactInput == True` and the output amount otherwise.
- `exactInput`: A boolean indicating whether a swap is performed with exact input or with exact output.

##### Returns

- `sqrtUpdatedPrice`: A `Q96` number with the sqrt price after swapping the amount in/out, not to exceed the price target.
- `amountIn`: A `uint64` with the amount to be swapped in.
- `amountOut`: A `uint64` with the amount of a token to be swapped out.

##### Execution

```python
def swapWithin(
    sqrtCurrentPrice: Q96,
    sqrtTargetPrice: Q96,
    liquidity: uint64,
    amountRemaining: uint64,
    exactInput: bool
    ) -> tuple[Q96, uint64, uint64]:

    # check the direction of the trade (token1 <--> token 0)
    zeroToOne = sqrtCurrentPrice >= sqrtTargetPrice

    # check how much amount-in we need to get to sqrtTargetPrice from sqrtCurrentPrice
    if exactInput:
       if zeroToOne:
           amountIn = getAmount0Delta(sqrtCurrentPrice, sqrtTargetPrice, liquidity, True)
       else:
           amountIn = getAmount1Delta(sqrtCurrentPrice, sqrtTargetPrice, liquidity, True)
    else:
        if zeroToOne:
            amountOut = getAmount1Delta(sqrtCurrentPrice, sqrtTargetPrice, liquidity, False)
        else:
            amountOut = getAmount0Delta(sqrtCurrentPrice, sqrtTargetPrice, liquidity, False)

    # update sqrtUpdatedPrice accordingly

    if (exactInput and amountRemaining >= amountIn) or (not exactInput and amountRemaining >= amountOut):
        # enough tokens to change price to target price
        sqrtUpdatedPrice = sqrtTargetPrice
    else:
        # update price depending on whether amountRemaining represents token0 or token1
        # and whether it is amount to add to the pool (exactInput) or subtract from the pool.
        isToken0 = exactInput ? zeroToOne : !zeroToOne
        addsAmount = exactInput
        sqrtUpdatedPrice = computeNextPrice(sqrtCurrentPrice, liquidity, amountRemaining, isToken0, addsAmount)

    # calculate the actual input and output amounts. The updated price guarantees that amountRemaining
    # will not be exceeded, independently whether it is of token0 or token1
    if zeroToOne:
        amountIn = getAmount0Delta(sqrtCurrentPrice, sqrtUpdatedPrice, liquidity, True)
        amountOut = getAmount1Delta(sqrtCurrentPrice, sqrtUpdatedPrice, liquidity, False)
    else:
        amountIn = getAmount1Delta(sqrtCurrentPrice, sqrtUpdatedPrice, liquidity, True)
        amountOut = getAmount0Delta(sqrtCurrentPrice, sqrtUpdatedPrice, liquidity, False)

    return (sqrtUpdatedPrice, amountIn, amountOut)

```

where the functions `getAmount0Delta`, `getAmount1Delta` and `computeNextPrice` are defined in the [Introduce DEX module LIP][dexmodulelip].


#### crossTick

The function crosses a tick either from left to right or from right to left by updating all the necessary state store information.

```python
def crossTick(tickId: TickID, leftToRight: bool, currentHeight: uint32) -> None:
    poolId = tickId[:NUM_BYTES_POOL_ID]
    # update pool incentives at the current liquidity
    updatePoolIncentives(poolId, currentHeight)
    # update liquidity
    if leftToRight:
        pools[poolId].liquidity += ticks(poolId, tickId).liquidityNet
    else:
        pools[poolId].liquidity -= ticks(poolId, tickId).liquidityNet

    # update fee growth outside
    feeGrowthGlobal0Q96 = bytesToQ96(pools[poolId].feeGrowthGlobal0)
    feeGrowthOutside0Q96 = bytesToQ96(ticks(poolId, tickId).feeGrowthOutside0)
    ticks(poolId, tickId).feeGrowthOutside0 = q96ToBytes(sub_96(feeGrowthGlobal0Q96, feeGrowthOutside0Q96))
    feeGrowthGlobal1Q96 = bytesToQ96(pools[poolId].feeGrowthGlobal1)
    feeGrowthOutside1Q96 = bytesToQ96(ticks(poolId, tickId).feeGrowthOutside1)
    ticks(poolId, tickId).feeGrowthOutside1 = q96ToBytes(sub_96(feeGrowthGlobal1Q96, feeGrowthOutside1Q96))
    # update incentives per liquidity outside
    incentivesAccumulatorQ96 = bytesToQ96(pools[poolId].incentivesPerLiquidityAccumulator)
    incentivesOutsideQ96 = bytesToQ96(ticks(poolId, tickId).incentivesPerLiquidityOutside)
    ticks(poolId, tickId).incentivesPerLiquidityOutside = q96ToBytes(sub_96(incentivesAccumulatorQ96, incentivesOutsideQ96))
```

#### transferFeesFromPool

The function computes the correct amount of validator fees, transfers them from the given pool address to the validator incentives account, and locks the transferred tokens.

```python
def transferFeesFromPool(amount: uint64, id: TokenID, pool: PoolID) -> None:
    validatorFee = 0
    if id == TOKEN_ID_LSK:
        validatorFee = roundDown_96(muldiv_96(Q96(amount), Q96(VALIDATORS_LSK_INCENTIVE_PART), Q96(FEE_TIER_PARTITION)))

    if validatorFee > 0:
        transferFromPool(pool, ADDRESS_VALIDATOR_INCENTIVES, id, validatorFee)
        Token.lock(ADDRESS_VALIDATOR_INCENTIVES, MODULE_NAME_DEX, id, validatorFee)
```

#### raiseSwapException

The function raises an exception during a failed swap and it emits the necessary persistent event.

```python
def raiseSwapException(reason: uint32, tokenIn: TokenID, tokenOut: TokenID, senderAddress: Address) -> None:
    # should not happen
    emitPersistentEvent(
        module = MODULE_NAME_DEX,
        name = EVENT_NAME_SWAP_FAILED,
        data = {
            "senderAddress": senderAddress,
            "tokenIdIn": tokenIn,
            "tokenIdOut": tokenOut,
            "reason": reason
        },
        topics = [senderAddress]
    )
    raise Exception()
```

#### computeCurrentPrice

For a given pair of input and output tokens and a swap route between these tokens, the function returns the price of the input token in the terms of the output token along the route. E.g. price 4 means that 1 input token is worth 4 output tokens. Function raises an exception if the swap route is invalid.

```python
def computeCurrentPrice(tokenIn: TokenID, tokenOut: TokenID, swapRoute: list[PoolID]) -> Q96:
    price = Q96(1)
    tokenInPool = tokenIn
    for poolId in swapRoute:
        if poolId not in pools:
            raise Exception("Not a valid pool")

        if tokenInPool == getToken0Id(poolId):
            price = mul_96(price, bytesToQ96(pools[poolId].sqrtPrice))
            tokenInPool = getToken1Id(poolId)
        else if tokenInPool == getToken1Id(poolId):
            price = mul_96(price, inv_96(bytesToQ96(pools[poolId].sqrtPrice)))
            tokenInPool = getToken0Id(poolId)
        else:
            raise Exception("Incorrect swap path for price computation")
    if tokenInPool != tokenOut:
        raise Exception("Incorrect swap path for price computation")
    # convert from sqrt price
    return mul_96(price, price)
```

## Appendix

### DEX routing

#### Rationale

The commands “swap with exact input” and “swap with exact output” expect a swap route as an input parameter. The protocol logic does not specify how a trader may obtain such a route, and this could be a hard task for some pair of input and output tokens. The task of finding a swap route could be solved by a separate service called router (or DEX router). The router could be integrated with the online DEX user interface.

There are several advantages to the off-chain routing. Firstly, the on-chain computations are limited in memory and performance. Secondly, the algorithm of an off-chain service can be easily updated without a need for a hard fork.

Decentralized exchange offers a new utility case for LSK token, namely a main liquidity token for swaps, which simplifies the routing algorithm. Using LSK as a liquidity token also allows to keep swap slippage reasonably small.

#### Structure of DEX router

The DEX router can be implemented as part of the Service instance of the DEX sidechain. The DEX router is comprised of internal logic and storage, an interface for the UI to obtain routes for swaps, and it would interact with full nodes of the DEX sidechain to dry run transactions. More specifically, the DEX router needs to dry run the different swap commands and fetch the emitted events in order to estimate the outcome of different swaps.

Additionally the DEX router maintains a **pools graph**, where the set of vertices consists of all the token IDs contained in DEX pools and the set of edges consists of all pool IDs. Two vertices are connected by an edge in the pools graph if there is a pool with the two tokens corresponding to the vertices. This construction simplifies the routing, making it essentially a problem of finding a path in the graph.

We assume that the LSK token will be the most liquid and prominent token in the Lisk ecosystem. Therefore, for any other token we generally assume that a pool with the LSK token will be the most utilized by traders and most liquid due to the amount of liquidity provided. By focusing the incentives on LSK pools, we further want to strengthen this focus for traders and liquidity providers on LSK pools. This has the benefit that liquidity is not scattered around many different pools of tokens, but focused on the LSK pools which will allow for lower slippage and therefore better user experience for traders. It further greatly simplifies the router algorithm, as we assume that for any tokens the best route is either a direct swap or a swap via the LSK token.

#### Specification

LSK token acts as a liquidity token in the decentralized exchange, so the swap traffic is routed via it. The routing algorithm thus is governed by the following principles:

- If there exists a swap route for a given input and output tokens, the routing algorithm returns a non-empty route.
- The router prefers swap routes via the LSK token.
- When possible, the swap router returns the optimal route.

The sketch of the algorithm:

1. For a pair of input/output tokens try to find all paths directly via LSK in 2 hops. There might be several because of different pool fee tiers.
2. If no paths directly via LSK can be found, try to find all direct paths between input/output tokens.

Paths found in steps 1. or 2. are _regular_.

3. If there are regular paths, dry run all of them and return the most optimal.
4. If no regular paths are found, perform a breadth-first search on the pools graph to find a path between input and output tokens and return it without dry running. This path is called _exceptional_.

Next the algorithm is specified precisely.

##### Pools graph

Let `pools` denote the pools substore as an object.

Function `constructPoolsGraph` constructs an undirected multigraph and returns a tuple of vertices and edges of this pools graph.

```python
def constructPoolsGraph(pools) -> PoolsGraph:
    vertices = set()        # empty set, no duplicated elements allowed
    for poolId in pools:
        vertices.add(getToken0Id(poolId))
        vertices.add(getToken1Id(poolId))
    edges = pools.keys()
    return (vertices, edges)
```

Function `getAdjacent` takes a pools graph and a vertex as an input. It returns a list of all edges [incident](<https://en.wikipedia.org/wiki/Incidence_(graph)>) to the given vertex together with the second incident vertex.

```python
def getAdjacent(poolsGraph: PoolsGraph, vertex: TokenID) -> list[object]:
    result = []
    for edge in pools:
        if getToken0Id(edge) == vertex:
            adjacent = getToken1Id(edge)
            result.add({edge: edge, vertex: adjacent})
        else if getToken1Id(edge) == vertex:
            adjacent = getToken0Id(edge)
            result.add({edge: edge, vertex: adjacent})
    return result
```

##### Computing routes

Function `computeRegularRoute` returns a sequence of vertices in a regular route between the given vertices or empty array if such route does not exist. A regular route swaps via LSK if possible, or directly between two tokens otherwise. The return value is either empty, or has `tokenIn` as the first value and `tokenOut` as the last.

```python
def computeRegularRoute(tokenIn: TokenID, tokenOut: TokenID, poolsGraph: PoolsGraph) -> list[TokenID]:
    lskAdjacent = getAdjacent(poolsGraph, TOKEN_ID_LSK)
    if tokenIn in lskAdjacent and tokenOut in lskAdjacent:
        # swap via LSK
        return [tokenIn, TOKEN_ID_LSK, tokenOut]
    else if tokenOut in getAdjacent(poolsGraph, tokenIn):
        # direct swap
        return [tokenIn, tokenOut]
    else:
        # no regular routes found
        return []
```

Function `computeExceptionalRoute` returns a sequence of vertices in a route between two tokens in pools graph. The return value is either empty if the route does not exist, or has `tokenIn` as the first value and `tokenOut` as the last.

```python
def computeExceptionalRoute(tokenIn: TokenID, tokenOut: TokenID, poolsGraph: PoolsGraph) -> list[TokenID]:
    result = []
    routes = queue([])  # empty FIFO for breadth graph search
    routes.add({path: [], endVertex: tokenIn})
    visited = [tokenIn]
    # graph breadth search
    while routes not empty:
        route = routes.get()    # the next route in queue, pushes it out of the queue
        if route.endVertex == tokenOut:
            return route.path.add(tokenOut)
        for adjacent in getAdjacent(poolsGraph, route.endVertex):
            if adjacent.vertex not in visited:
                routes.add({path: copy(route.path).add(adjacent.edge),
                endVertex: adjacent.vertex})
                visited.add(adjacent.vertex)
    return []
```

Function `getOptimalSwapPool` finds a pool to swap given amount of tokens with the best value for user. If there is no direct pool then an exception is thrown. The function dry runs swap transactions and is thus computationally intense. The functions `dryRunSwapExactIn` and `dryRunSwapExactOut` are endpoints for off-chain services defined in the [Introduce DEX module LIP][dexmodulelip].

```python
def getOptimalSwapPool(tokenIn: TokenID, tokenOut: TokenID, amount: uint64,
                        exactIn: bool) -> tuple(PoolID, uint64):
    token0, token1 = tokenIn, tokenOut sorted lexicographically
    candidatePools = []
    for setting in dexGlobalData.poolCreationSettings:
        # all possible pools to swap token0 and token1
        potentialPoolId = token0 + token1 + setting.feeTier.to_bytes(4, byteorder='big')
        if potentialPoolId in pools:
            candidatePools.add(potentialPoolId)
    if candidatePools is empty:
        raise Exception("No pool swapping this pair of tokens")

    computedAmounts = []
    for pool in candidatePools:
        if exactIn:
            try:
                amountOut = dryRunSwapExactIn(tokenIn, amount, tokenOut, 0, [pool])[1]
                computedAmounts.add(amountOut)
            except:
                pass
        else:
            try:
                amountIn = dryRunSwapExactOut(tokenIn, MAX_UINT_64, tokenOut, amount, [pool])[0]
                computedAmounts.add(amountIn)
            except:
                pass
    if exactIn:
        let i be the index of a maximal element in computedAmounts
    else:
        let i be the index of a minimal element in computedAmounts
    return (candidatePools[i], computedAmounts[i])
```

##### Routing algorithm

Function `getRoute` implements the routing algorithm to find a swapping route between a given pair of tokens.

- `tokenIn`: The token ID of the input token.
- `tokenOut`: The token ID of the output token.
- `poolsGraph`: The tuple of vertices and edges of the pools graph.
- `amount`: The amount of tokens to be swapped. It could be either exact input amount or exact output amount depending on the `exactIn` flag.
- `exactIn`: If `True` then the route is found for the exact input command, otherwise for the exact output command.

```python
def getRoute(tokenIn: TokenID, tokenOut: TokenID, poolsGraph: PoolsGraph, amount: uint64,
                exactIn: bool) -> list[TokenID]:
    bestRoute = []
    regularRoute = computeRegularRoute(tokenIn, tokenOut, poolsGraph)
    # dry run regular routes if not empty
    if regularRoute not empty:
        if exactIn:
            poolTokenIn = tokenIn
            poolAmountIn = amount
            # find most optimal pools along the route
            for poolTokenOut in regularRoute[1:]:
                (optimalPool, poolAmountOut) = getOptimalSwapPool(
                    poolTokenIn, poolTokenOut, poolAmountIn, exactIn)
                bestRoute.add(optimalPool)
                poolTokenIn = poolTokenOut
                poolAmountIn = poolAmountOut
        else:
            poolTokenOut = tokenOut
            poolAmountOut = amount
            for poolTokenIn in reversed regularRoute[:-1]:
                (optimalPool, poolAmountIn) = getOptimalSwapPool(
                    poolTokenIn, poolTokenOut, poolAmountOut, exactIn)
                bestRoute.add(optimalPool)
                poolTokenOut = poolTokenIn
                poolAmountOut = poolAmountIn
            bestRoute = reverse bestRoute   # the output of router can be used directly as the swapRoute parameter
        return bestRoute

    exceptionalRoute = computeExceptionalRoute(tokenIn, tokenOut, poolsGraph)
    if route is empty or length(exceptionalRoute) > MAX_HOPS_SWAP:
        # no valid route is found
        return []
    # find pool sequence with the best liquidity at the current price
    poolTokenIn = tokenIn
    for poolTokenOut in exceptionalRoute[1:]:
        candidatePools = []
        token0, token1 = poolTokenIn, poolTokenOut sorted lexicographically
        for setting in dexGlobalData.poolCreationSettings:
            # all possible pools to swap tokenIn and tokenOut
            candidatePools.add(token0 + token1 + setting.feeTier.to_bytes(4, byteorder='big'))
        let bestPool be pool in candidatePools with maximal pools[pool].liquidity
        bestRoute.add(bestPool)
        poolTokenIn = poolTokenOut
    return bestRoute
```

[dexmodulelip]: https://github.com/LiskHQ/lips-staging/blob/main/proposals/lip-introduce_DEX_Module.md
[dexmodulelipappendix]: https://github.com/LiskHQ/lips-staging/blob/lip-Introduce_DEX_Module/proposals/lip-introduce_DEX_Module.md#Appendix
[uniswapv3whitepaper]: https://uniswap.org/whitepaper-v3.pdf
[tokenidlip]: https://github.com/LiskHQ/lips/blob/main/proposals/lip-0051.md#token-identification
[swapuniswap]: https://github.com/Uniswap/v3-core/blob/c05a0e2c8c08c460fb4d05cfdda30b3ad8deeaac/contracts/UniswapV3Pool.sol#L596
