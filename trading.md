# Trading

Polymarket's exchange contract is permissioned. You can't just trade any CTFs on the exchange. The token ID has to be whitelisted. Order matching is also permissioned, an exchange operator has to match and execute trades for traders.

Traders submit orders off-chain by signing the Order struct.

#### Terminologies

**Complement**: Two token IDs from an ERC-1155 contract are said to be complement of each other if they represent two sides of a condition.

**Maker Order**: A maker order is typically the order that first lands on the orderbook with a limit price.

**Taker Order**: A taker order is typically the order that lands on the orderbook after the maker order in a trade, but it does not necessarily mean it's a market order.

**Maker Amount**: If it's a buy, then it's the USDC amount. If it's a sell, then it's CTF amount.

**Taker Amount**: If it's a buy, then it's CTF amount. If it's a sell, then it's USDC amount.

**Mint Trade**: One side of the trade bids YES and the opposite side of the trade bids a complementary NO.

**Merge Trade**: One side of the trade sells YES and the opposite side of the trade sells a complementary NO.

**Complementary Trade**: One side of the trade wants to sell a YES or NO token to the opposite side of the trade.

#### Token Registration

Polymarket's admin whitelists a token by calling the function `registerToken`. `token0` and `token1` have to be the complement of each other (the YES and NO of the same question) but there is nothing stopping the admin from using the wrong token IDs.

Once the token ID is listed on the `registry`, both tokens become tradable on the CTF exchange.

```solidity
function _registerToken(uint256 token0, uint256 token1, bytes32 conditionId) internal {
    if (token0 == token1 || (token0 == 0 || token1 == 0)) revert InvalidTokenId();
    if (registry[ token0].complement != 0 || registry[ token1].complement != 0) revert AlreadyRegistered();

    registry[ token0] = OutcomeToken({complement: token1, conditionId: conditionId});

    registry[ token1] = OutcomeToken({complement: token0, conditionId: conditionId});

    emit TokenRegistered(token0, token1, conditionId);
    emit TokenRegistered(token1, token0, conditionId);
}
```

#### The Order Struct

```solidity
struct Order {
    /// @notice Unique salt to ensure entropy
    uint256 salt;
    /// @notice Maker of the order, i.e the source of funds for the order
    address maker;
    /// @notice Signer of the order
    address signer;
    /// @notice Address of the order taker. The zero address is used to indicate a public order
    address taker;
    /// @notice Token Id of the CTF ERC1155 asset to be bought or sold
    /// If BUY, this is the tokenId of the asset to be bought, i.e the makerAssetId
    /// If SELL, this is the tokenId of the asset to be sold, i.e the takerAssetId
    uint256 tokenId;
    /// @notice Maker amount, i.e the maximum amount of tokens to be sold
    uint256 makerAmount;
    /// @notice Taker amount, i.e the minimum amount of tokens to be received
    uint256 takerAmount;
    /// @notice Timestamp after which the order is expired
    uint256 expiration;
    /// @notice Nonce used for onchain cancellations
    uint256 nonce;
    /// @notice Fee rate, in basis points, charged to the order maker, charged on proceeds
    uint256 feeRateBps;
    /// @notice The side of the order: BUY or SELL
    Side side;
    /// @notice Signature type used by the Order: EOA, POLY_PROXY or POLY_GNOSIS_SAFE
    SignatureType signatureType;
    /// @notice The order signature
    bytes signature;
}

enum SignatureType {
    // 0: ECDSA EIP712 signatures signed by EOAs
    EOA,
    // 1: EIP712 signatures signed by EOAs that own Polymarket Proxy wallets
    POLY_PROXY,
    // 2: EIP712 signatures signed by EOAs that own Polymarket Gnosis safes
    POLY_GNOSIS_SAFE,
    // 3: EIP1271 signatures signed by smart contracts. To be used by smart contract wallets or vaults
    POLY_1271
}

enum Side {
    // 0: buy
    BUY,
    // 1: sell
    SELL
}

enum MatchType {
    // 0: buy vs sell
    COMPLEMENTARY,
    // 1: both buys
    MINT,
    // 2: both sells
    MERGE
}
```

The gist is the `tokenId` is always going to be the CTF to be bought or sold depending on the order's side. If it's a buy order, the maker amount is the USDC amount the buyer is willing to pay and the taker amount is the CTF amount the buyer wants. If it's a sell order, the maker amount is the CTF amount the seller is willing to part with and the taker amount is the USDC amount the seller wants. There are match types and we will talk about them in upcoming pages.

#### Trading Logic

**Validations**

`matchOrders` is pretty much the only function used for trading. There is another function `fillOrders` but I don't ever see it being used.&#x20;

The operator always matches a taker order against a list of maker orders and the operator has to specify the fill amount for each order. The fill amount is always in terms of the order's maker amount. If the taker order is a buy order, then `takerFillAmount` is the amount of USDC to fill. If a maker order is a sell order, then `makerFillAmounts[i]` is the amount of CTF to fill. It makes sense because you need to match USDC against CTF to create a trade (at least for complementary trades, there are other trade types and we will explore them in a bit).

```solidity
/// @notice Matches a taker order against a list of maker orders
/// @param takerOrder       - The active order to be matched
/// @param makerOrders      - The array of maker orders to be matched against the active order
/// @param takerFillAmount  - The amount to fill on the taker order, always in terms of the maker amount
/// @param makerFillAmounts - The array of amounts to fill on the maker orders, always in terms of the maker amount
function matchOrders(
    Order memory takerOrder,
    Order[] memory makerOrders,
    uint256 takerFillAmount,
    uint256[] memory makerFillAmounts
) external nonReentrant onlyOperator notPaused {
    _matchOrders(takerOrder, makerOrders, takerFillAmount, makerFillAmounts);
}
```

`matchOrders` perform the following checks:

1. Certain orders have a specified `taker` for private trades and it checks `msg.sender` is the `taker` (I don't think it's ever used)
2. Order isn't expired
3. Order signature is valid. Valid signers can be an EOA, Polymarket's proxy wallet or Polymarket's Gnosis safe
4. Order's feeRateBps is not above the contract's max fee rate bps which is 10% (Polymarket doesn't charge a fee at the moment so it's also not going to hit)
5. The token ID is registered
6. The order is not fully filled or cancelled. Each fill increments the order's fill amount until it is fully filled. Orders can be cancelled on-chain but I also have not seen it being available as a feature from the UI (Pro-traders might use it). It isn't as important as a permissionless exchange to have on-chain cancellation if the traders trust the operator to not execute orders cancelled off-chain.
7. The order's nonce is valid. Each order comes with a nonce and it has to be equal to the order maker's current nonce. Multiple orders can share the same nonce. This is also likely something to be used by pro-traders.

**Remaining Liquidity Check**

During order validation, the exchange verifies the order's maker amount is at least as much as the order's remaining amount. It reverts if there is insufficient liquidity and sets the order to be filled if after the match there is 0 remaining liquidity. The order's remaining liquidity is also decremented.

```solidity
function _updateOrderStatus(bytes32 orderHash, Order memory order, uint256 makingAmount)
    internal
    returns (uint256 remaining)
{
    OrderStatus storage status = orderStatus[orderHash];
    // Fetch remaining amount from storage
    remaining = status.remaining;

    // Update remaining if the order is new/has not been filled
    remaining = remaining == 0 ? order.makerAmount : remaining;

    // Throw if the makingAmount(amount to be filled) is greater than the amount available
    if (makingAmount > remaining) revert MakingGtRemaining();

    // Update remaining using the makingAmount
    remaining = remaining - makingAmount;

    // If order is completely filled, update isFilledOrCancelled in storage
    if (remaining == 0) status.isFilledOrCancelled = true;

    // Update remaining in storage
    status.remaining = remaining;
}
```

**Taker Order's Taker Amount**

The actual taker order's taking amount is calculated with the formula below. If `takerFillAmount` is 100% of the taker order's maker amount then it cancels out with the denominator and we end up with the full taker order's taker amount. If not it gives us a portion of the taker amount.

$$
\text{takingAmount}
= \text{takerFillAmount}
\times \frac{\text{takerOrderTakerAmount}}{\text{takerOrderMakerAmount}}
$$

The maker and taker asset IDs are defined in terms of the taker order. If the taker order is a buy order, then the maker asset is USDC and the taker asset is CTF and vice versa. Polymarket uses 0 to represent USDC.

```solidity
function _deriveAssetIds(Order memory order) internal pure returns (uint256 makerAssetId, uint256 takerAssetId) {
    if (order.side == Side.BUY) return (0, order.tokenId);
    return (order.tokenId, 0);
}
```

**Fill Maker Orders**

The exchange transfers the taker's maker asset to the exchange. It is all very confusing with the terminologies, but remember a maker asset is the asset the order maker **has**, and a taker asset is the asset the order maker **wants**.

```
uint256 making = takerFillAmount;
_transfer(takerOrder.maker, address(this), makerAssetId, making);
```

Then it calls an internal function `_fillMakerOrders` with the arguments `_takerOrder`, `makerOrders` and `makerFillAmounts`. This function fills maker orders with the exchange acting as the counterparty.

There are **3 types of orders**. A taker order does not necessarily have to be of opposite type of a maker order. They can both be `buy` or they can both be `sell` for the reasons below.

1. If the trade is a **complementary** trade, then the taker order and the maker order must be of opposite types and the token ID must be the same. ie. The maker buys token 1 from the taker or sells token 1 to the taker.
2. If the trade is **mint** trade, then the maker and the taker order must both be a `buy`. Their token IDs must be complement of each other. ie. The maker bids YES for 0.6c and the taker bids NO for 0.4c, the combined price is equal to $1 and it can deposit the $1 into the CTF contract via `splitPosition` to mint 100 YES shares for the maker and 100 NO shares for the taker.
3. If the trade is **merge** trade, then the maker and the taker order must both be a `sell`. Their token IDs must be complement of each other. ie. The maker sells 1 YES for 0.6c and the taker sells 1 NO for 0.4c, the combined price is equal to $1 and it can merge 1 YES and 1 NO via the CTF's `mergePositions` to get back $1, with 0.6c going to the maker and 0.4c going to the taker.

```solidity
function _deriveMatchType(Order memory takerOrder, Order memory makerOrder) internal pure returns (MatchType) {
    if (takerOrder.side == Side.BUY && makerOrder.side == Side.BUY) return MatchType.MINT;
    if (takerOrder.side == Side.SELL && makerOrder.side == Side.SELL) return MatchType.MERGE;
    return MatchType.COMPLEMENTARY;
}
```

`_validateTakerAndMaker` checks if the token IDs are equal or are complementary in the case of mint or merge.

```solidity
/// @notice Ensures the taker and maker orders can be matched against each other
/// @param takerOrder   - The taker order
/// @param makerOrder   - The maker order
function _validateTakerAndMaker(Order memory takerOrder, Order memory makerOrder, MatchType matchType)
    internal
    view
{
    if (!CalculatorHelper.isCrossing(takerOrder, makerOrder)) revert NotCrossing();

    // Ensure orders match
    if (matchType == MatchType.COMPLEMENTARY) {
        if (takerOrder.tokenId != makerOrder.tokenId) revert MismatchedTokenIds();
    } else {
        // both bids or both asks
        validateComplement(takerOrder.tokenId, makerOrder.tokenId);
    }
}
```

Another validation is on the orders' prices. Orders cannot be matched against each other if their prices do not cross. This means

1. For a **mint** order, the combined price has to be ≥ 1. They need to provide at least $1 in total to receive 1 YES and 1 NO.
2. For a **merge** order, the combined price has to be <= 1. The CTF contract cannot give more than $1 in total to the order maker and taker.
3. For a **complementary** trade, the bid price must be ≥ the ask price. The seller must get at least as much as what he asked for.

```solidity
function _isCrossing(uint256 priceA, uint256 priceB, Side sideA, Side sideB) internal pure returns (bool) {
    if (sideA == Side.BUY) {
        if (sideB == Side.BUY) {
            // if a and b are bids
            return priceA + priceB >= ONE;
        }
        // if a is bid and b is ask
        return priceA >= priceB;
    }
    if (sideB == Side.BUY) {
        // if a is ask and b is bid
        return priceB >= priceA;
    }
    // if a and b are asks
    return priceA + priceB <= ONE;
}
```

After validations, similar taking amount calculation is performed on each maker order. The `fillAmount` is the maker order's making amount.

```solidity
uint256 making = fillAmount;
(uint256 taking, bytes32 orderHash) = _performOrderChecks(makerOrder, making);
```

$$
\text{takingAmount}
= \text{makerFillAmounts[i]}
\times \frac{\text{makerOrders[i].takerAmount}}{\text{makerOrders[i].makerAmount}}
$$

There is a fee on the trade if the order's `feeRateBps` is nonzero. As far as I know Polymarket does not charge a fee and we will go further into details in the **Fees** page.

With all the information available, the exchange now fills the maker order against the exchange.

1. It transfers the maker's maker asset to the exchange.
2. `_executeMatchCall` does nothing if it's a **complementary** trade, splits position if it's a **mint** trade and merge positions if it's a **merge** trade.
3. After executing the match call, the contract must be able to fill the maker order's amount of want tokens. If it is a **complementary** trade, `_executeMatchCall` does nothing but it should have already pulled enough tokens from the taker order to fill the maker order. If it's a **mint** trade, the position split should have minted sufficient CTFs to be sent to the maker. If it's a **merge** trade, the positions merge should have redeemed sufficient USDC to be sent to the maker.
4. Finally, it sends the maker's "want" tokens to the maker after a fee (if any).

```solidity
/// @notice Fills a maker order using the Exchange as the counterparty
/// @param makingAmount - Amount to be filled in terms of maker amount
/// @param takingAmount - Amount to be filled in terms of taker amount
/// @param maker        - The order maker
/// @param makerAssetId - The Token Id of the Asset to be sold
/// @param takerAssetId - The Token Id of the Asset to be received
/// @param matchType    - The match type
/// @param fee          - The fee charged to the Order maker
function _fillFacingExchange(
    uint256 makingAmount,
    uint256 takingAmount,
    address maker,
    uint256 makerAssetId,
    uint256 takerAssetId,
    MatchType matchType,
    uint256 fee
) internal {
    // Transfer makingAmount tokens from order maker to Exchange
    _transfer(maker, address(this), makerAssetId, makingAmount);

    // Executes a match call based on match type
    _executeMatchCall(makingAmount, takingAmount, makerAssetId, takerAssetId, matchType);

    // Ensure match action generated enough tokens to fill the order
    if (_getBalance(takerAssetId) < takingAmount) revert TooLittleTokensReceived();

    // Transfer order proceeds minus fees from the Exchange to the order maker
    _transfer(address(this), maker, takerAssetId, takingAmount - fee);

    // Transfer fees from Exchange to the Operator
    _chargeFee(address(this), msg.sender, takerAssetId, fee);
}
```

**Execute Taker Order Transfer**

After fulfilling all maker orders, the exchange still needs to fulfil the taker order as the taker hasn't received anything yet!

First, it checks if there is enough tokens to be sent to the order taker. If there is a surplus (more tokens than what the taker asked for), the exchange gives the taker the surplus. It does not capture the surplus.

```solidity
function _updateTakingWithSurplus(uint256 minimumAmount, uint256 tokenId) internal returns (uint256) {
    uint256 actualAmount = _getBalance(tokenId);
    if (actualAmount < minimumAmount) revert TooLittleTokensReceived();
    return actualAmount;
}
```

Then the exchange calculates the taker order's fee (if any), and sends tokens to the order taker.

```solidity
uint256 fee = CalculatorHelper.calculateFee(
    takerOrder.feeRateBps, takerOrder.side == Side.BUY ? taking : making, making, taking, takerOrder.side
);

// Execute transfers

// Transfer order proceeds post fees from the Exchange to the taker order maker
_transfer(address(this), takerOrder.maker, takerAssetId, taking - fee);

// Charge the fee to taker order maker, explicitly transferring the fee from the Exchange to the Operator
_chargeFee(address(this), msg.sender, takerAssetId, fee);
```

After sending the order taker the "want" tokens, the exchange checks if there it holds any remaining maker asset provided by the taker. If so, it refunds the taker.

The exchange should never hold any USDC or CTFs after each trade.

```solidity
// Refund any leftover tokens pulled from the taker to the taker order
uint256 refund = _getBalance(makerAssetId);
if (refund > 0) _transfer(address(this), takerOrder.maker, makerAssetId, refund);
```
