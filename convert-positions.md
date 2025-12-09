# Convert Positions

Convert positions is a feature in Polymarket that is used for converting a NO position in a neg risk market to YES positions of other questions in the same neg risk market. Position conversion works because it does not change the expected value of the position.

Imagine we are in the 2028 Presidential Election market with JD Vance, AOC and Gavin Newsom as the candidates.

Say I hold 1 share of JD Vance **NO**, my expected payout if JD Vance loses is 1 dollar and my expected payout if JD Vance wins is 0.

| JD Vance | AOC   | Gavin Newsom |
| -------- | ----- | ------------ |
| 1 NO     | 0 NO  | 0 NO         |
| 0 YES    | 0 YES | 0 YES        |

Conversion allows me to convert my JD Vance **NO** into 1 AOC **YES** and 1 Gavin Newsom **YES**.&#x20;

| JD Vance | AOC   | Gavin Newsom |
| -------- | ----- | ------------ |
| 0 NO     | 0 NO  | 0 NO         |
| 0 YES    | 1 YES | 1 YES        |

My expected payout if JD Vance loses, meaning AOC or Gavin Newsom must have won, is 1 dollar. And my expected payout if JD Vance wins, meaning both AOC and Gavin Newsom must have lost, is 0. It is exactly the same as the expected payout if I only held 1 JD Vance **NO**.

Now let's say I hold 1 JD Vance **NO** and 1 AOC **NO**.&#x20;

* My expected payout if JD Vance wins is 1 dollar (AOC resolves to NO)
* My expected payout if AOC wins is 1 dollar (JD Vance resolves to NO)
* My expected payout if Gavin Newsom wins is 2 dollars (both AOC and JD Vance resolve to No)
* So my minimum payout is **1 dollar**

| JD Vance | AOC   | Gavin Newsom |
| -------- | ----- | ------------ |
| 1 NO     | 1 NO  | 0 NO         |
| 0 YES    | 0 YES | 0 YES        |

Now, let's say I want to convert my JD Vance and AOC NO shares into Gavin Newsom YES.

| JD Vance | AOC   | Gavin Newsom |
| -------- | ----- | ------------ |
| 0 NO     | 0 NO  | 0 NO         |
| 0 YES    | 0 YES | 1 YES        |

When I convert 2 NO shares from different questions into a YES share of the last question, I am only going to get 1 YES share instead of 2. Where did the extra YES share go? Turns out I am also getting 1 dollar. This is because by having 2 NO shares from different questions, at least one of them is guaranteed to resolve to NO (this is a winner takes all market). Therefore it makes sense to release the collateral early and only give the converter 1 YES share.

Now the expected values are:

* If JD Vance wins, I get 1 dollar from the conversion and 0 for holding Gavin Newsom YES
* If AOC wins, I get 1 dollar from the conversion and 0 for holding Gavin Newsom YES
* If Gavin Newsom wins, I get 1 dollar from the conversion and 1 dollar for holding Gavin Newsom YES
* The expected payout is **exactly the same** as before the conversion!

Now you might ask, why would I do that if my expected payout does not change?

* **Liquidity Release and Capital Efficiency**: Conversion often releases locked collateral (e.g., converting 1 NO A + 1 NO B to 1 YES C + 1 USDC in a three-candidate market), freeing up funds for reinvestment elsewhere. This reduces opportunity costs compared to holding illiquid positions or waiting for resolution.
* **Cost Savings on Fees and Slippage**: Unlike trading on the order book (which can incur fees and potential slippage), conversions are handled protocol-level with minimal or no additional costs, making it more economical for large positions.
* **Simplified Position Management**: If you hold multiple NO positions across different outcomes, conversion lets you consolidate into a cleaner exposure (fewer tokens, easier to track and manage).

#### NegRiskAdapter.convertPositions

The index set must only have up to "the market's question count" bits set  as it represents the set of positions to convert

```solidity
uint256 questionCount = md.questionCount();
if ((_indexSet >> questionCount) > 0) revert InvalidIndexSet();
```

The function loops through the index set and see how many NO positions does the user want to convert. `noPositionCount` is incremented when the `index-th` bit of the `_indexSet` is set.

```solidity
uint256 index = 0;
uint256 noPositionCount;

// count number of no positions
while (index < questionCount) {
    unchecked {
        if ((_indexSet & (1 << index)) > 0) {
            ++noPositionCount;
        }
        ++index;
    }
}
```

The YES position count is the market's question count minus the NO position count. The function mints wrapped collateral for each YES position. It is **OK** to mint additional wrapped collateral without increasing USDT backing. Remember, the expected payout does not change after the conversion.

```solidity
uint256 yesPositionCount = questionCount - noPositionCount;
wcol.mint(yesPositionCount * _amount);
```

The function does another loop to keep track of the NO position IDs to burn (if the `index-th` bit is set) and keep track of the YES position IDs to mint (if the `index-th` bit is not set). It splits position to mint the position. The position is backed by wrapped collateral, which was minted in the lines above.

```solidity
uint256[] memory noPositionIds = new uint256[](noPositionCount);
uint256[] memory yesPositionIds = new uint256[](yesPositionCount);
uint256[] memory accumulatedNoPositionIds = new uint256[](yesPositionCount);

// populate noPositionIds and yesPositionIds
// split yes positions
{
    uint256 noIndex;
    uint256 yesIndex;
    index = 0;

    while (index < questionCount) {
        bytes32 questionId = NegRiskIdLib.getQuestionId(_marketId, uint8(index));

        if ((_indexSet & (1 << index)) > 0) {
            // NO
            noPositionIds[noIndex] = getPositionId(questionId, false);

            unchecked {
                ++noIndex;
            }
        } else {
            // YES
            yesPositionIds[yesIndex] = getPositionId(questionId, true);
            accumulatedNoPositionIds[yesIndex] = getPositionId(questionId, false);

            // split position to get yes and no tokens
            // the no tokens will be discarded
            _splitPosition(getConditionId(questionId), _amount);

            unchecked {
                ++yesIndex;
            }
        }
        unchecked {
            ++index;
        }
    }
}
```

After the loop, the function burns the user's NO positions and also the NO positions that came from the splits above. The NO positions from the split have to be burned, otherwise the user will be able to merge back the YES and NO positions that were just split to get more than his expected payout.

```solidity
// transfer the caller's no tokens _and_ accumulated no tokens to the burn address
// these must never be redeemed
{
    ctf.safeBatchTransferFrom(
        msg.sender, NO_TOKEN_BURN_ADDRESS, noPositionIds, Helpers.values(noPositionIds.length, _amount), ""
    );
    ctf.safeBatchTransferFrom(
        address(this),
        NO_TOKEN_BURN_ADDRESS,
        accumulatedNoPositionIds,
        Helpers.values(yesPositionCount, _amount),
        ""
    );
}
```

If the user converts more than one NO position, the function releases USDC to the converter for the reason mentioned above. At least `n - 1` of the NO positions are guaranteed to win.

```solidity
if (noPositionIds.length > 1) {
    // collateral out is always proportional to the number of no positions minus 1
    uint256 multiplier = noPositionIds.length - 1;
    
    // ...
    
    // transfer collateral to sender
    wcol.release(msg.sender, multiplier * amountOut);
}
```

Finally, the function transfers YES positions to the converter.

```solidity
if (yesPositionIds.length > 0) {
    // ...
    
    // transfer yes tokens to sender
    ctf.safeBatchTransferFrom(
        address(this), msg.sender, yesPositionIds, Helpers.values(yesPositionIds.length, amountOut), ""
    );
}
```
