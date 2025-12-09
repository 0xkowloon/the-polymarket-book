# Market Resolution

Polymarket uses UMA for market resolution. UMA and CTF are connected via the contract [UmaCtfAdapter](https://github.com/Polymarket/uma-ctf-adapter/blob/main/src/UmaCtfAdapter.sol). The adapter acts as the CTF's oracle and it requests for data / retrieves results from UMA's [Optimistic Oracle](https://github.com/UMAprotocol/protocol/blob/master/packages/core/contracts/optimistic-oracle-v2/implementation/OptimisticOracleV2.sol).

#### Terminologies

**Optimistic Oracle**: An optimistic oracle is the protocol developed by UMA that allows anyone to ask questions and get answers. By optimistic it means answerers are assumed to be honest and answers are by default accepted (hence the oracle is "optimistic"). The reality is of course different, so it is possible to dispute an answer.

[**Yes or No Query**](https://docs.uma.xyz/verification-guide/yes_or_no): A UMA price identifier intended for plaintext binary questions. The answers must be one of YES (p2), NO (p1), UNKNOWN (p3) or TOO\_EARLY\_RESPONSE (p4).

**Ancillary Data**: Human-readable text that describes the question and the rules of resolution. Polymarket appends the function caller to the ancillary data and refers to it as the `initializer`.

**Bond**: A bond is required to be posted by proposers and disputers to prevent them from acting maliciously. The higher the stake, the higher the bond should be.

**Reward**: A reward is given to the result proposer. Similar to bonds, the higher the stake, the higher the reward should be. For Polymarket it ranges from 5 to 750 USDC.

**Liveness**: It is the amount of time that must have elapsed before a price proposal expires. When it expires, the price is considered final.

**Price**: Price is not the price of an asset in the traditional sense. It refers to the answer of a question. When Polymarket initializes a market, it makes a "price request" to UMA's Optimistic Oracle.

#### Initializing a question

The function `initialize` is called when Polymarket initializes a market. It does the following:

1. Saves the question on-chain

```solidity
function _saveQuestion(
    address creator,
    bytes32 questionID,
    bytes memory ancillaryData,
    uint256 requestTimestamp,
    address rewardToken,
    uint256 reward,
    uint256 proposalBond,
    uint256 liveness
) internal {
    questions[questionID] = QuestionData({
        requestTimestamp: requestTimestamp,
        reward: reward,
        proposalBond: proposalBond,
        liveness: liveness,
        manualResolutionTimestamp: 0,
        resolved: false,
        paused: false,
        reset: false,
        refund: false,
        rewardToken: rewardToken,
        creator: creator,
        ancillaryData: ancillaryData
    });
}
```

2. Prepares CTF condition with the generated question ID. The question ID is a keccak256 hash of the modified ancillary data.

```solidity
bytes memory data = AncillaryDataLib._appendAncillaryData(msg.sender, ancillaryData);
questionID = keccak256(data);

ctf.prepareCondition(address(this), questionID, 2);
```

3. Makes a price request to UMA.&#x20;

```solidity
optimisticOracle.requestPrice(
    YES_OR_NO_IDENTIFIER, requestTimestamp, ancillaryData, IERC20(rewardToken), reward
);
```

4. Makes the price request event based (It forbids TOO\_EARLY\_RESPONSE and makes the price request automatically refund reward on dispute).

```solidity
optimisticOracle.setEventBased(YES_OR_NO_IDENTIFIER, requestTimestamp, ancillaryData);
```

5. Makes Optimistic Oracle sends a `priceDisputed` callback on price dispute.

```solidity
optimisticOracle.setCallbacks(
    YES_OR_NO_IDENTIFIER,
    requestTimestamp,
    ancillaryData,
    false, // DO NOT set callback on priceProposed
    true, // DO set callback on priceDisputed
    false // DO NOT set callback on priceSettled
);
```

6. Sets bond and liveness

```solidity
if (bond > 0) optimisticOracle.setBond(YES_OR_NO_IDENTIFIER, requestTimestamp, ancillaryData, bond);
if (liveness > 0) {
    optimisticOracle.setCustomLiveness(YES_OR_NO_IDENTIFIER, requestTimestamp, ancillaryData, liveness);
}
```

Technically the question can be instantly resolved, there isn't a timestamp that we must live pass to propose a price. If it is too early the answer will be disputed.

#### Price Proposal

A market's result is proposed to UMA's Optimistic Oracle by a proposer by calling the function [proposePrice](https://github.com/UMAprotocol/protocol/blob/master/packages/core/contracts/optimistic-oracle-v2/implementation/OptimisticOracleV2.sol#L360C14-L360C26). The price request's state must be `Requested` (The price request must exist and must not have a proposed price already).

```solidity
require(
    _getState(requester, identifier, timestamp, ancillaryData) == State.Requested,
    "proposePriceFor: Requested"
);
```

The function sets the `proposer` and the `proposedPrice` and it cannot be set again (Ok I lied, I will talk about dispute).

The `expirationTime` is set to the current timestamp + the price request's custom liveness. What that means is even after the price has been proposed, the price request isn't considered final and retrieving the price request status will not return `Expired` until the timestamp is past this expiration time. Polymarket's UMA CTF adapter can only report payouts to the CTF contract after expiration time.

```solidity
request.proposer = proposer;
request.proposedPrice = proposedPrice;

// If a custom liveness has been set, use it instead of the default.
request.expirationTime = getCurrentTime().add(
    request.requestSettings.customLiveness != 0 ? request.requestSettings.customLiveness : defaultLiveness
);
```

When the proposer submits a price request, it has to pay for a `bond`. This is needed in order to prevent malicious proposers. If the proposer is malicious, a disputer can dispute the price request and confiscates the bond if the disputer turns out to be right.

<pre class="language-solidity"><code class="lang-solidity"><a data-footnote-ref href="#user-content-fn-1">totalBond</a> = request.requestSettings.bond.add(request.finalFee);
if (totalBond > 0) request.currency.safeTransferFrom(msg.sender, address(this), totalBond);
</code></pre>

#### Price Dispute

A price proposal can be disputed as long as it is not expired. It is done by calling [disputePrice](https://github.com/UMAprotocol/protocol/blob/master/packages/core/contracts/optimistic-oracle-v2/implementation/OptimisticOracleV2.sol#L458).

```solidity
require(
    _getState(requester, identifier, timestamp, ancillaryData) == State.Proposed,
    "disputePriceFor: Proposed"
);
```

The disputer has to pay the same amount of bond and fee as the price proposer to enter into a dispute as the loser has to pay the winner. **Without a bond on each side, the side that is not paying can act maliciously for free**.

```solidity
uint256 finalFee = request.finalFee;
uint256 bond = request.requestSettings.bond;
totalBond = bond.add(finalFee);
if (totalBond > 0) {
    request.currency.safeTransferFrom(msg.sender, address(this), totalBond);
}
```

Half of the bond goes to the store contract and not to the future winner of the dispute. The reason given by UMA is

> The part of the bond that doesn't go to the disputer in a successful dispute goes to the UMA `Store` contract. The reason the disputer does not get the full amount is to prevent a malicious proposer from making a bad proposal and then front-running an honest disputer to dispute themselves if they get caught.

```solidity
// Along with the final fee, "burn" part of the loser's bond to ensure that a larger bond always makes it
// proportionally more expensive to delay the resolution even if the proposer and disputer are the same
// party.

// The total fee is the burned bond and the final fee added together.
uint256 totalFee = finalFee.add(_computeBurnedBond(request));
if (totalFee > 0) {
    request.currency.safeIncreaseAllowance(address(store), totalFee);
    _getStore().payOracleFeesErc20(address(request.currency), FixedPoint.Unsigned(totalFee));
}
```

When there is a dispute, the current price request is discarded and the Optimistic Oracle makes a new price request with the same data.

```solidity
_getOracle().requestPrice(
    identifier,
    _getTimestampForDvmRequest(request, timestamp),
    _stampAncillaryData(ancillaryData, requester)
);
```

The original reward sent by Polymarket's UMA CTF adapter is refunded to the contract. The code inside the `if` statement is executed because `refundOnDispute` is set to true by UMA CTF adapter by calling `setEventBased` during its initial price request.

```solidity
// Compute refund.
uint256 refund = 0;
if (request.reward > 0 && request.requestSettings.refundOnDispute) {
    refund = request.reward;
    request.reward = 0;
    request.currency.safeTransfer(requester, refund);
}
```

Finally, it makes a callback to Polymarket's UMA CTF adapter to notify the contract of the dispute. `callbackOnPriceDisputed` is set to true by UMA CTF adapter also by calling `setCallbacks` during the initial price request.

```solidity
if (address(requester).isContract() && request.requestSettings.callbackOnPriceDisputed)
    OptimisticRequester(requester).priceDisputed(identifier, timestamp, ancillaryData, refund);
```

UMA CTF adapter is expected to implement the function [priceDisputed](https://github.com/Polymarket/uma-ctf-adapter/blob/main/src/UmaCtfAdapter.sol#L163C2-L163C3) that observes the [OptimisticRequester](https://github.com/UMAprotocol/protocol/blob/master/packages/core/contracts/optimistic-oracle-v2/implementation/OptimisticOracleV2.sol#L30) interface.

The function refunds the question creator the original reward if the question is already resolved. This is normally not the case. It should only happen if the contract's admin has manually resolved the question.

The `refund` flag is set to true only if `questionData`'s `reset` flag is true. This is also not the case for the first dispute. The refund flag is only set to true on reset, which happens as the last step of the function. The reset flag is only going to be set to true if there is a second dispute.

```solidity
/// @notice Callback which is executed on dispute
/// Resets the question and sends out a new price request to the OO
/// @param ancillaryData    - Ancillary data of the request
function priceDisputed(bytes32, uint256, bytes memory ancillaryData, uint256) external onlyOptimisticOracle {
    bytes32 questionID = keccak256(ancillaryData);
    QuestionData storage questionData = questions[questionID];

    // If a Question is already resolved, e.g by resolveManually, the priceDisputed callback should not update
    // any storage parameters.
    // Refund the reward to the question creator
    if (questionData.resolved) {
        TransferHelper._transfer(questionData.rewardToken, questionData.creator, questionData.reward);
        return;
    }

    if (questionData.reset) {
        questionData.refund = true;
        return;
    }

    // If the question has not been reset previously, reset the question
    // Ensures that there are at most 2 OO Requests at a time for a question
    _reset(address(this), questionID, false, questionData);
}
```

`_reset` creates a brand new price request and sets the `reset` flag to true, so the next dispute will see `refund` also being set to true. When the `refund` flag is on, the reward refunded by the Optimistic Oracle is held by the adapter contract and a natural/manual market resolution refunds the reward to the question's original creator.

The new price request has the same **question ID**.

```solidity
uint256 requestTimestamp = block.timestamp;
// Update the question parameters in storage
questionData.requestTimestamp = requestTimestamp;
questionData.reset = true;
if (resetRefund) questionData.refund = false;

// Send out a new price request with the new timestamp
_requestPrice(
    requestor,
    requestTimestamp,
    questionData.ancillaryData,
    questionData.rewardToken,
    questionData.reward,
    questionData.proposalBond,
    questionData.liveness
);
```

If you go back to the reset/refund logic above, `priceDisputed` returns early if `reset` is already set to true. This means the adapter **is not going to make a price request for the third time**. In that case contract admin has to step in to resolve the question (The function used to be called `emergencyResolve`, it seems to have been renamed to `resolveManually`).

#### Resolution

To resolve a question, anyone can call the function [resolve](https://github.com/Polymarket/uma-ctf-adapter/blob/main/src/UmaCtfAdapter.sol#L129). Remember the adapter acts as a bridge between UMA's Optimistic Oracle and the CTF. It retrieves the result from UMA if it's available and constructs a CTF compatible payout array to settle score.

To check if the question is ready to be resolved, the adapter calls UMA Optimistic Oracle's `hasPrice`.

```solidity
function _hasPrice(QuestionData storage questionData) internal view returns (bool) {
    return optimisticOracle.hasPrice(
        address(this), YES_OR_NO_IDENTIFIER, questionData.requestTimestamp, questionData.ancillaryData
    );
}
```

Price is available if the price request's state is `Settled`, `Resolved` or `Expired`.

1. It is settled if the price request has been officially settled by calling `settle` or `settleAndGetPrice`.
2. It is resolved if there is a dispute and the dispute result is determined.
3. It is expired if there is a price proposal and the it is past the expiration time with no dispute, before any settlement is done.

```solidity
if (request.settled) return State.Settled;

if (request.disputer == address(0))
    return request.expirationTime <= getCurrentTime() ? State.Expired : State.Proposed;

return
    _getOracle().hasPrice(
        identifier,
        _getTimestampForDvmRequest(request, timestamp),
        _stampAncillaryData(ancillaryData, requester)
    )
        ? State.Resolved
        : State.Disputed;
```

If price is available, the adapter calls `settleAndGetPrice` to settle score and also determines who receives the payout.

1. If the price request expires uncontested, the oracle sends the proposer back the original bond, final fee and also the reward.
2. If the price request dispute is resolved, the winner (original proposer or the disputer) gets back a) their bond, b) the unburned portion of the loser's bond, c) their final fee and d) the reward.

```solidity
if (state == State.Expired) {
    // In the expiry case, just pay back the proposer's bond and final fee along with the reward.
    request.resolvedPrice = request.proposedPrice;
    payout = request.requestSettings.bond.add(request.finalFee).add(request.reward);
    request.currency.safeTransfer(request.proposer, payout);
} else if (state == State.Resolved) {
    // In the Resolved case, pay either the disputer or the proposer the entire payout (+ bond and reward).
    request.resolvedPrice = _getOracle().getPrice(
        identifier,
        _getTimestampForDvmRequest(request, timestamp),
        _stampAncillaryData(ancillaryData, requester)
    );
    bool disputeSuccess = request.resolvedPrice != request.proposedPrice;
    uint256 bond = request.requestSettings.bond;

    // Unburned portion of the loser's bond = 1 - burned bond.
    uint256 unburnedBond = bond.sub(_computeBurnedBond(request));

    // Winner gets:
    // - Their bond back.
    // - The unburned portion of the loser's bond.
    // - Their final fee back.
    // - The request reward (if not already refunded -- if refunded, it will be set to 0).
    payout = bond.add(unburnedBond).add(request.finalFee).add(request.reward);
    request.currency.safeTransfer(disputeSuccess ? request.disputer : request.proposer, payout);
} else revert("_settle: not settleable");
```

It also sets the price request's `resolvedPrice`, which is used by the adapter to construct a payout array. The payout array is always of **length 2** and the only valid values are \[0,1], \[1,0] and \[1,1]. They corresponding to No winning, Yes winning and a draw.

There is a code comment on \[1,1] being invalid for neg risk markets. We can talk about it more in the chapter on neg risk, but the general idea is neg risk markets have multiple connected questions and one of them must resolve to YES and the rest must resolve to NO.

```solidity
/// @notice Construct the payout array given the price
/// @param price - The price retrieved from the OO
function _constructPayouts(int256 price) internal pure returns (uint256[] memory) {
    // Payouts: [YES, NO]
    uint256[] memory payouts = new uint256[](2);
    // Valid prices are 0, 0.5 and 1
    if (price != 0 && price != 0.5 ether && price != 1 ether) revert InvalidOOPrice();

    if (price == 0) {
        // NO: Report [Yes, No] as [0, 1]
        payouts[0] = 0;
        payouts[1] = 1;
    } else if (price == 0.5 ether) {
        // UNKNOWN: Report [Yes, No] as [1, 1], 50/50
        // Note that a tie is not a valid outcome when used with the `NegRiskOperator`
        payouts[0] = 1;
        payouts[1] = 1;
    } else {
        // YES: Report [Yes, No] as [1, 0]
        payouts[0] = 1;
        payouts[1] = 0;
    }
    return payouts;
}
```

This payout array is sent to the CTF contract for score settling. See the page **Conditional Tokens Framework** on how `reportPayouts` and `redeemPositions` work.

```solidity
ctf.reportPayouts(questionID, payouts);
```

[^1]: 
