# Conditional Tokens Framework

[ConditionalTokens](https://github.com/Polymarket/conditional-tokens-contracts/blob/master/contracts/ConditionalTokens.sol)

The conditional tokens framework is an ERC-1155 that enables splitting a market into 2 opposite sides. For example, we can have a market "Will Donald Trump win the 2024 presidential election?" and there will be 2 tokens that represent the results, YES and NO.

#### Terminologies

**Question ID**: A question ID is the identifier for the question to be answered by the oracle. The contract is agnostic to who the oracle is, but in the context of Polymarket, the oracle is Polymarket's UMA CTF adapter.

**Condition ID:** A condition ID is the keccak256 hash of the `oracle` address, the question ID and the `outcomesSlotCount`. The outcome slots count is the number of potential outcomes which should be used for this condition. It can be up to 256, but it is always 2 for Polymarket. The condition can only be resolved to YES or NO. Including the oracle address in the condition ID also means there can be different oracles asking the same question, but the condition IDs will not be the same. Theoretically there is no one stopping another prediction market on Polygon from using Polymarket's CTF contract. Their token IDs and collateral will not co-mingle with each other.

**Collection ID**: A collection ID is the ID of a specific outcome. It is calculated with `parentCollectionId`, `conditionId` and `indexSet`. The formula is quite complex, but if you wish to dig deeper you can visit [this link](https://github.com/Polymarket/conditional-tokens-contracts/blob/master/contracts/CTHelpers.sol#L392). In CTF language a collection can be the child of another collection, which creates nested conditions. In reality I have not seen it being used by any prediction markets ever. A top-level collection ID has a parent collection ID of `bytes32(0)`. `indexSet` refers to possible conditions expressed in a bitmap. A full condition set is equal to `0b11` , whereas `0b01` is equal to `YES` and `0b10` is equal to `NO`

**Position ID**: A position ID refers to a condition backed by a specific collateral token. It is the final ERC-1155 token IDs that we can trade on Polymarket. Collateral token in this case is USDC, but it is possible to create a different position with another collateral token such as USDT. They both point to the same question/condition/collection. If a market result is reported, both positions can be simultaneously resolved.

#### Getting a question to be trade ready

The sequence of functions that need to be called is as follows:

1. Generate a unique question ID from a legible question in human language. This is done in the UMA CTF adapter which we will talk about in the next page.
2. Call `prepareCondition(oracle, questionId, 2)` with the question ID we just generated. Polymarket's UMA CTF adapter does step 1 and 2 in the same transaction. The current implementation of the CTF contract has a vulnerability that can DOS UMA CTF adapter's question initialization. The function accepts `oracle` as an argument, but it should have just used `msg.sender` instead. Anyone can call `prepareCondition` with the same arguments as the UMA CTF adapter and if it lands on-chain earlier UMA CTF adapter's transaction will revert.

#### Getting CTF tokens

**1 full CTF = 1 USDC**: If you deposit 1 USDC into the CTF contract for a specific condition ID, you receive 1 YES token and 1 NO token. You are free to sell the tokens in the market for however much price you can get, but the maximum amount you can get from 1 YES + 1 NO through the CTF contract is always 1 USDC. For a market that has resolved (to YES), your 1 YES token is worth 1 USDC and your 1 NO token is worth 0 USDC.

To get CTF tokens, you give USDC allowance to the CTF contract and then call `splitPosition` with the arguments `USDC` address, `bytes32(0)`, `conditionId`, `partition` which is just an array of size 2 (`[1, 2]`) and the `amount` of USDC to deposit.

The contract calculates the position (token) IDs with the parameters provided:

```solidity
uint[] memory positionIds = new uint[](partition.length);
positionIds[i] = CTHelpers.getPositionId(collateralToken, CTHelpers.getCollectionId(parentCollectionId, conditionId, indexSet));
```

Individual partition must be a subset of the full index set in order for the contract to correctly calculate the position ID.

```solidity
uint indexSet = partition[i];
require(indexSet > 0 && indexSet < fullIndexSet, "got invalid index set");
require((indexSet & freeIndexSet) == indexSet, "partition not disjoint");
```

The contract mints a full set of tokens for the given CTF condition.

```solidity
uint[] memory amounts = new uint[](partition.length);
amounts[i] = amount;

_batchMint(
    msg.sender,
    // position ID is the ERC 1155 token ID
    positionIds,
    amounts,
    ""
);
```

#### Getting back collaterals by merging

If you have 1 YES token and 1 NO token, one of the positions must resolve to a full payout (If you own 1 YES and 1 NO of "Will Donald Trump win the 2024 presidential election?", you must get back 1 USDC in the end. The market can only resolve one way or the other). Therefore you don't have to wait for the payout to be reported.

You merge your positions by calling `mergePositions`. It is an inverse of `splitPosition`, the contract burns your CTF tokens and sends you back USDC.

One thing I didn't mention earlier: in both `splitPosition` and `mergePositions`, the contract checks a full set of partition is provided through the variable `freeIndexSet`. It is initialized as the full index set and with each partition it is `XOR`ed with the partition to zero out individual bits. Only if the final free index set is 0 (all partitions are covered) does the contract mints CTFs / sends back collateral tokens.

```solidity
/// In splitPosition
if (freeIndexSet == 0) {
    // Partitioning the full set of outcomes for the condition in this branch
    if (parentCollectionId == bytes32(0)) {
        require(collateralToken.transferFrom(msg.sender, address(this), amount), "could not receive collateral tokens");
    }
}

/// In mergePositions
if (freeIndexSet == 0) {
    if (parentCollectionId == bytes32(0)) {
        require(collateralToken.transfer(msg.sender, amount), "could not send collateral tokens");
    }
}
```

#### Getting back collaterals by redeeming

**Report payouts**

Before a redemption can happen the market result has to be available. Market results are made available by calling `reportPayouts`. This function has to be called by the address that originally prepared the condition otherwise it will result in a different condition ID.

```solidity
// IMPORTANT, the oracle is enforced to be the sender because it's part of the hash.
bytes32 conditionId = CTHelpers.getConditionId(msg.sender, questionId, outcomeSlotCount);
```

This is effectively an access control.

Polymarket does this by retrieving the result from UMA's optimistic oracle and then reporting the result to the CTF. The only valid payouts are `[0, 1]`, `[1, 0]` and `[1, 1]`. Having both payouts as 1 means YES and NO each gets 50% of the payout.

```solidity
library PayoutHelperLib {

    function isValidPayoutArray(uint256[] memory payouts) internal pure returns (bool) {
        if (payouts.length != 2) return false;

        // Payout must be [0,1], [1,0] or [1,1]
        // if payout[0] is 1, payout[1] must be 0 or 1
        if ((payouts[0] == 1) && (payouts[1] == 0 || payouts[1] == 1)) {
            return true;
        }

        // If payout[0] is 0, payout[1] must be 1 
        if ((payouts[0] == 0) && (payouts[1] == 1)) {
            return true;
        }
        return false;
    }
}
```

For each condition, CTF provides 2 mappings `payoutNumerators` and `payoutDenominator`. Payout numerators refers to what a position gets out of the total payout and payout denominator is the total payout.

```solidity
uint den = 0;
for (uint i = 0; i < outcomeSlotCount; i++) {
    uint num = payouts[i];
    den = den.add(num);

    require(payoutNumerators[conditionId][i] == 0, "payout numerator already set");
    payoutNumerators[conditionId][i] = num;
}
require(den > 0, "payout is all zeroes");
payoutDenominator[conditionId] = den;
```

$$
\text{payout} = \text{userPositionBalanceOfSpecificOutcome} \times \frac{\text{payoutNumerators}[\text{conditionId}][\text{outcomeId}]}{\text{payoutDenominator}[\text{conditionId}]}
$$

The function `redeemPositions` just sums the payout for all position IDs specified by the positions holder.

```solidity
/// Loop through each indexSet to calculate each position's payout
/// and add it to the totalPayout
uint payoutNumerator = 0;
for (uint j = 0; j < outcomeSlotCount; j++) {
    if (indexSet & (1 << j) != 0) {
        payoutNumerator = payoutNumerator.add(payoutNumerators[conditionId][j]);
    }
}

uint payoutStake = balanceOf(msg.sender, positionId);
if (payoutStake > 0) {
    totalPayout = totalPayout.add(payoutStake.mul(payoutNumerator).div(den));
    _burn(msg.sender, positionId, payoutStake);
}

/// Send back collateral token if the positions belong to a top-level collection
if (totalPayout > 0) {
    if (parentCollectionId == bytes32(0)) {
        require(collateralToken.transfer(msg.sender, totalPayout), "could not transfer payout to message sender");
    }
}
```
