```
LIP: <LIP number>
Title:  Introduce DEX module
Author: Iker Alustiza <iker@lightcurve.io>
        Jan Hackfeld <jan.hackfeld@lightcurve.io>
	       Sergey Shemyakov <sergey.shemyakov@lightcurve.io>
Type: Standards Track
Created: <YYYY-MM-DD>
Updated: <YYYY-MM-DD>
Required: LIP 40, LIP 51, LIP 52
```

## Abstract

In this LIP, we introduce the DEX module and specify its state store and other fundamental aspects of it.
Together with LIP "Swap" and LIP "LPs", this LIP completely specifies the DEX module.
The DEX module defines a decentralized exchange based on an automated market maker with a concentrated liquidity algorithm.


## Copyright

This LIP is licensed under the [Creative Commons Zero 1.0 Universal](https://creativecommons.org/publicdomain/zero/1.0/).


## Motivation

The Lisk platform is a multi-chain and multi-token ecosystem.
Each sidechain can potentially define (and create) one or many native tokens by using [the Token module][tokenLIP].
However, as of now there is no easy way to acquire these tokens for the basic user until they are listed in one of the typical centralized exchanges.
This represents a centralized point of failure in the accesibility of blockchain applications.
The DEX module provides the tools to facilitate the accessibility of Lisk ecosystem tokens by providing an state-of-art decentralized exchange (DEX) that can be utilized in any sidechain created with the Lisk SDK.

## Rationale

The DEX module is based on the concentrated liquidity algorithm introduced in the [Uniswap v3 whitepaper][uniswapv3whitepaper].

### Number Representation

The mathematical model of the concentrated liquidity algorithm utilizes non-integer numbers for the underlying computations.
For example, we think of the token price as of a fractional number.
This allows for much more precise swaps, even when the traded amounts are small.

In practice the fractional number representation has to be completely deterministic and transparent for implementation.
This requirement guarantees that all the correct implementations of the DEX protocol output the same binary messages and update the state in exactly the same way for the same inputs.
Thus we rely on fixed point arithmetic, which is specified in the [Appendix](#fixed-point-arithmetic) and is used in this document and other DEX module related LIPs.

In practice, we work with `Q96` unsigned numbers with 96 bits for integer and 96 bits for fractional parts (also see [the Wikipedia page about signed Q number format][Q_wiki]). Note that the intermediate results of arithmetic computations with `Q96` numbers may need more memory space, e.g. the implementation of `div_n(a, b) = (a << n) // b` needs to store the result of `a << n`.

### Fee tiers

Swap fees are a natural way to incentivize liquidity providers to add tokens into liquidity pools.
Following the [design of Uniswap v3][UniswapV3Fees], the protocol of the DEX module allows creation of pools only with fee tiers specified in protocol settings store and records fee tiers in units of parts-per-million of the swap amount (i.e. 0.0001%).
The pools with higher price volatility are recommended to have higher fee tiers to compensate for potential impermanent loss.

Each fee tier has a value of tick spacing associated with it.
Pools with higher fee tier assume greater price changes, so not all ticks in the pool are initialized to optimize total performance.
The following fee tiers and tick spacings are recommended: `{100, 2}, {500, 10}, {3000, 60}, {10000, 200}`.
See also the [Uniswap v3 implementation][UniswapV3Ticks].

## Specification

In this LIP, we specify the state store, internal auxiliary functions, internal math functions, protocol logic for other modules, and endpoints for other modules of the DEX module. The DEX module has name `MODULE_NAME_DEX`.

### Types

|Name            | Type              | Validation                           | Description                           |
|----------------|-------------------|--------------------------------------|---------------------------------------|
| Q96            | integer           | must be non-negative                 | Internal representation of a Q96 number|
| SqrtPrice            | Q96           | must be not less than `MIN_SQRT_RATIO` and not greater than `MAX_SQRT_RATIO`                 | A Q96 representation of a square root of price|
| TokenID            | bytes           | must have length `NUM_BYTES_TOKEN_ID`| Used to identify token types in the Lisk ecosystem as specified in LIP 0051 |
| PoolID            | bytes           | must have length `NUM_BYTES_POOL_ID`| Used to identify pools in Lisk DEX |
| PositionID     | bytes           | must have length `NUM_BYTES_POSITION_ID`| Used to identify positions in Lisk DEX |
| Address        | bytes | must have length `NUM_BYTES_ADDRESS` |  Account address in Lisk ecosystem |


#### Q96 type conversion

The Q96 numbers are stored as byte arrays of maximal length `MAX_NUM_BYTES_Q96`, however the internal functions of DEX module work with objects of type Q96 defined as in the table above. We define two functions to serialize and deserialize Q96 numbers to type bytes, `bytesToQ96` and `q96ToBytes`. The functions check that the length of byte representation does not exceed `MAX_NUM_BYTES_Q96`.

```python
    def bytesToQ96(numberBytes: bytes) -> Q96:
        if length(bytes) > MAX_NUM_BYTES_Q96:
            raise Exception()
        return big-endian decoding of bytes

    def q96ToBytes(numberQ96: Q96) -> bytes:
        result = big-endian encoding of numberQ96 as integer
        if length(result) > MAX_NUM_BYTES_Q96:
            raise Exception("Overflow when serializing a Q96 number")
        return result
```

Note that a Q96 representation of number 0 is serialized to empty bytes, and empty bytes are deserialized to a Q96 representation of 0.

### Notation and Constants

We define the following constants:

| Name                                          | Type   | Value            | Description                                           |
|-----------------------------------------------|--------|------------------|-------------------------------------------------------|
| **General Constants**                |        |                  |                            |                            
| `NUM_BYTES_ADDRESS`                | `uint32` | 20                     | The number of bytes of an address.  |     
| `MAX_NUM_BYTES_Q96`                    | `uint32` | 24                    | The maximal number of bytes of a serialized fractional number in Q96 format. |
| `ADDRESS_LIQUIDITY_PROVIDER_REWARDS_POOL`                    | bytes |  `SHA256(b"liquidityProviderRewardsPool")[:NUM_BYTES_ADDRESS]`  | The address of the liquidity provider rewards pool.  |
| `ADDRESS_TRADER_REWARDS_POOL`                    | bytes |  `SHA256(b"traderRewardsPool")[:NUM_BYTES_ADDRESS]`  | The address of the trader rewards pool.  |
| `ADDRESS_VALIDATOR_REWARDS_POOL`                    | bytes |  `SHA256(b"validatorRewardsPool")[:20]`  | The address of the validator rewards pool.  |
| `MAX_UINT_32`                            |    `uint32`    | 4294967295 | Maximal value of a `uint32` number   |
| `MAX_UINT_64`         |       `uint64`    | 18446744073709551615      |   Maximal value of a `uint64` number  |
| **DEX Module Constants**                |        |                  |                            |                            
| `MODULE_NAME_DEX`                | string | "dex"                     | Name of the DEX module.  |     
| `NUM_BYTES_POOL_ID`                | `uint32` | 20                     | The number of bytes of a pool ID.  |    
| `NUM_BYTES_TICK_ID`                | `uint32` | 24                     | The number of bytes of a price tick ID.  |      
| `NUM_BYTES_POSITION_ID`                | `uint32` | 28                     | The number of bytes of a position ID.  |     
| `MAX_NUMBER_CROSSED_TICKS`                | `uint32` | TBA                     | Maximum number of price ticks to be crossed by a single swap.  |
| `MAX_HOPS_SWAP`                | `uint32` | TBA                     | Maximum number of different pools that a complete swap can interact with.  |   
| `MAX_NUM_POSITIONS_FEE_COLLECTION`  | `uint32` | TBD            | The maximum number of positions for which it is possible to collect fees in one transaction.                                 |
| `TOKEN_ID_FEE_DEX`        | bytes | TBD    | The ID of the token used for fees. This defines the type of token in which the additional fees for pool creation and position creation are paid.         |
| `TOKEN_ID_REWARDS`        | bytes | TBD    | The token ID of the token used for liquidity provider, trader and validator incentives.         |
| `POOL_CREATION_FEE`       | `uint64` | TBD    | This amount of tokens is transferred to the protocol fee account when creating a new pool.|
| `POSITION_CREATION_FEE`   | `uint64` | TBD    | This amount of tokens is transferred to the protocol fee account when creating a new position.|       
| `TARGET_BALANCE_TRADER_REWARDS_POOL`    | `uint64`  | TBD       |      The goal balance of the trader rewards pool, is used to compute trader incentives.      |                   
| **Token Module Constants**                |        |                  |                           |                            
| `CHAIN_ID_ALIAS_NATIVE`              |  bytes |   0x0000           |    `chainID` value of a native token.     |      
| `NUM_BYTES_TOKEN_ID`                | `uint32` | 8                     | The number of bytes of a token ID.  |                      
| **DEX Module Store**                    |        |                  | |                                                       |
| `SUBSTORE_PREFIX_STATE`                    | bytes  | 0x0000           | Substore prefix of the DEX global state substore. |                                                       |
| `SUBSTORE_PREFIX_POOL`                    | bytes  | 0x4000           | Substore prefix of the pools substore. |
| `SUBSTORE_PREFIX_PRICE_TICK`                    | bytes  | 0x8000           | Substore prefix of the price tick substore. |
| `SUBSTORE_PREFIX_POSITION`                    | bytes  | 0xc000           | Substore prefix of the positions substore. |
| `SUBSTORE_PREFIX_SETTINGS`                    | bytes  | 0xe000           | Substore prefix of the protocol settings substore. |
| **DEX Module Command Names**              |        |                  |                                                |
| `COMMAND_SWAP_EXACT_INPUT`                    | string | "swapExactInput"                | Command name of swap exact input command. |
| `COMMAND_SWAP_EXACT_OUTPUT`                   | string | "swapExactOutput"                | Command name of swap exact output command. |
| `COMMAND_SWAP_WITH_PRICE_LIMIT`               | string | "swapPriceLimit"                | Command name of swap with price limit command. |
| `COMMAND_CREATE_POOL`                    | string | "createPool"                | Command name of create pool command. |
| `COMMAND_CREATE_POSITION`                    | string | "createPosition"               | Command name of create position command. |
| `COMMAND_ADD_LIQUIDITY`                    | string | "addLiquidity"                | Command name of add liquidity command. |
| `COMMAND_REMOVE_LIQUIDITY`                 | string | "removeLiquidity"                | Command name of remove liquidity command. |
| `COMMAND_COLLECT_FEES`                 | string | "collectFees"                | Command name of collect fees command. |
| **DEX Module event names**              |        |                  |                                                |
| `EVENT_NAME_AMOUNT_BELOW_MIN`                 | string |     "amountBelowMin"     | Event name of the `AmountBelowMin` event. |
| `EVENT_NAME_FEES_INCENTIVES_COLLECTED`                 | string |     "feesIncentivesCollected"       | Event name of the `FeesIncentivesCollected` event. |
| `EVENT_NAME_POOL_CREATED`                 | string |        "poolCreated"        | Event name of the `PoolCreated` event. |
| `EVENT_NAME_POOL_CREATION_FAILED`                 | string |       "poolCreationFailed"          | Event name of the `PoolCreationFailed` event. |
| `EVENT_NAME_POSITION_CREATED`                 | string |      "positionCreated"        | Event name of the `PositionCreated` event. |
| `EVENT_NAME_POSITION_CREATION_FAILED`                 | string |   "positionCreationFailed"   | Event name ID of the `PositionCreationFailed` event. |
| `EVENT_NAME_POSITION_UDPATED`                 | string |  "positionUpdated"  | Event name ID of the `PositionUpdated` event. |
| `EVENT_NAME_POSITION_UDPATE_FAILED`                 | string |   "positionUpdateFailed"   | Event name of the `PositionUpdateFailed` event. |
| `EVENT_NAME_SWAPED`                 | string |     "swapped"  | Event name of the `Swapped` event. |
| `EVENT_NAME_SWAP_FAILED`                 | string |       "swapFailed"      | Event name of the `SwapFailed` event. |
| **Math Constants**                         |        |                  |                                              |    
| `MIN_TICK`                                  | `sint32`  | -887272     | The minimum possible tick value as a sint32. |
| `MAX_TICK`                              | `sint32` | 887272       | The maximum possible tick value as a sint32. |
| `MIN_SQRT_RATIO`                              | `uint256` | 4295128738 Todo: check with devs | The minimum possible price value in the `Q96` representation. |
| `MAX_SQRT_RATIO`                              | `uint256` | 1461446703529909599612049957420313862569572983184 Todo: check with devs | The maximum possible price value in the `Q96` representation. |
| `PRICE_VALUE_FOR_BIT_POSITION_IN_Q96`   | `uint256` |  TBA     | Array of `uint256` values with the pre-computed values of price for certain values of `tickValue` in the `Q96` representation. |
| `PRICE_VALUE_FOR_TICK_1`   | `uint256` |  TBA     | The pre-computed value for `tickToPrice(1)` in the `Q96` representation. |

#### Logic from Other Modules

Calling a function `fct` implemented in another module `module` is represented by `module.fct(required inputs)`.

#### Prices and sqrt prices

For a pool with the two tokens `token0` and `token1`, we express prices as the price of `token0` in terms of the other token `token1`.
For instance, for a pool with tokens LSK and USD a price of 4 means that the price of 1 LSK is 4 USD.
In order to avoid computing square roots in several parts of the DEX module logic, we actually do not use the price, but instead the square root of the price, denoted by sqrtPrice.
In the previous example, the square root price would be sqrt(4) = 2.

### DEX Module Store

The key-value pairs in the module store are organized as follows:

#### DEX Global State substore

##### Substore Prefix, Store Key, and Store Value

* The substore prefix is set to `SUBSTORE_PREFIX_STATE`.
* The store key is empty bytes.
* The store value is the serialization of an object following the JSON schema `dexGlobalStateSchema` presented below.
* Notation: We let `dexGlobalState` denote the object stored in the DEX module store with substore prefix `SUBSTORE_PREFIX_STATE` and empty key.

##### JSON Schema

```java
dexGlobalStateSchema = {
    "type": "object",
    "required": [
        "positionCounter",
        "collectableLSKFees"
    ],
    "properties": {
        "positionCounter": {
            "dataType": "uint64",
            "fieldNumber": 1
        },
        "collectableLSKFees": {
            "dataType": "uint64",
            "fieldNumber": 2
        }
    }
}
```

##### Properties

In this section, we describe the properties of DEX global state substore.

* `positionCounter`: denotes the total number of positions currently in all DEX pools.
* `collectableLSKFees`: a `uint64` number with the total fees in LSK token payed by traders to the DEX module but not yet collected by liquidity providers.

#### Pools Substore

##### Substore Prefix, Store Key, and Store Value

* The substore prefix is set to `SUBSTORE_PREFIX_POOLS`.
* The store key is a byte array of length `NUM_BYTES_POOL_ID` representing a pool ID.
* Each store value is the serialization of an object following the JSON schema `poolsSchema` presented below.
* Notation: We let `pools(poolId)` denote the object stored in the DEX module store with substore prefix `SUBSTORE_PREFIX_POOLS` and key equal to `poolId`.

The ID of the pool, `poolId` is defined as the concatenation of

* `tokenId0`, the byte array of length `NUM_BYTES_TOKEN_ID` with ID of `token0`.
* `tokenId1`, the byte array of length `NUM_BYTES_TOKEN_ID` with ID of `token1`.
* `feeTier.to_bytes(4, byteorder = 'big', signed = False)` where `feeTier` is the fee tier of the pool given in units of parts-per-million of the swap amount.

It is a requirement that `tokenId0` precedes `tokenId1` in the lexicographical ordering of byte arrays.

##### JSON Schema

```java
poolsSchema = {
    "type": "object",
    "required": [
        "liquidity",
        "sqrtPrice",
        "feeGrowthGlobal0",
        "feeGrowthGlobal1",
        "tickSpacing"
    ],
    "properties": {
        "liquidity": {
            "dataType": "uint64",
            "fieldNumber": 1
        },
        "sqrtPrice": {
            "dataType": "bytes",
            "maxLength": MAX_NUM_BYTES_Q96,
            "fieldNumber": 2
        },
        "feeGrowthGlobal0": {
            "dataType": "bytes",
            "maxLength": MAX_NUM_BYTES_Q96,
            "fieldNumber": 3
        },
        "feeGrowthGlobal1": {
            "dataType": "bytes",
            "maxLength": MAX_NUM_BYTES_Q96,
            "fieldNumber": 4
        },
        "tickSpacing": {
            "dataType": "uint32",
            "fieldNumber": 5
        }
    }
}
```

##### Properties

In this section, we describe the properties of pools substore.

* `liquidity`: A `uint64` integer representing the virtual liquidity of the pool.
* `sqrtPrice`: A `byte` array with the square root of the current price in `Q96` format.
* `feeGrowthGlobal0`: A `byte` array with the total amount of fees in `token0` that have been collected per unit of virtual liquidity  in `Q96` format.
* `feeGrowthGlobal1`: A `byte` array with the total amount of fees in `token1` that have been collected per unit of virtual liquidity in `Q96` format.
* `tickSpacing`: A `uint32` integer providing the tick spacing of the pool. A tick with index `i` can only be initialized in this pool if `i % tickSpacing == 0` holds.

#### Price tick substore

##### Tick value byte serialization

Swap command functionality needs to easily access tick substore keys in a sequential order, e.g. accessing the next or previous initialized tick. Tick values are signed integers, so we need a custom byte serialization logic to make sure that sequential ticks will be serialized in the same byte lexicographical order. The next function serializes ticks are unsigned integers shifted by `2**31`.

```python
def tickToBytes(tickValue: int32) -> bytes:
    if tickValue > MAX_TICK or tickValue < MIN_TICK:
        raise Exception()
    return (tickValue + 2**31).to_bytes(4, byteorder='big')

def bytesToTick(serializedTick: bytes) -> int32:
    if length(serializedTick) != 4:
        raise Exception()
    tickValue = int.from_bytes(serializedTick, byteorder='big', signed=false) - 2**31
    if tickValue > MAX_TICK or tickValue < MIN_TICK:
        raise Exception()
    return tickValue
```

Note that everywhere else the price ticks are serialized as usual `sint32` numbers.

##### Substore Prefix, Store Key, and Store Value

* The substore prefix is set to `SUBSTORE_PREFIX_PRICE_TICK`.
* Each store key is a byte array `poolId + tickToBytes(tickValue)` of length `NUM_BYTES_TICK_ID`, for a byte array `poolId` of length `NUM_BYTES_POOL_ID` presenting a pool ID and a tick value `tickValue`.
* Each store value is the serialization of an object following the JSON schema `priceTickSchema` presented below.
* We denote `tick(poolId, tickValue)` to be the price tick substore value with key `poolId + tickToBytes(tickValue)`.

##### JSON Schema

```java
priceTickSchema = {
    "type": "object",
    "required": [
        "liquidityNet",
        "liquidityGross",
        "feeGrowthOutside0",
        "feeGrowthOutside1"
    ],
    "properties": {
        "liquidityNet": {
            "dataType": "sint64",
            "fieldNumber": 1
        },
        "liquidityGross": {
            "dataType": "uint64",
            "fieldNumber": 2
        },
        "feeGrowthOutside0": {
            "dataType": "bytes",
            "maxLength": MAX_NUM_BYTES_Q96,
            "fieldNumber": 3
        },
        "feeGrowthOutside1": {
            "dataType": "bytes",
            "maxLength": MAX_NUM_BYTES_Q96,
            "fieldNumber": 4
        }
    }
}
```

##### Properties

In this section, we describe the properties of price tick substore.

* `liquidityNet`: A `sint64` integer representing the amount of liquidity added (or, if negative, removed) when the tick is crossed going left to right.
* `liquidityGross`: A `uint64` integer representing the gross tally of liquidity pointing to the tick.
* `feeGrowthOutside0`: A `byte` array with the fee for `token0` accumulated “outside” the tick  in `Q96` format.
* `feeGrowthOutside1`: A `byte` array with the fee for `token1` accumulated “outside” the tick  in `Q96` format.

#### Positions substore

##### Substore Prefix, Store Key, and Store Value

* The substore prefix is set to `SUBSTORE_PREFIX_POSITION`.
* Each store key is a byte array of length `NUM_BYTES_POSITION_ID` representing a position ID.
* Each store value is the serialization of an object following the JSON schema `positionSchema` presented below.
* Notation: We let `positions(positionId)` denote the object stored in the DEX module store with substore prefix `SUBSTORE_PREFIX_POSITION` and key equal to `positionId`.

A position ID is a byte array of length `NUM_BYTES_POSITION_ID` given by `poolId + index.to_bytes(8, byteorder = 'big', signed = False)`, where `poolId` is the pool ID of the corresponding pool and `index` the index of the position. Indices are assigned to the positions sequentially among all the positions in all pools.

##### JSON Schema

```java
positionSchema = {
    "type": "object",
    "required": [
        "tickLower",
        "tickUpper",
        "liquidity",
        "feeGrowthInsideLast0",
        "feeGrowthInsideLast1",
        "ownerAddress"
    ],
    "properties": {
        "tickLower": {
            "dataType": "sint32",
            "fieldNumber": 1
        },
        "tickUpper": {
            "dataType": "sint32",
            "fieldNumber": 2
        },
        "liquidity": {
            "dataType": "uint64",
            "fieldNumber": 3
        },
        "feeGrowthInsideLast0": {
            "dataType": "bytes",
            "maxLength": MAX_NUM_BYTES_Q96,
            "fieldNumber": 4
        },
        "feeGrowthInsideLast1": {
            "dataType": "bytes",
            "maxLength": MAX_NUM_BYTES_Q96,
            "fieldNumber": 5
        },
        "ownerAddress": {
            "dataType": "bytes",
            "length": NUM_BYTES_ADDRESS,
            "fieldNumber": 6
        }
    }
}
```

##### Properties

In this section, we describe the properties of positions substore.

* `tickLower`: A `sint32` tick value with the lower price tick of the position. It is assigned during position creation and cannot be updated.
* `tickUpper`: A `sint32` tick value with the upper price tick of the position. It is assigned during position creation and cannot be updated.
* `liquidity`: A `uint64` integer representing the virtual liquidity added by the position.
* `feeGrowthInsideLast0`: A `byte` array with the fees per unit of virtual liquidity for `token0` since the last time the fees were collected, in `Q96` format.
* `feeGrowthInsideLast1`: A `byte` array with the fees per unit of virtual liquidity for `token1` since the last time the fees were collected, in `Q96` format.
* `ownerAddress`: A `byte` array with the address of the owner of the position.

#### Protocol settings substore

##### Substore Prefix, Store Key, and Store Value

* The substore prefix is `SUBSTORE_PREFIX_SETTINGS`.
* The store key is empty bytes.
* The store value is the serialization of an object following the JSON schema `settingsSchema`.
* Notation: We let `settings` denote the object stored in the DEX module store with substore prefix `SUBSTORE_PREFIX_SETTINGS` and key equal to empty bytes.

```java
settingsSchema = {
    "type": "object",
    "required": [
    "protocolFeeAddress",
    "protocolFeePart",
    "validatorsLSKRewardsPart",
    "poolCreationSettings"
    ],
    "properties": {
        "protocolFeeAddress": {
            "dataType": "bytes",
            "length": NUM_BYTES_ADDRESS,
            "fieldNumber": 1
        },
        "protocolFeePart": {
            "dataType": "uint32",
            "fieldNumber": 2
        },
        "validatorsLSKRewardsPart": {
            "dataType": "uint32",
            "fieldNumber": 3
        },
        "poolCreationSettings": {
            "type": "array",
            "fieldNumber": 4,
            "items": {
                "type": "object",
                "required": ["feeTier", "tickSpacing"],
                "properties": {
                    "feeTier": {
                        "dataType": "uint32",
                        "fieldNumber": 1
                    },
                    "tickSpacing": {
                        "dataType": "uint32",
                        "fieldNumber": 2
                    }
                }
            }
        }
    }
}
```

##### Properties

In this section, we describe the properties of protocol settings substore.

* `protocolFeeAddress`: A byte array with the address to which the protocol fee is assigned.
* `protocolFeePart `: A `uint32` with a value between 0 and 10^6 (inclusive) denoting the protocol fee in parts-per-million of the swap amount.
For instance, `protocolFeePart == 100000` means that 10 % of the fees collected in a trade are assigned to `protocolFeeAddress`.
* `validatorsLSKRewardsPart`: A `uint32` with a value between 0 and 10^6 (inclusive) denoting the portion of LSK fees that are paid to the validators, in parts-per-million.
* `poolCreationSettings`: An array of allowed pool settings in the DEX protocol, each entry records a fee tier together with tick spacing. A `uint32` number `feeTier` between 0 and 10^6 (inclusive) represents a fee tier in units of parts-per-million of the swap amount, i.e. any swap in a pool with fee tier 10000 pays `1%` of the amount as fee. If tick spacing of a pool is given by a `uint32` number `tickSpacing` then only ticks with indices `i` satisfying `i % tickSpacing == 0` can be initialized in this pool.

### Internal Auxiliary Functions

In this section we specify certain internal auxiliary functions that are used in both DEX user interactions and hence, in several commands of the DEX module.

#### poolIdToAddress

This function computes the token store address of a pool given the pool ID.

```python
def poolIdToAddress(poolId: PoolID) -> Address:
    return SHA256(poolId)[:NUM_BYTES_ADDRESS]
```

#### getToken0Id

This function returns the first token ID of a given pool.

```python
def getToken0Id(poolId: PoolID) -> TokenID:
    return poolId[:NUM_BYTES_TOKEN_ID]
```

#### getToken1Id

This function returns the second token ID of a given pool.

```python
def getToken1Id(poolId: PoolID) -> TokenID:
    return poolId[NUM_BYTES_TOKEN_ID : 2*NUM_BYTES_TOKEN_ID]
```

#### getFeeTier

This function returns the fee tier of a given pool given in units of parts-per-million of the swap amount.

```python
def getFeeTier(poolId: PoolID) -> uint32:
    # deserialize the last 4 bytes of poolId
    return int.from_bytes(poolId[-4:], byteorder='big', signed = False)
```

#### getPositionIndex

This function returns the index of a given position.

```python
def getPositionIndex(positionId: PositionID) -> uint64:
    # deserialize the last 8 bytes of positionId
    return int.from_bytes(positionId[-8:], byteorder = 'big', signed = False)
```

#### transferToPool

This function transfers an amount of a certain token from a user account to the balance of an existing pool.

```python
def transferToPool(
    senderAddress: Address,
    poolId: PoolID,
    tokenId: TokenID,
    amount: uint64
    ) -> None:
    poolAddress = poolIdToAddress(poolId)
    Token.transfer(senderAddress, poolAddress, tokenID, amount)
    Token.lock(poolAddress, MODULE_NAME_DEX, tokenID, amount)
```  
#### transferFromPool

This function transfers an amount of a certain token from the balance of an existing pool to a user account.

```python
def transferFromPool(
    poolId: PoolID,
    recipientAddress: Address,
    tokenId: TokenID,
    amount: uint64
    ) -> None:
    poolAddress = poolIdToAddress(poolId)
    Token.unlock(poolAddress, MODULE_NAME_DEX, tokenID, amount)
    Token.transfer(poolAddress, recipientAddress, tokenID, amount)
```

#### transferPoolToPool

This function transfers an amount of a certain token from the balance of an existing pool to the balance of another existing pool. It is used for multihop swaps.

##### Parameters

* `poolIdSend`: The pool ID of the sending pool.
* `poolIdReceive`: The pool ID of the receiving pool.
* `tokenId`: The token IDs of the tokens to send.
* `amount`: A `uint64` with the amount of a token to be transferred.

##### Execution

```python
def transferPoolToPool(
    poolIdSend: PoolID,
    poolIdReceive: PoolID,
    tokenId: TokenID,
    amount: uint64
    ) -> None:
    poolAddressSend = poolIdToAddress(poolIdSend)
    poolAddressReceive = poolIdToAddress(poolIdReceive)
    Token.unlock(poolAddressSend, MODULE_NAME_DEX, tokenID, amount)
    Token.transfer(poolAddressSend, poolAddressReceive, tokenID, amount)
    Token.lock(poolAddressReceive, MODULE_NAME_DEX, tokenID, amount)
```

#### transferToProtocolFeeAccount

This function transfers an amount of a certain token from a user account to the protocol fee address.

```python
def transferToProtocolFeeAccount(
    senderAddress: Address,
    tokenId: TokenID,
    amount: uint64
    ) -> None:
    Token.transfer(senderAddress, settings.protocolFeeAddress, tokenId, amount)
```   

### Internal Math Functions

In this section we specify certain internal mathematical functions that are used in both DEX user interactions and hence, in several commands of the DEX module.
We also define the constant `LOG_MAX_TICK = ⌊log(MAX_TICK, 2)⌋ = 19`, which will be used throughout the section.

#### tickToPrice

This function computes the sqrt price given a tick value as `sqrt(1.0001)^tickValue`.
This function is the equivalent to the [getSqrtRatioAtTick function][getSqrtRatioAtTick] in Uniswap V3.
See also [their Typescript version][getSqrtRatioAtTickTypescript].

```python
def tickToPrice(tickValue: int32) -> SqrtPrice:
    # this pseudo code is a simplified version of
    # https://github.com/Uniswap/v3-core/blob/c05a0e2c8c/contracts/libraries/TickMath.sol#L23

    if tickValue < MIN_TICK or tickValue > MAX_TICK:
        raise Exception()

    absTick = abs(tickValue)
    sqrtPrice = Q96(1)
    # iterate the bit representation of absTick from the highest to the lowest
    for i in range(19, -1, -1):
        if ((absTick >> i) & 1) == 1:
            sqrtPriceAtBit = PRICE_VALUE_FOR_BIT_POSITION_IN_Q96[i]
            sqrtPrice = mul_96(sqrtPrice, sqrtPriceAtBit)

    if tickValue > 0:
        sqrtPrice = inv_96(sqrtPrice)

    return sqrtPrice
```

where the value for the array of constants `PRICE_VALUE_FOR_BIT_POSITION_IN_Q96` are pre-computed as `Q96(1.0001^((-2^i)/2))` for `i` in `[0, LOG_MAX_TICK]`. In the notation `log(x, b)`, `b` signifies the base of the logarithm.


#### priceToTick

This function computes the tick value of the greatest price tick such that `tickToPrice(tick) <= sqrtPrice`.
This function is the equivalent to the [getTickAtSqrtRatio function][getTickAtSqrtRatio] in Uniswap V3.
See also [their Typescript version][getTickAtSqrtRatioTypescript].

```python
def priceToTick(sqrtPrice: SqrtPrice) -> int32:
    invertedPrice = False
    sqrtPriceOriginal = sqrtPrice
    if sqrtPrice >= PRICE_VALUE_FOR_TICK_1:
        sqrtPrice = inv_96(sqrtPrice)
        invertedPrice = True

    tickValue = 0
    tempPrice = Q96(1)
    for i in range(LOG_MAX_TICK, -1, -1):
        sqrtPriceAtBit = PRICE_VALUE_FOR_BIT_POSITION_IN_Q96[i]
        newPrice = mul_96(tempPrice, sqrtPriceAtBit)
        if sqrtPrice <= newPrice:
            tickValue += 1 << i
            tempPrice = newPrice

    # at this point sqrtPrice <= tempPrice and tempPrice = price at tick tickValue
    if not invertedPrice:
        # tickToPrice(-tickValue) <= sqrtPriceOriginal as required
        tickValue = -tickValue
    # need this check here because otherwise priceToTick(tickToPrice(tick))
    # might be not equal to tick due to e.g. Q96 rounding.
    if tickToPrice(tickValue) > sqrtPriceOriginal:
        tickValue -= 1

    return tickValue
```

#### getAmount0Delta

This function computes the delta of amount of `token0` between two prices according to the formula `liquidity / sqrt(lowerPrice) - liquidity / sqrt(upperPrice)`.
This function is the equivalent to the [getAmount0Delta function][getAmount0Delta] in Uniswap V3.
See also [their Typescript version][getAmount0DeltaTypescript].

##### Parameters

* `sqrtPrice1`: A Q96 number with the first sqrt price.
* `sqrtPrice2`: A Q96 number with the second sqrt price.
* `liquidity`: A `uint64` with the current virtual liquidity.
* `roundUp`:  A `bool` indicating if the output should be round up, when `True`, or down, when `False`.

##### Returns

The amount of `token0` required to move the price between the two input prices with the given liquidity.

##### Execution

```python
def getAmount0Delta(
    sqrtPrice1: SqrtPrice,
    sqrtPrice2: SqrtPrice,
    liquidity: uint64,
    roundUp: bool
    ) -> uint64:
    if liquidity == 0:
        raise Exception()

    # set prices from small to big
    if sqrtPrice1 > sqrtPrice2:
        sqrtPrice1, sqrtPrice2 = sqrtPrice2, sqrtPrice1

    num1 = Q96(liquidity)
    num2 = sub_96(sqrtPrice2, sqrtPrice1)

    # eq 6.16 in the Uniswap v3 whitepaper
    amount0 = div_96(muldiv_96(num1, num2, sqrtPrice2), sqrtPrice1)

    if not roundUp:
        amount0 = Q_96_ToInt(amount0)
    else:
        amount0 = Q_96_ToIntRoundUp(amount0)
    return amount0    
```

#### getAmount1Delta

This function computes the delta of amount of `token1` between two prices according to the formula `liquidity * (sqrt(upperPrice) - sqrt(lowerPrice))`.
This function is the equivalent to the [getAmount1Delta function][getAmount0Delta] in Uniswap V3.
See also [their Typescript version][getAmount1DeltaTypescript].

##### Parameters

* `sqrtPrice1`: A Q96 number with the first sqrt price.
* `sqrtPrice2`: A Q96 number with the second sqrt price.
* `liquidity`: A `uint64` with the current virtual liquidity.
* `roundUp`:  A `bool` indicating if the output should be round up, when `True`, or down, when `False`.

##### Returns

The amount of `token1` required to move the price between the two input prices with the given liquidity.

##### Execution

```python
def getAmount1Delta(
    sqrtPrice1: SqrtPrice,
    sqrtPrice2: SqrtPrice,
    liquidity: uint64,
    roundUp: bool
    ) -> uint64:
    if liquidity == 0:
        raise Exception()

    # set prices from small to big
    if sqrtPrice1 > sqrtPrice2:
        sqrtPrice1, sqrtPrice2 = sqrtPrice2, sqrtPrice1

    liquidity = Q96(liquidity)
    # eq 6.14 in the Uniswap v3 whitepaper
    amount1 = mul_96(liquidity, sub_96(sqrtPrice2, sqrtPrice1))

    if not roundUp:
        amount1 = Q_96_ToInt(amount1)
    else:
        amount1 = Q_96_ToIntRoundUp(amount1)
    return amount1
```

#### computeNextPrice

This function computes the next sqrt price given a delta amount of a token and depending on the swap direction.
This function is the equivalent to the functions [getNextSqrtPriceFromAmount0RoundingUp][getNextSqrtPriceFromAmount0RoundingUp] and [getNextSqrtPriceFromAmount1RoundingDown][getNextSqrtPriceFromAmount1RoundingDown] in Uniswap V3.
See also their Typescript version for the [former][getNextSqrtPriceFromAmount0RoundingUpTypescript] and for the [latter][getNextSqrtPriceFromAmount1RoundingDownTypescrypt].

##### Parameters
* `sqrtPrice`: A Q96 number with the sqrt price.
* `liquidity`: A `uint64` with the current virtual liquidity.
* `amount`: A `uint64` with the delta amount of a token.
* `isToken0`: A `bool` setting the input token. If `True`, `amount` represents an amount of `token0`. Otherwise, `amount` represents an amount of `token1`.
* `addsAmount`: A `bool` whether to add (`True`), or remove (`False`), the input `amount`.

##### Returns

The sqrt price after adding the token amount.

##### Execution

```python
def computeNextPrice(
    sqrtPrice: SqrtPrice,
    liquidity: uint64,
    amount: uint64,
    isToken0: bool,
    addsAmount: bool
    ) -> SqrtPrice:
    if liquidity == 0:
        raise Exception()

    liquidity = Q96(liquidity)
    amount = Q96(amount)   
    if isToken0:        
        # eq 6.15 in the Uniswap v3 whitepaper
        if addsAmount:
            denom = add_96(mul_96(amount, sqrtPrice), liquidity)

        else:
            denom = sub_96(liquidity, mul_96(amount, sqrtPrice))

        nextSqrtPrice = muldiv_96_RoundUp(liquidity, sqrtPrice, denom)

    else:
        # eq 6.13 in the Uniswap v3 whitepaper
        if addsAmount:        
            nextSqrtPrice = add_96(sqrtPrice, div_96(amount, liquidity))      
        else:
            nextSqrtPrice = sub_96(sqrtPrice, muldiv_96_RoundUp(amount, Q96(1), liquidity))

    return nextSqrtPrice
```

### Protocol Logic for Other Modules

#### updateProtocolFee

Function sets the protocol fee address and the protocol fee.

```python
def updateProtocolFee(protocolFeeAddress: Address, protocolFeePart: uint32) -> None:
    if protocolFeePart > 1000000:
        raise Exception("Fee part can not be greater than 100%")

    settings.protocolFeeAddress = protocolFeeAddress
    settings.protocolFeePart = protocolFeePart
```

#### updateValidatorsLSKRewardsPart

Function sets the validator LSK rewards fee and the reward pool address.

```python
def updateValidatorsLSKRewardsPart(validatorsLSKRewardsPart: uint32) -> None:
    if validatorsLSKRewardsPart > 1000000:
        raise Exception()

    settings.validatorsLSKRewardsPart = validatorsLSKRewardsPart
```

#### addPoolCreationSettings

Function adds a new pair of fee tier and tick spacing to the pool creation settings. Fee tiers are given in units of parts-per-million of the swap amount, so they are numbers between 0 and 10^6 (inclusive). Fee tier has to be different from the already existing fee tiers.

##### Execution

```python
def addPoolCreationSettings(feeTier: uint32, tickSpacing: uint32) -> None:
    if feeTier > 1000000:
        raise Exception("Fee tier can not be greater than 100%")
    for creationSettings in settings.poolCreationSettings:
        if creationSettings.feeTier == feeTier:
            raise Exception("Can not update fee tier")

    settings.poolCreationSettings.append({"feeTier": feeTier, "tickSpacing": tickSpacing})
```

#### getCurrentSqrtPrice

The function returns the current sqrt price of a given pool.

#####  Parameters
* `poolId`: The pool ID of the requested pool.
* `priceDirection`: A `bool` signifying the price direction. If `True`, the price is returned in units of `token1`/`token0`. Otherwise it is returned in `token0`/`token1`.

##### Execution

```python
def getCurrentSqrtPrice(poolId: bytes, priceDirection: bool) -> SqrtPrice:
    if pools(poolId) does not exist:
        raise Exception()
    q96SqrtPrice = bytesToQ96(pools(poolId).sqrtPrice)
    if priceDirection:
        return q96SqrtPrice
    else:
        return inv_96(q96SqrtPrice)
```

### Endpoints for Off-Chain Services

#### getProtocolSettings

Function returns the protocol settings substore as an object.

```python
def getProtocolSettings() -> dict[str, Any]:
    return protocol settings substore deserialized with settingsSchema
```

#### getCurrentSqrtPrice

The function returns the current sqrt price of a given pool.
It has the same inputs, outputs and execution as the function exposed for other modules with the same name.

Todo: decide whether we return sqrt price or actual price. Decide on the format.

#### getPriceImpact

Todo: implement.

#####  Parameters

* `poolId`: A byte array of length `NUM_BYTES_POOL_ID` with a pool ID.
* `amount0`:
* `amount1`:
* `swapDirection`: A `bool` with the direction of the swap. If `True`, `amount` represents an amount of `token0`. Otherwise, `amount` represents an amount of `token1`.

##### Returns

Open point, how to revert swaps efficiently?

##### Execution


#### getPool

The function returns the store value entry of a given pool Id in the pools substore.

#####  Parameters

* `poolId`: A byte array of length `NUM_BYTES_POOL_ID` with a pool ID.

##### Returns

The store value entry in the pools substore of the pool with pool ID equal to `poolId`.

##### Execution

```python
getPool(poolId):
    if pools(poolId) does not exist:
        raise Exception()
    return pools(poolId)
```

#### getPriceTick

The function returns the store value entry of a given price tick in the price tick substore.

#####  Parameters

* `poolId`: A byte array of length `NUM_BYTES_POOL_ID` with a pool ID.
* `tickValue`:  A `sint32` integer with the tick value.

##### Returns

The store value entry in the price tick substore of the price tick with storage key containing `poolId` and `tickValue`.

##### Execution

```python
getPriceTick(poolId, tickValue):
    if tick(poolId, tickValue) is empty:
        raise Exception()
    return tick(poolId, tickValue)
```

#### getPosition

The function returns the store value entry of a given position ID in the positions substore.

#####  Parameters

* `positionId`: A byte array of length `NUM_BYTES_POSITION_ID` with a position ID.

##### Returns

The store value entry in the positions substore of the position with position ID equal to `positionId`.

##### Execution

```python
getPosition(positionId):
    if positions(positionId) does not exist
        raise Exception()
    return positions(positionId)
```

### Genesis Block Processing

The following two steps are executed as part of the genesis block processing, see the [LIP 60][LIP60] for details.

#### Genesis assets schema

```java
genesisDEXSchema = {
    "type": "object",
    "required": [
        "stateSubstore",
        "poolSubstore",
        "priceTickSubstore",
        "positionSubstore",
        "settingsSubstore"
    ],
    "properties": {
        "stateSubstore":{
            "type": "object",
            "fieldNumber": 1,
            "required": [
                "positionCounter",
                "collectableLSKFees"
            ],
            "properties": {
                "positionCounter": {
                    "dataType": "uint64",
                    "fieldNumber": 1
                },
                "collectableLSKFees": {
                    "dataType": "uint64",
                    "fieldNumber": 2
                }
            }
        },
        "poolSubstore": {
            "type": "array",
            "fieldNumber": 2,
            "items": {
                "type": "object",
                "required": [
                    "poolId",
                    "liquidity",
                    "sqrtPrice",
                    "feeGrowthGlobal0",
                    "feeGrowthGlobal1",
                    "tickSpacing"
                ],
                "properties": {
                    "poolId": {
                        "dataType": "bytes",
                        "length": NUM_BYTES_POOL_ID,
                        "fieldNumber": 1
                    },
                    "liquidity": {
                        "dataType": "uint64",
                        "fieldNumber": 2
                    },
                    "sqrtPrice": {
                        "dataType": "bytes",
                        "maxLength": MAX_NUM_BYTES_Q96,
                        "fieldNumber": 3
                    },
                    "feeGrowthGlobal0": {
                        "dataType": "bytes",
                        "maxLength": MAX_NUM_BYTES_Q96,
                        "fieldNumber": 4
                    },
                    "feeGrowthGlobal1": {
                        "dataType": "bytes",
                        "maxLength": MAX_NUM_BYTES_Q96,
                        "fieldNumber": 5
                    },
                    "tickSpacing": {
                        "dataType": "uint32",
                        "fieldNumber": 6
                    }
                }
            }
        },
        "priceTickSubstore": {
            "type": "array",
            "fieldNumber": 3,
            "items": {
                "type": "object",
                "required": [
                    "tickId",
                    "liquidityNet",
                    "liquidityGross",
                    "feeGrowthOutside0",
                    "feeGrowthOutside1"
                ],
                "properties": {
                    "tickId": {
                        "dataType": "bytes",
                        "length": NUM_BYTES_TICK_ID,
                        "fieldNumber": 1
                    },
                    "liquidityNet": {
                        "dataType": "sint64",
                        "fieldNumber": 2
                    },
                    "liquidityGross": {
                        "dataType": "uint64",
                        "fieldNumber": 3
                    },
                    "feeGrowthOutside0": {
                        "dataType": "bytes",
                        "maxLength": MAX_NUM_BYTES_Q96,
                        "fieldNumber": 4
                    },
                    "feeGrowthOutside1": {
                        "dataType": "bytes",
                        "maxLength": MAX_NUM_BYTES_Q96,
                        "fieldNumber": 5
                    }
                }
            }
        },
        "positionSubstore": {
            "type": "array",
            "fieldNumber": 4,
            "items": {
                "type": "object",
                "required": [
                    "positionId",
                    "tickLower",
                    "tickUpper",
                    "liquidity",
                    "feeGrowthInsideLast0",
                    "feeGrowthInsideLast1"
                ],
                "properties": {
                    "positionId": {
                        "dataType": "bytes",
                        "length": NUM_BYTES_POSITION_ID,
                        "fieldNumber": 1
                    },
                    "tickLower": {
                        "dataType": "sint32",
                        "fieldNumber": 2
                    },
                    "tickUpper": {
                        "dataType": "sint32",
                        "fieldNumber": 3
                    },
                    "liquidity": {
                        "dataType": "uint64",
                        "fieldNumber": 4
                    },
                    "feeGrowthInsideLast0": {
                        "dataType": "bytes",
                        "maxLength": MAX_NUM_BYTES_Q96,
                        "fieldNumber": 5
                    },
                    "feeGrowthInsideLast1": {
                        "dataType": "bytes",
                        "maxLength": MAX_NUM_BYTES_Q96,
                        "fieldNumber": 6
                    },
                    "ownerAddress": {
                        "dataType": "bytes",
                        "length": NUM_BYTES_ADDRESS,
                        "fieldNumber": 7
                    }
                }
            }
        },
        "settingsSubstore": {
            "type": "object",
            "fieldNumber": 5,
            "required": [
            "protocolFeeAddress",
            "protocolFeePart",
            "validatorsLSKRewardsPart",
            "poolCreationSettings"
            ],
            "properties": {
                "protocolFeeAddress": {
                    "dataType": "bytes",
                    "length": NUM_BYTES_ADDRESS,
                    "fieldNumber": 1
                },
                "protocolFeePart": {
                    "dataType": "uint32",
                    "fieldNumber": 2
                },
                "validatorsLSKRewardsPart": {
                    "dataType": "uint32",
                    "fieldNumber": 3
                },
                "poolCreationSettings": {
                    "type": "array",
                    "fieldNumber": 4,
                    "items": {
                        "type": "object",
                        "required": ["feeTier", "tickSpacing"],
                        "properties": {
                            "feeTier": {
                                "dataType": "uint32",
                                "fieldNumber": 1
                            },
                            "tickSpacing": {
                                "dataType": "uint32",
                                "fieldNumber": 2
                            }
                        }
                    }
                }
            }
        }
    }
}
```
#### Genesis State Initialization
During the genesis state initialization stage, the following steps are executed. If any step fails, the block is discarded and has no further effect.

Let `genesisBlockAssetBytes` be the data bytes included in the block assets for the DEX module and let `genesisBlockAssetObject` be the deserialization of `genesisBlockAssetBytes` according to the `genesisDEXSchema`, given above.

- Initial checks on the properties of `genesisBlockAssetBytes`:
    - Position counter is correct: `int.from_bytes(positionCounter, byteorder = 'big', signed = False) == length(genesisBlockAssetObject.positionSubstore)`.
    - Across all elements of the `poolSubstore` array, all elements have unique `poolId`.
    - Across all elements of the `priceTickSubstore` array, all elements have unique `tickId`.
    - Across all elements of the `positionSubstore` array, all elements have unique `positionId`.

- Check tick range:
    - For every element of `positionSubstore` array check that `tickLower >= MIN_TICK`, `tickUpper <= MAX_TICK` and `tickLower <= tickUpper`.
- Check pool existence for ticks and positions:
    - For every element of `priceTickSubstore`, let `poolId = tickId[:NUM_BYTES_POOL_ID]`. Check that array `poolSubstore` has an element with the same `poolId`.
    - For every element of `positionSubstore`, let `poolId = tickId[:NUM_BYTES_POOL_ID]`. Check that array `poolSubstore` has an element with the same `poolId`.
- Check tick spacings:
    - For each element `priceTick` of `priceTickSubstore` the following should return `True`:

    ```python
    tickValueBytes = priceTick.tickId[-4:]
    tickValue = bytesToTick(tickValueBytes)

    poolId = priceTick.tickId[:NUM_BYTES_POOL_ID]
    pool = (element of poolSubstore with element.poolId == poolId)
    return tickValue % pool.tickSpacing == 0
    ```
    - For each element `position` of `positionSubstore` the following should return `True`:

    ```python
    poolId = position.positionId[:NUM_BYTES_POOL_ID]
    pool = (element of poolSubstore with element.poolId == poolId)
    return (position.tickLower % pool.tickSpacing == 0 and
            position.tickUpper % pool.tickSpacing == 0)
    ```
- Check that all initialized ticks correspond to positions and vice versa:
    - For every element `priceTick` of `priceTickSubstore` array, let `tickValue = int.from_bytes(priceTick.tickId[-4:], byteorder = 'big', signed = True)`; check that there exists an element `position` of `positionSubstore` array such that `tickValue == position.tickLower or tickValue == position.tickUpper`.
    - For every element `position` of `positionSubstore` array there are elements `priceTickLower` and `priceTickUpper` of `priceTickSubstore` such that the following logic returns `True`:

    ```python
    initializedTickLower = int.from_bytes(priceTickLower.tickId[-4:], byteorder = 'big', signed = True)
    initializedTickUpper = int.from_bytes(priceTickUpper.tickId[-4:], byteorder = 'big', signed = True)

    return (position.tickLower == initializedTickLower and
            position.tickUpper == initializedTickUpper)
    ```
- Check fee proportion:
    - The values `settingsSubstore.protocolFeePart` and `validatorsLSKRewardsPart` are not greater than 10^6.
    - For each element of `settingsSubstore.poolCreationSettings` the value of `feeTier` is not greater than 10^6.

- Create an entry in the DEX global state substore with:
```python
state = genesisBlockAssetObject.stateSubstore
storeKey = empty bytes
storeValue = {
"positionCounter": state.positionCounter,
"collectableLSKFees": state.collectableLSKFees
} serialized using dexGlobalStateSchema
```
- For each entry `pool` in `genesisBlockAssetObject.poolSubstore`, create an entry in the pools substore with:

```python
storeKey = pool.poolId
storeValue = {
    "liquidity": pool.liquidity,
    "sqrtPrice": pool.sqrtPrice,
    "feeGrowthGlobal0": pool.feeGrowthGlobal0,
    "feeGrowthGlobal1": pool.feeGrowthGlobal1,
    "tickSpacing": pool.tickSpacing
} serialized using poolsSchema
```

- For each entry `priceTick` in `genesisBlockAssetObject.priceTickSubstore`, create an entry in the price tick substore with:

```python
storeKey = priceTick.tickId
storeValue = {
    "liquidityNet": priceTick.liquidityNet,
    "liquidityGross": priceTick.liquidityGross,
    "feeGrowthOutside0": priceTick.feeGrowthOutside0,
    "feeGrowthOutside1": priceTick.feeGrowthOutside1
} serialized using priceTickSchema
```

- For each entry `position` in `genesisBlockAssetObject.positionSubstore`, create an entry in the positions substore with:

```python
storeKey = position.positionId
storeValue = {
    "tickLower": position.tickLower,
    "tickUpper": position.tickUpper,
    "liquidity": position.liquidity,
    "feeGrowthInsideLast0": position.feeGrowthInsideLast0,
    "feeGrowthInsideLast1": position.feeGrowthInsideLast1,
    "ownerAddress": position.ownerAddress
} serialized using positionSchema
```

- Create an entry in the protocol settings substore with:

```python
settings = genesisBlockAssetObject.settingsSubstore
storeKey = empty bytes
storeValue = {
    "protocolFeeAddress": settings.protocolFeeAddress,
    "protocolFeePart": settings.protocolFeePart,
    "validatorsLSKRewardsPart": settings.validatorsLSKRewardsPart,
    "poolCreationSettings": [{"feeTier": poolCreationSettings.feeTier,
        "tickSpacing": poolCreationSettings.tickSpacing
        } for each poolCreationSettings in settings.poolCreationSettings]
} serialized using settingsSchema
```

#### Genesis State Finalization

The DEX module doesn't execute any logic during genesis state finalization.

<!--To finalize the state of a genesis block `g`, the following logic is executed. If any step fails, the block is discarded and has no further effect.

Let `genesisBlockAssetBytes` be the data bytes included in the block assets for the DEX module and let `genesisBlockAssetObject` be the deserialization of `genesisBlockAssetBytes` according to the `genesisDEXSchema`, given above.

Then check:

```python
positionSubstore = genesisBlockAssetObject.positionSubstore
for each position in positionSubstore:
    nftIndex = position.positionId[-8:]
    nftId = {
        "chainId": CHAIN_ID_ALIAS_NATIVE,
        "collection": NFT_COLLECTION_DEX,
        "index": nftIndex
    }
    if NFT.getNFTowner(nftId) does not exits:
        raise Exception()
```-->

## Backwards Compatibility

This LIP introduces a new module with its dedicated module store and commands. Adding this module to an existing blockchain therefore requires a hardfork.

## Appendix

### Fixed Point Arithmetic

#### Definition

We represent a positive real number `r` as a `Qn` by the positive integer `Qn(r) = floor(r*2^n)` (inspired by the [Q number format][Q_wiki] for signed numbers).
In this representation, we have `n` bits of fractional precision.
We do not limit the size of `r` as modern libraries can handle integers of arbitrary size; note that all the `Qn` conversions assume no loss of significant digits.

#### Operations on Integers

For an integer `a` in the `Qn` format, we define:

* Rounding down: `roundDown_n(a) = a >> n`

* Rounding up:

```python
def roundUp_n(a):
    if a % (1 << n) == 0:
        return a >> n
    else:
        return (a >> n) + 1
```

#### Qn Arithmetic Operations

In the following definition, `*` is the integer multiplication, `//` is the integer division (rounded down). Division by zero is not permitted and should raise an error in any implementation.

For two numbers `a`,`b` and `c` in `Qn` format, we define the following arithmetic operations which all return a `Qn`:

* Addition: `add_n(a, b) = a + b`
* Subtraction: `sub_n(a,b) = a - b`, only defined if `a >= b`
* Multiplication: `mul_n(a, b) = (a * b) >> n`
* Division: `div_n(a, b) = (a << n) // b`
* Multiplication and then division: `muldiv_n(a, b, c) = roundDown_n(((a * b) << n) // c)`
* Multiplication and then division, rounding up: `muldiv_n_RoundUp(a, b, c) = roundUp_n(((a * b) << n) // c)`
* Convert to integer rounding down: `Q_n_ToInt(a) = roundDown_n(a)`
* Convert to integer rounding up: `Q_n_ToIntRoundUp(a) = roundUp_n(a)`
* Inversion in the decimal precision space: `inv_n(a) = div_n(1 << n, a)`

[tokenLIP]: https://github.com/LiskHQ/lips/blob/main/proposals/lip-0051.md
[uniswapv3whitepaper]: https://uniswap.org/whitepaper-v3.pdf
[Q_wiki]: https://en.wikipedia.org/wiki/Q_(number_format)
[q6496Uniswap]: https://docs.uniswap.org/sdk/guides/fetching-prices#understanding-sqrtprice
[tokenidLIP]: https://github.com/LiskHQ/lips/blob/main/proposals/lip-0051.md#token-identification
[getSqrtRatioAtTick]: https://github.com/Uniswap/v3-core/blob/c05a0e2c8c08c460fb4d05cfdda30b3ad8deeaac/contracts/libraries/TickMath.sol#L23
[uint256LIP]: https://github.com/LiskHQ/lips-staging/blob/main/proposals/lip_add_256_bit_integer_types_to_lisk_codec.md
[getSqrtRatioAtTickTypescript]: https://github.com/Uniswap/v3-sdk/blob/52fa487363e6ffa0dd96b554c0e2448f257dd69f/src/utils/tickMath.ts#L41
[getTickAtSqrtRatio]: https://github.com/Uniswap/v3-core/blob/c05a0e2c8c08c460fb4d05cfdda30b3ad8deeaac/contracts/libraries/TickMath.sol#L61
[getTickAtSqrtRatioTypescript]: https://github.com/Uniswap/v3-sdk/blob/52fa487363e6ffa0dd96b554c0e2448f257dd69f/src/utils/tickMath.ts#L82
[getAmount0Delta]: https://github.com/Uniswap/v3-core/blob/c05a0e2c8c08c460fb4d05cfdda30b3ad8deeaac/contracts/libraries/SqrtPriceMath.sol#L153
[getAmount0DeltaTypescript]: https://github.com/Uniswap/v3-sdk/blob/52fa487363e6ffa0dd96b554c0e2448f257dd69f/src/utils/sqrtPriceMath.ts#L25
[getAmount1Delta]: https://github.com/Uniswap/v3-core/blob/c05a0e2c8c08c460fb4d05cfdda30b3ad8deeaac/contracts/libraries/SqrtPriceMath.sol#L182
[getAmount1DeltaTypescript]: https://github.com/Uniswap/v3-sdk/blob/52fa487363e6ffa0dd96b554c0e2448f257dd69f/src/utils/sqrtPriceMath.ts#L38
[getNextSqrtPriceFromAmount1RoundingDown]: https://github.com/Uniswap/v3-core/blob/c05a0e2c8c08c460fb4d05cfdda30b3ad8deeaac/contracts/libraries/SqrtPriceMath.sol#L68
[getNextSqrtPriceFromAmount1RoundingDownTypescrypt]: https://github.com/Uniswap/v3-sdk/blob/52fa487363e6ffa0dd96b554c0e2448f257dd69f/src/utils/sqrtPriceMath.ts#L100
[getNextSqrtPriceFromAmount0RoundingUp]: https://github.com/Uniswap/v3-core/blob/c05a0e2c8c08c460fb4d05cfdda30b3ad8deeaac/contracts/libraries/SqrtPriceMath.sol#L28
[getNextSqrtPriceFromAmount0RoundingUpTypescript]: https://github.com/Uniswap/v3-sdk/blob/52fa487363e6ffa0dd96b554c0e2448f257dd69f/src/utils/sqrtPriceMath.ts#L71
[LIP60]: https://github.com/LiskHQ/lips/blob/main/proposals/lip-0060.md
[UniswapV3Fees]: https://docs.uniswap.org/protocol/concepts/V3-overview/fees
[UniswapV3Ticks]: https://github.com/Uniswap/v3-core/blob/main/contracts/UniswapV3Factory.sol
