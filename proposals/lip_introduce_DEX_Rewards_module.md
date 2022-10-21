```
LIP: <LIP number>
Title:  Introduce DEX Rewards module
Author: Sergey Shemyakov <sergey.shemyakov@lightcurve.io>
Type: Standards Track
Created: <YYYY-MM-DD>
Updated: <YYYY-MM-DD>
Required: LIP 0044, LIP 0051
```

## Abstract

This LIP introduces the DEX Rewards module that is responsible for the minting and distributing rewards on the DEX sidechain. Specifically, it is fundamental to the token economics of the DEX native token. In this LIP we define the constants, state transitions and events of the DEX Rewards module.

## Copyright

This LIP is licensed under the [Creative Commons Zero 1.0 Universal](https://creativecommons.org/publicdomain/zero/1.0/).

## Motivation

The decentralized exchange sidechain in the Lisk ecosystem has three different types of actors: validators, traders and liquidity providers. The goal of the DEX Rewards module is to incentivize the participation of validators and liquidity providers and to guide the DEX sidechain native token economics.

The DEX sidechain will keep big volumes of different tokens, so the job of securing the chain is essential. Validator incentivization aims to motivate users to invest their DEX tokens into supporting the chain.

Liquidity providers are the main actors in a decentralized exchange: their liquidity makes swaps possible. DEX Rewards module aims to attract token holders by giving out rewards. In additional to financial gain, liquidity providers obtain a stake in the whole sidechain and are motivated to further contribute to the success of the exchange.

## Rationale

Incentivization is accomplished together with the [DEX module][dexModule]: the DEX Rewards module mints reward tokens to the block validator and Liquidity Provider Rewards Pool. The DEX module distributes tokens from this pool among liquidity providers. Additionally, the DEX module collects LSK rewards for DEX sidechain validators and the DEX Rewards module distributes the rewards between validators once every round.


<img src="lip-introduce_DEX_Rewards_module/rewards-diagram.png" alt="drawing" width="845">

_Figure 1: the diagram of DEX rewards payout. The green arrows indicate the logic specified in the DEX Rewards module, the blue arrows indicate the DEX module logic._

The rewards in LSK for validators provide additional incentive to be a DEX sidechain validator and thus boosts the value of the DEX sidechain token.

### Liquidity Provider Reward Payout

Liquidity provider rewards are paid out in the DEX module, however we explain the payout logic here.

Liquidity providers are incentivized to provide liquidity in particular pools, as indicated by the incentivized pool list in the DEX module. The rewards are paid out together with the collected fees. The amount is proportional to the liquidity provided and to the number of blocks when the price was in the range of the position at the end of the block. Additionally, the rewards depend on the incentivized pool multiplier: the higher is the multiplier of a pool, the bigger reward is distributed among the liqudity providers of the pool.

These payout algorithms aim to incentivize the actors proportionally to their contribution and to the time duration when their liquidity was active. The minting of a fixed amount of rewards per block bounds the reward token inflation.

## Specification

The DEX Rewards module has module name `MODULE_NAME_DEX_REWARDS`.

### Notation and Constants

We define the following constants:

| **Name**                  |   **Type**        |       **Value**       |       **Description**                                 |
|---------------------------|-------------------|-----------------------|-------------------------------------------------------|
|`MODULE_NAME_DEX_REWARDS`  |     `string`        |     "dexRewards"      |Name of the DEX Rewards module.                   |
|`MODULE_NAME_DEX`          |     `string`        |           "dex"       |Name of the DEX module, as  defined in the [DEX module][dexModule].|
|`NUM_BYTES_ADDRESS`        |   `uint32`        |           20          |The number of bytes of an address.|
|`ADDRESS_LIQUIDITY_PROVIDER_REWARDS_POOL` |`bytes`|`SHA256(b"liquidityProviderRewardsPool")[:NUM_BYTES_ADDRESS]`|  The address of the liquidity provider rewards pool, as  defined in the [DEX module][dexModule].|
|`ADDRESS_VALIDATOR_REWARDS_POOL`|`bytes`|`SHA256(b"validatorRewardsPool")[:NUM_BYTES_ADDRESS]`|The address of the validator rewards pool, as  defined in the [DEX module][dexModule].|
|`TOKEN_ID_DEX_NATIVE`      |   `bytes`           |           TBA         |Token ID of the native token of DEX sidechain.         |
|   `TOKEN_ID_LSK`          |   `bytes`           |`0x00 00 00 00 00 00 00 00`|       Token ID of the LSK token.                  |
|   `EPOCH_LENGTH_REWARD_REDUCTION`          |   `uint32`           | 3000000 |           The duration of the epoch after which block reward decreases.   |
|   `PERCENTAGE_VALIDATOR_REWARD`          |   `uint32`           | 50 |     The fraction of block reward paid to the validator, in percents. The rest is paid to liqudity providers.   |
|`EVENT_NAME_VALIDATOR_TRADE_REWARDS_PAYOUT`|`string`|   "validatorTradeRewardsPayout"  |Name of the validator trade rewards payout event.|
|`EVENT_NAME_GENERATOR_REWARDS_PAYOUT`|    `string`   |       "generatorRewardsPayout"            | Name of the generator rewards payout event.    |
|`REWARD_NO_REDUCTION`      |   `uint32`        |       0                   |Return code for no block reward reduction.         |
|`REWARD_REDUCTION_SEED_REVEAL`|`uint32`        |       1                   |Return code for block reward reduction because of the failed seed reveal.|
|`REWARD_REDUCTION_MAX_PREVOTES`|`uint32`       |       2                   |Return code for block reward reduction because the block header does not imply the maximal number of prevotes.|
|`REWARD_REDUCTION_FACTOR_BFT`|`uint32`         |       4                   |The reduction factor for validator block reward in case when the block header does not imply the maximal number of prevotes.|

### State Store

This module does not use the state store.

### Commands

This module does not define any commands.

### Events

#### ValidatorTradeRewardsPayout

This event is emitted when the validator LSK fees are transferred to a validator. It has the name `EVENT_NAME_VALIDATOR_TRADE_REWARDS_PAYOUT`.

##### Topics

- `validatorAddress`: the address of the validator that obtains the fee.

##### Data

```java
validatorTradeRewardsPayoutSchema = {
    type: "object",
    properties: {
        "amount": {
            "dataType": "uint64",
            "fieldNumber": 1
        }
    }
    required = ["amount"]
}
```

- `amount`: the amount of rewards in LSK paid to the validator.

#### GeneratorRewardMinted

This event is emitted when the block generation validator reward is minted. It has the name `EVENT_NAME_GENERATOR_REWARDS_PAYOUT`.

##### Topics

- `generatorAddress`: the address of the block generator that obtains the reward.

##### Data

```java
generatorRewardsPayoutSchema = {
    "type": "object",
    "required": ["amount", "reduction"],
    "properties": {
        "amount": {
            "dataType": "uint64",
            "fieldNumber": 1
        },
        "reduction": {
            "dataType": "uint32",
            "fieldNumber": 2
        }
    }
}

```
- `amount`: the amount of rewards minted.
- `reduction`: an integer code specifying the reason for reduction. Allowed values are: `REWARD_REDUCTION_SEED_REVEAL`, `REWARD_REDUCTION_MAX_PREVOTES`, `REWARD_NO_REDUCTION`.

### Internal Functions

The DEX Rewards module defines the following internal functions.

#### getDefaultBlockReward

The function returns the block reward for a given height.

##### Execution

```python
def getDefaultBlockReward(height: int) -> int:
    if height < EPOCH_LENGTH_REWARD_REDUCTION:
        return 800000000
    if height < 2*EPOCH_LENGTH_REWARD_REDUCTION:
        return 700000000
    if height < 3*EPOCH_LENGTH_REWARD_REDUCTION:
        return 600000000
    if height < 4*EPOCH_LENGTH_REWARD_REDUCTION:
        return 500000000
    return 400000000
```

#### transferValidatorLSKRewards

This function transfers the validator LSK rewards from the validator rewards pool, sharing it equally among the current validators. The referenced functions are defined in the [Token module LIP](https://github.com/LiskHQ/lips/blob/main/proposals/lip-0051.md).

```python
def transferValidatorLSKRewards(validators: list[Address]) -> None:
    availableRewards = Token.getLockedAmount(ADDRESS_VALIDATOR_REWARDS_POOL, MODULE_NAME_DEX, TOKEN_ID_LSK)
    shareAmount = availableRewards // length(validators)    # use integer division
    if shareAmount != 0:
        Token.unlock(ADDRESS_VALIDATOR_REWARDS_POOL, MODULE_NAME_DEX, TOKEN_ID_LSK, shareAmount * length(validators))

        for validator in validators:
            Token.transfer(ADDRESS_VALIDATOR_REWARDS_POOL,
                            TOKEN_ID_LSK,
                            shareAmount,
                            validator)
            emitEvent(
                module = MODULE_NAME_DEX_REWARDS,
                type = EVENT_NAME_VALIDATOR_TRADE_REWARDS_PAYOUT,
                data = {
                    "amount": shareAmount
                    },
                topics = [validator])
```

#### getValidatorBlockReward

This function is used to retrieve the reward of a block at a given height, taking into account the possible reward reductions for not participating actively in the BFT protocol or not revealing a valid seed (the reward reduction logic is analogous to the [LIP 0042](https://github.com/LiskHQ/lips/blob/main/proposals/lip-0042.md#getblockreward)).

```python
def getValidatorBlockReward(blockHeader: BlockHeader) -> tuple[uint64, uint32]:
    if Random.isSeedRevealValid(blockHeader.generatorAddress, blockHeader.seedReveal) == False:
        return (0, REWARD_REDUCTION_SEED_REVEAL)
    validatorReward = getDefaultBlockReward(blockHeader.height) * PERCENTAGE_VALIDATOR_REWARD / 100
    if impliesMaximalPrevotes(blockHeader) == False:
        return (validatorReward // REWARD_REDUCTION_FACTOR_BFT, REWARD_REDUCTION_MAX_PREVOTES)
    return (validatorReward, REWARD_NO_REDUCTION)
```

### Protocol Logic for Other Modules

#### getLPBlockRewards

This function returns the total amount of block rewards for liquidity providers in a block range with the given start height and end height. The reward for the block at start height is excluded, the reward for the block at end height is included.

```python
def getLPBlockRewards(startHeight: int, endHeight: int) -> int:
    if endHeight < startHeight:
        raise Exception()
    EPOCHS = [EPOCH_LENGTH_REWARD_REDUCTION,
        2*EPOCH_LENGTH_REWARD_REDUCTION,
        3*EPOCH_LENGTH_REWARD_REDUCTION,
        4*EPOCH_LENGTH_REWARD_REDUCTION]
    height = startHeight + 1    # reward for the start block is excluded
    rewards = 0
    for changeHeight in EPOCHS:
        if changeHeight > startHeight and changeHeight < endHeight:
            rewards += (changeHeight - height) * getDefaultBlockReward(height)
            height = changeHeight
    rewards += (endHeight - height) * getDefaultBlockReward(height)
    rewards += getDefaultBlockReward(endHeight)
    return rewards * (100 - PERCENTAGE_VALIDATOR_REWARD) / 100
```

### Endpoints for Off-Chain Services

This module does not define any specific endpoints for off-chain services.

### Block Processing

#### After Transactions Execution

The following steps are executed after processing all transactions in a block to distribute the rewards.

```python
def afterTransactionsExecute(b: Block) -> None:
    (blockReward,reduction) = getValidatorBlockReward(b.header)
    if blockReward > 0:
        Token.mint(b.header.generatorAddress, TOKEN_ID_DEX_NATIVE, blockReward)
    emitEvent(
        module = MODULE_NAME_DEX_REWARDS,
        type = EVENT_NAME_GENERATOR_REWARDS_PAYOUT,
        data = {
            "amount": blockReward,
            "reduction": reduction
        }
        topics = [b.header.generatorAddress]
    )

    liquidityReward = getDefaultBlockReward(blockHeader.height) * (100 - PERCENTAGE_VALIDATOR_REWARD) / 100
    Token.mint(ADDRESS_LIQUIDITY_PROVIDER_REWARDS_POOL, TOKEN_ID_DEX_NATIVE, liquidityReward)
    Token.lock(ADDRESS_LIQUIDITY_PROVIDER_REWARDS_POOL, MODULE_NAME_DEX, TOKEN_ID_DEX_NATIVE, liquidityReward)

    validators = Validators.getGeneratorList()
    if b.header.height % length(validators) == 0:
        transferValidatorLSKRewards(validators)
```

[dexModule]: https://github.com/LiskHQ/lips-staging/blob/add-LIP-DEX-Rewards-module/proposals/lip-introduce_DEX_Module.md
